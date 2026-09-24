---
title: "tesseract.js をブラウザで自前ホストして、画像 PDF を外部に送らずに OCR する"
emoji: "🔍"
type: "tech"
topics: ["JavaScript", "OCR", "tesseract", "pdfjs", "Cloudflare"]
published: true
---

請求書や領収書の PDF をブラウザの中だけで処理するサービスを作っています。文字が埋め込まれた PDF は pdf.js で読めますが、スマホで撮った写真やスキャンの PDF には文字のデータがありません。そこで tesseract.js をブラウザで動かし、worker・wasm・言語データをすべて自分のサーバーに置いて、画像を外部に送らずに OCR する形にしました。

先に結果を書きます。

- 写真 8 件 × 5 項目（日付・金額・取引先・書類種別・書類番号）で、正解は 15/40 から 28/40 まで上がりました。上がった分は OCR 自体ではなく、OCR の後の文字列の扱いと、横向きの写真の向きの判定です。
- 画像の前処理（二値化・照明補正・傾き補正・拡大）は 6 通り試して、どれも「前処理なし」に勝てませんでした。
- 手書きの数字は読めません。これは対象外にしました。
- 処理時間は 8 件で約 70 秒。1 件あたり数秒から 30 秒です。

外部の OCR API を使えば精度は上がりますが、書類には取引先名や金額が入っているので、サービスの前提として画像を外に出したくありませんでした。この記事は、その制約の中で tesseract.js をどう組み込み、どこでつまずいたかの記録です。

## 構成

処理の流れは次のとおりです。

1. pdf.js で PDF の 1〜2 ページ目を canvas に描く（幅 2000px、白背景）。
2. tesseract.js の worker に canvas を渡して文字列を得る。
3. 読めた文字が少なければ、canvas を 90 度・270 度に回してもう一度 OCR する。
4. 文字列を行に分け、日本語の 1 文字ずつの空白を詰めてから、既存の推定処理（日付や金額のラベルを探す純関数）に渡す。

tesseract.js は 5.1 系、pdf.js は `pdfjs-dist` です。配信は Cloudflare Workers の静的アセットで、フレームワークは使っていません。

## 1. pdf.js で白背景の canvas に描く

写真の PDF は、中身が 1 枚の JPEG です。pdf.js でページを描いてから tesseract に渡します。

```ts
export async function renderPageBitmap(data: ArrayBuffer, pageIndex = 0, pixelWidth = 1600): Promise<HTMLCanvasElement> {
  const doc = await pdfjs.getDocument(docParams(data)).promise;
  try {
    const page = await doc.getPage(pageIndex + 1);
    const base = page.getViewport({ scale: 1 });
    const scale = Math.min(pixelWidth / base.width, 4); // 大きすぎる拡大は避ける
    const viewport = page.getViewport({ scale });
    const canvas = document.createElement("canvas");
    canvas.width = Math.floor(viewport.width);
    canvas.height = Math.floor(viewport.height);
    // 白で塗ってから描く。塗らないと透明部分が OCR で黒背景になり、文字が読めない。
    const ctx = canvas.getContext("2d");
    if (ctx) {
      ctx.fillStyle = "#ffffff";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
    }
    await page.render({ canvas, viewport, background: "#ffffff" }).promise;
    page.cleanup();
    return canvas;
  } finally {
    await doc.destroy();
  }
}
```

最初に踏んだのが、白で塗らずに描いた canvas を渡すと何も読めないという問題です。canvas の初期値は透明で、tesseract に渡すときにアルファが落ちて黒になります。黒地に黒文字なので、結果はほぼ空でした。`page.render` の `background` だけでは足りない場合があったので、塗ってから描いています。

幅は 2000px にしました。スマホの写真は元が 3000〜4000px あるので縮小になります。あとで書きますが、1.5 倍に拡大しても精度は上がりませんでした。

## 2. tesseract.js のファイルを自分のサーバーに置く

tesseract.js は既定で、worker と wasm を jsDelivr から、言語データを GitHub から読みに行きます。画像そのものは外部に送られませんが、外部への通信が発生することと、CDN の都合で動かなくなる可能性を避けたかったので、全部を `/tesseract/` 配下に置きました。

置いたファイルは次の 4 種類です。

| 置き場所 | 中身 | 出どころ |
|---|---|---|
| `/tesseract/worker.min.js` | Web Worker 本体 | `node_modules/tesseract.js/dist/worker.min.js` |
| `/tesseract/core/tesseract-core-lstm.wasm(.js)` | SIMD 非対応のブラウザ用 | `node_modules/tesseract.js-core/` |
| `/tesseract/core/tesseract-core-simd-lstm.wasm(.js)` | SIMD 対応のブラウザ用 | 同上 |
| `/tesseract/tessdata/jpn.traineddata`, `eng.traineddata` | 言語データ（fast 版） | tessdata_fast リポジトリ |

`corePath` にはディレクトリを指定します。tesseract.js がブラウザの SIMD 対応を見て、`lstm` か `simd-lstm` のどちらかを自分で選びます。LSTM だけの版（legacy エンジンを含まない版）にすると、wasm が 2.8MB で済みます。

言語データは `.traineddata.gz` ではなく展開したものを置き、`gzip: false` を指定します。Cloudflare 側の圧縮に任せる方が扱いが簡単でした。jpn が 2.4MB、eng が 4.1MB です。

```ts
import { createWorker, OEM, type Worker } from "tesseract.js";

let workerPromise: Promise<Worker> | null = null;

function getWorker(): Promise<Worker> {
  if (!workerPromise) {
    workerPromise = createWorker(["jpn", "eng"], OEM.LSTM_ONLY, {
      workerPath: "/tesseract/worker.min.js",
      corePath: "/tesseract/core",
      langPath: "/tesseract/tessdata",
      gzip: false,
      logger: (m: { status: string; progress: number }) => {
        if (m.status === "recognizing text" && onProgress) onProgress(m.progress);
      },
    }).catch((e) => {
      workerPromise = null; // 失敗したら次回やり直せるようにする
      throw e;
    });
  }
  return workerPromise;
}
```

worker は 1 つを使い回します。言語データの読み込みに数秒かかるので、ファイルごとに作り直すと待ち時間が積み上がります。`createWorker` の Promise を保持しておき、失敗したときだけ捨てて次回作り直すようにしました。進捗のコールバックは worker 単位でしか登録できないので、モジュールの変数を呼び出しごとに差し替えています。

`worker.recognize` には canvas をそのまま渡せます。出力は `text` だけ取れば十分で、`hocr` と `tsv` を切ると少し速くなります。

```ts
async function recognizeText(worker: Worker, canvas: HTMLCanvasElement): Promise<string> {
  const { data } = await worker.recognize(canvas, {}, { blocks: true, text: true, hocr: false, tsv: false });
  return data.text ?? "";
}
```

## 3. 横向きの写真は回して読み直す

8 件の写真のうち 1 件が、机の上で横向きに撮られたものでした。tesseract は向きを直してくれないので、この 1 件は 5 項目とも読めませんでした。

向きの判定には OSD 用の言語データを追加する方法もありますが、ファイルが増えるのと、写真では判定が安定しなかったので、別の方法にしました。読めた結果から「読めている度合い」を点数にして、低ければ 90 度と 270 度に回して読み直し、いちばん点数が高いものを使います。

```ts
/**
 * OCR の結果がどれだけ「読めているか」の目安。漢字の数と、4 桁以上の数字列（日付・金額・番号）の数から出す。
 * 横向きの写真を OCR すると、漢字がほとんど出ず数字列も切れ切れになる。
 */
export function ocrReadabilityScore(text: string): number {
  const kanji = (text.match(/[㐀-鿿]/g) ?? []).length;
  const digitRuns = (text.match(/[0-9]{4,}/g) ?? []).length;
  return kanji + digitRuns * 5;
}

export const OCR_READABLE_SCORE = 40;
```

しきい値の 40 は、手元の写真で正しい向きなら漢字が 61 以上・数字列が 2 以上、誤った向きなら漢字が 23 以下・数字列が 0 だったことから決めました。英語だけの書類は漢字が 0 になりますが、数字列で点が付きます。180 度は試していません。上下逆に撮る人はほとんどいないので、時間を優先しました。

```ts
export async function ocrCanvasToLines(canvas: HTMLCanvasElement): Promise<Line[]> {
  const worker = await getWorker();
  let best = await recognizeText(worker, canvas);
  let bestScore = ocrReadabilityScore(best);
  if (bestScore < OCR_READABLE_SCORE) {
    for (const deg of [90, 270] as const) {
      const text = await recognizeText(worker, rotateCanvas(canvas, deg));
      const score = ocrReadabilityScore(text);
      if (score > bestScore) {
        best = text;
        bestScore = score;
      }
    }
  }
  return linesFromOcrText(best);
}
```

回転は canvas を作り直して `ctx.rotate` で描くだけです。回した後の canvas も白で塗ってから描きます。これで横向きの 1 件は 5 項目中 4 項目が埋まり、全体は 24/40 から 28/40 になりました。処理時間はこの 1 件だけ 3 倍になり、他の 7 件は 1 回目で基準を超えるので回しません。

## 4. OCR の文字列を推定処理に渡す前にやること

tesseract の日本語の出力は、「請 求 書」「合 計」のように 1 文字ずつ空白が入ることが多いです。ラベルの照合は「合計」という並びを探すので、このままでは当たりません。一方で、列の区切りとして入っている広い空白は残したいので、「日本語の文字どうしの間にある 1 個の空白」だけを詰めます。

```ts
const CJK = "\\u3040-\\u30ff\\u3400-\\u9fff\\uf900-\\ufaff\\uff00-\\uffef";
const CJK_SPACE = new RegExp(`([${CJK}]) (?=[${CJK}])`, "g");

export function collapseCjkSpaces(text: string): string {
  return text
    .split("\n")
    .map((line) => {
      let prev = "";
      let s = line;
      while (s !== prev) {
        prev = s;
        s = s.replace(CJK_SPACE, "$1");
      }
      return s;
    })
    .join("\n");
}
```

行の分割は tesseract の `text` の改行をそのまま使っています。座標から行を組み直す方法も試しましたが、tesseract 自身の行分けの方が正確でした。

この後は、文字が埋め込まれた PDF と同じ推定処理（ラベルの右の日付を取る、合計のラベルに近い金額を取る、など）に流します。ただし OCR 由来の行には誤読の癖があるので、`ocr: true` のときだけ効く緩い規則を足しました。

- 「2026年6月1 A」のように「日」が「A」に化ける、「7月 1 7 日」のように数字の間に空白が入る。日付として読む前に「…日」の形に整える。
- 「¥」が「\」や「#」に化ける。「\」は円記号として扱う（JIS の円記号でもあるので、文字が埋め込まれた PDF にも効く）。「#」は番号のラベルとしては使わない。
- 「8 円」のような 1 桁の誤読。OCR のときは 10 円未満を捨てる。
- 社名の前後に郵便番号や 1〜2 文字の英字のノイズが付く。先頭・末尾のそういう塊を落とす。
- 「株式会社」だけが 1 行として出る。名前の本体が無い行は社名にしない。

この種の規則は、文字が埋め込まれた PDF の精度を落とさないことが条件です。既存のテストと、実物の PDF 21 件での結果が変わらないことを確かめてから入れました。

## 5. 精度をどう測ったか

ブラウザを起動して測るのは遅いので、同じ tesseract.js・同じ言語データを Node から動かす道具を作りました。写真を幅 2000px の PNG にし、Node で OCR して文字列をファイルに保存し、推定処理をテストランナーから通して正解表と突き合わせます。

```js
import { createRequire } from "node:module";
const require = createRequire(resolve("app/package.json"));
const { createWorker, OEM } = require("tesseract.js");

const worker = await createWorker(["jpn", "eng"], OEM.LSTM_ONLY, {
  corePath: resolve("app/public/tesseract/core"),
  langPath: resolve("app/public/tesseract/tessdata"),
  cachePath: resolve("logs/tesseract-cache"),
  gzip: false,
});
const { data } = await worker.recognize(pngPath, {}, { text: true });
```

Node 版では `corePath` と `langPath` にローカルのディレクトリを渡せます。ブラウザの結果と完全には一致しません。pdf.js が描いた画像と、macOS の `sips` が描いた画像で読みが少し変わり、同じ規則で通しても 3 項目くらいぶれます。傾向を見るには十分でした。

結果は次のとおりです。

| 段階 | 正解 |
|---|---|
| OCR を通して既存の推定処理にそのまま渡す | 15/40 |
| OCR 由来の行に緩い規則を足す | 24/40 |
| 横向きの写真を回して読み直す | 28/40 |

残りの 12 項目は、手書きの日付・金額・番号が 6、印字の誤読が 4、ラベルの「発行No.」が読めなかったものが 2 です。手書きは tesseract の印字用モデルでは読めないので、対象外として案内に書きました。

### 前処理は効かなかった

OCR の精度を上げる定番として、二値化・照明のむらの補正・傾きの補正・拡大があります。sharp で 6 通り試しましたが、どれも前処理なしに勝てませんでした。

| 前処理 | 正解 |
|---|---|
| なし（色のまま渡す） | 28/40 |
| 1.5 倍に拡大 | 27/40 |
| 解像度の指定（user_defined_dpi） | 27/40 |
| 傾きの補正だけ | 25/40 |
| グレースケール化 + 照明のむらの補正 | 24/40 |
| 色を保った照明のむらの補正 | 20/40 |
| Sauvola 法の局所二値化 | 15〜17/40 |

tesseract 4 以降の LSTM は内部で二値化するので、外で二値化すると情報を削るだけになるようです。写真の影や紙のしわに対しても、そのまま渡す方が良い結果でした。

ここで 1 つ落とし穴がありました。sharp は既定で画像に埋め込まれた ICC プロファイル（iPhone の写真の Display P3 など）を適用して画素の値を変えます。これだけで 28/40 が 24/40 に落ちました。前処理の効果を比べるときは `sharp(path, { ignoreIcc: true })` にして、「素通し」が元画像と同じ結果になることを先に確かめる必要があります。sharp が書く PNG の解像度情報（pHYs）とアルファチャンネルも tesseract の読みを変えたので、外して RGB で渡しています。

## 6. その他のハマりどころ

- ベーシック認証をかけた環境で、ヘッドレス Chrome に `Network.setExtraHTTPHeaders` で認証ヘッダーを付けて動作確認をすると、OCR が永遠に終わりません。CDP のヘッダーはページの通信にしか付かず、Web Worker が読む wasm と言語データが 401 になるためです。エラーも出ないので気づくのに時間がかかりました。認証なしのローカル配信で確認するようにしました。
- 言語データを `jpn` だけにすると、英数字の読みが落ちます。`["jpn", "eng"]` の 2 つを渡すと数字と英字が安定しました。読み込みは 6.5MB 分増えます。
- 1 ページ目だけでは足りない書類があるので、2 ページ目まで読みます。3 ページ目以降は時間に見合いませんでした。
- 使い終わった worker は `terminate` してメモリを空けます。次に呼ばれたときに作り直します。

## まとめ

tesseract.js を自前ホストすれば、画像を外部に送らずにブラウザだけで OCR できます。ファイルは 4 種類、合計で約 15MB です。精度は印字の書類なら実用の範囲で、手書きは無理です。精度を上げる鍵は画像の前処理ではなく、OCR の後の文字列の扱いと向きの判定でした。

この処理は、電子帳簿保存法向けに請求書・領収書 PDF のファイル名を付け直す [電帳リネーム](https://dencho-rename.com) の有料機能として動いています。OCR で埋めた値には「要確認」の印を付けて、利用者に確認してもらう前提にしています。
