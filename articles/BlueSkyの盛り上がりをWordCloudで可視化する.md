:::note info
viviONグループでは、DLsiteやcomipoなど、二次元コンテンツを世の中に届けるためのサービスを運営しています。
ともに働く仲間を募集していますので、興味のある方は[こちら](#一緒に二次元業界を盛り上げていきませんか)まで。
:::

## はじめに

BlueSkyのAPIは、現在（2024年12月）開発者登録などは不要で、簡単に利用できます。かつてのTwitterのように、さまざまなデータを利用をできるため、自由なアプリケーション開発が可能です。
BlueSkyにはトレンドを表示する機能などはありませんが、データを取得して自分で加工ができます。本記事ではBlueSkyのAPIでフィードを取得し、WordCloudでBlueSkyの投稿を可視化します。

## 環境

```json
"next": "15.1.0"
"react": "^19.0.0"
"@atproto/api": "^0.13.20"
"kuromojin": "^3.0.0"
"typescript": "^5"
```

Next.jsを利用していますが、掲載するコードはフレームワークに依存しない記法で書いていますので、適宜読み替えてください。

## 実装

以下のステップで実装を進めます。

1. BlueSkyのAPIを利用して、フィードを取得する
2. 取得したフィードから、WordCloud用に前処理を行う
3. WordCloudを生成する

### BlueSkyのAPIを利用して、フィードを取得する

BlueSkyは、ハンドル/パスワードを用いた認証とOAuthを用いた認証を提供しています。
今回は簡易的に実装を進めたいので、ハンドル/パスワードを用いた認証を利用します。

:::note warn
ハンドル/パスワードを用いた認証は、late limitが100回/日に制限されています。
セッション情報をcookieなどに保存して使い回すことで、APIリクエストの回数を減らせます。公開サービスなどで利用する場合は情報を露出しないように注意してください。
:::

```bash
npm install @atproto/api
```

`Agent`を作成し、ハンドル/パスワードを用いてログインすることで、`Agent`にsession認証情報が保存されます。
これで、`Agent`を通した同一セッションのリクエストに認証情報が付与されます。


```ts
import { AtpAgent } from "@atproto/api";

const agent = new AtpAgent({
  service: 'https://bsky.social'
});

await this.agent.login({
  identifier: 'handle.bsky.social', // ハンドル
  password: 'password', // パスワード、App Passwordを利用することを推奨されています
});
```

同一セッションでフィードを取得します。

```ts
await this.agent.app.bsky.feed.getFeed({
  feed: 'at-uri',
});
```

`at-uri`は、`at://{did}/app.bsky.feed.generator/{feedId}`の形式で指定します。
[フィード — Bluesky](https://bsky.app/feeds)から適当なフィードを開くと、URLが`https://bsky.app/profile/{handle}/feed/{feedId}`の形式で表示されていることがわかります。

`handle`はユーザーのハンドル名ですが、`at-uri`に必要なのは`did`です。フィードを共有からURLを取得すると、`handle`部分が`did`の形式で取得できますが、ユーザーの使いやすさを考慮して、両対応にするべきでしょう。

フィードと同様に、ハンドルから`did`を取得するAPIも提供されています。

```ts
await this.agent.com.atproto.identity.resolveHandle({
  handle: 'handle.bsky.social',
});
```

これを利用して`at-uri`を作成し、フィードを取得します。

```ts
let did: string;

if (handle.startsWith("did:")) {
  did = handle;
} else {
  did = await getDid(handle); // ハンドルからdidを取得
}

const uri = `at://${did}/app.bsky.feed.generator/${feedId}`;

const feed = await this.agent.app.bsky.feed.getFeed({
  feed: uri,
  limit: 100,
}).then(res => res.data);
```

`getFeed`にはいくつかのオプションが指定できます。
- `limit` : 取得するフィードの数
  - デフォルトは50
  - `1`以上`100`以下の整数で指定する
- `cursor` : ページネーションに使用する
  - 未指定、または空文字の場合は最新の投稿を取得する
  - 続きから取得するためには、前回のレスポンスの`cursor`を指定する

返ってくるデータの形式はドキュメントを参照してください。TypeScriptの型定義がありますが、一部の型が不足している場合があり、APIのレスポンスを参照して型付けを行う必要があります。

https://docs.bsky.app/docs/api/app-bsky-feed-get-feed

:::note warn
精確な盛り上がりを知りたい場合は、投稿時間を検証して一定期間の投稿を取得すると良いです。
:::

### 取得したフィードから、WordCloud用に前処理を行う

取得したフィードから、WordCloud用に前処理を行います。
テキストの形態素解析には、`kuromojin`を利用します。
`kuromojin`は、人気の形態素解析ライブラリ`kuromoji.js`をラップしたライブラリです。`kuromojin`は、Promiseを返す関数を提供しており、非同期で形態素解析が行えます。

```bash
npm install kuromojin
```

BlueSkyのAPIから取得した投稿を`kuromojin`へ渡す前に、テキストを抽出します。

```ts
function hasProp<K extends PropertyKey>(data: object, prop: K): data is Record<K, unknown> {
  return prop in data;
}

const texts: string[] = [];

feed.forEach((post) => {
  let text = "";

  // PostView.recordの型が空オブジェクトになってしまうので、型定義を上書き
  if (hasProp(post.post.record, "text") && typeof post.post.record.text === "string") {
    text = post.post.record.text;
  }

  const stopWords = ["これ", "それ", "あれ", "どれ", "です", "ます", "いる", "ある", "ない"];
  const stopWordsPattern = new RegExp(`\\b(${stopWords.join("|")})\\b`, "g");

  text = text
    .replace(stopWordsPattern, "")
    .replace(/[!"#$%&'()*+,\-./:;<=>?@[\\\]^_`{|}~]/g, "")
    .replace(/\d+/g, "0")
    .replace(/[^\p{Script=Hiragana}\p{Script=Katakana}\p{Script=Han}ー]/gu, "")
    .replace(/\s+/g, " ")
    .trim();

  texts.push(text);
});
```

`PostView.record`の型が空オブジェクトになってしまうので、型定義を上書きしています。

`kuromoji`で使われている辞書データの関係で、意味のない単語で分かち書きされることがあります。このタイミングで改行や記号を削除し、意味のない単語がなるべく出現しないような処理をしています。ここはお好みで処理を変更してください。

次に、`kuromojin`を利用して形態素解析をします。

Next.jsで`kuromojin`を利用する場合、辞書へのパスの指定が必要でした。
`kuromojin`の`tokenize`関数を実行する際に、オプションとして`dicPath`を指定します。
ここでは、`node_modules/kuromoji/dict`を指定していますが、環境によっては異なる場合があります。
（Next.jsのローカル環境では、デフォルトのパスで動作しませんでした）

他にもURLを指定して辞書データを取得する方法もありますが、`kuromoji.js`側のURLパース処理に問題があるらしく、取得がうまくいきませんでした。
`kuromoji.js`は2024年12月現在、メンテナンスされていないプロジェクトなので、URLを指定して取得したい場合はforkして修正するか、ローカルに辞書データを持つ必要があります。

https://github.com/azu/kuromojin/issues/12



```ts
type Datum = {
  text: string;
  value: number;
};

type Token = {
  surface_form: string;
  pos: string;
};

const path = "node_modules/kuromojin/dict";
const wordCloud: Datum[] = [];

await Promise.all(
  texts.map(async (text) => {
    const tokens = await tokenize(text, { dicPath: path });
    tokens.forEach((token) => {
      // 一般名詞、固有名詞、サ変接続の単語のみを抽出
      const pos = ["名詞", "固有名詞", "サ変接続"];
      if (token.pos === "名詞" && pos.includes(token.pos_detail_1)) {
        const word = wordCloud.find((w) => w.text === token.surface_form);
        if (word) {
          word.value += 1;
        } else {
          wordCloud.push({ text: token.surface_form, value: 1 });
        }
      }
    });
  }),
);

// 出現回数が多い順にソート
return wordCloud.sort((a, b) => b.value - a.value);
```

`tokenize`関数はPromiseを返すので、`Promise.all`で処理をまとめています。形態素解析の結果を元に、単語の出現回数をカウントします。
また、WordCloudのデータ形式に合わせて、`text`と`value`を持つオブジェクトを作成しています。これは、`d3-cloud`というライブラリに合わせた形なので、適宜変更してください。

今回は、見栄え重視で名詞、固有名詞、サ変接続の単語のみを抽出しています。前述の不要な単語の削除や置換と合わせて、意味のない単語の出現を抑えることができる筈です。
これで、WordCloud用のデータの準備が整いました。

### WordCloudを生成する

WordCloudを生成します。
WordCloudの生成には、ライブラリを用いるのが一般的ですが、WordCloudのレイアウトを計算する手順に興味があったので、自作してみました。

基本的な方針は、単語の出現回数に応じて単語のサイズを変えて配置することです。
単語の配置は、重ならないように単語の位置をランダムに決め、重なりがないように調整を繰り返します。

まずは、単語のサイズを決める関数を作成します。単語の出現回数に応じて、文字のサイズを変えることで、出現回数が多い単語ほど大きく表示されるようにします。

```ts
const calculateFontSize = (
  value: number,
  minVal: number,
  maxVal: number,
  minFontSize: number,
  maxFontSize: number,
) => {
  return ((value - minVal) / (maxVal - minVal)) * (maxFontSize - minFontSize) + minFontSize;
};
```

最小値と最大値の間で、出現回数に応じたフォントサイズを計算します。
`value`の値にバリエーションがないと、似通ったサイズの単語が多くなってしまいます。

次に、単語を配置する関数を作成します。

```ts
type Rect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

/**
 * 矩形同士の衝突判定
*/
const isOverlapping = (rect: Rect, rects: Rect[]) => {
  for (const r of rects) {
    if (
      rect.x < r.x + r.width &&
      rect.x + rect.width > r.x &&
      rect.y < r.y + r.height &&
      rect.y + rect.height > r.y
    ) {
      return true;
    }
  }
  return false;
};

/**
 * 矩形がcanvas内に収まっているか判定
 */
const isWithinCanvas = (rect: Rect, canvas: HTMLCanvasElement) => {
  return (
    rect.x >= 0 &&
    rect.y >= 0 &&
    rect.x + rect.width <= canvas.width &&
    rect.y + rect.height <= canvas.height
  );
};
```

衝突判定を行う関数です。単語の位置をランダムに決めたあと、重なりがないかどうかを判定します。
また、canvas内に収まっているかどうかも判定します。

これらの関数を使って、単語を配置します。

```ts
const width = 800;
const height = 600;

const target = document.getElementById("word-cloud");
const canvas = document.createElement("canvas");
const ctx = canvas.getContext("2d");

canvas.width = width;
canvas.height = height;

ctx.fillStyle = "white";
ctx.fillRect(0, 0, canvas.width, canvas.height); // 背景を塗りつぶす
ctx.textBaseline = "top"; // ベースラインをtopに設定

const rects: Rect[] = []; // 単語の配置を記録する
const MIN_FONT_SIZE = 10;
const MAX_FONT_SIZE = 150;
const minVal = Math.min(...data.map((d) => d.value));
const maxVal = Math.max(...data.map((d) => d.value));
const padding = 5; // 単語間の余白

// 単語を配置
data.forEach((d) => {
  const fontSize = calculateFontSize(d.value, minVal, maxVal, MIN_FONT_SIZE, MAX_FONT_SIZE);
  ctx.font = `normal ${fontSize}px sans-serif`;
  const textWidth = ctx.measureText(d.text).width;
  const textHeight = fontSize;
  const boxWidth = textWidth + padding * 2;
  const boxHeight = textHeight + padding * 2;

  let x, y: number;
  let count = 0;
  while (true) {
    x = Math.random() * (canvas.width - boxWidth);
    y = Math.random() * (canvas.height - boxHeight);

    const rect = { x, y, width: boxWidth, height: boxHeight };

    if (!isOverlapping(rect, rects) && isWithinCanvas(rect, canvas)) {
      rects.push(rect);
      ctx.fillStyle = `rgb(${Math.random() * 240}, ${Math.random() * 240}, ${Math.random() * 240})`;
      ctx.fillText(d.text, x + padding, y + padding);
      // 当たり判定が見たい場合は以下のコメントアウトを外す
      // ctx.strokeRect(rect.x, rect.y, rect.width, rect.height);
      break;
    }

    count++;
    // 100回試行しても重なる場合は描画を諦める
    if (count > 100) {
      console.error("Failed to place text");
      break;
    }
  }
  target.appendChild(canvas);
});
```

Canvasを作成して単語を配置します。
単語の配置はランダムな位置を決め、重なりがないかどうかを判定しています。
この実装では完全にランダムな位置を指定しており、あまり効率が良い実装ではありません。
（100回以上重なる場合は描画を諦めるようにしています）
他にも一定回数以上重なる場合はサイズを変えるなど工夫することで、試行回数を減らすことができるでしょう。

以上の実装から生成したWordCloudが、こちらです。

#### [Japanese Cluster](https://bsky.app/profile/jaz.bsky.social/feed/cl-japanese)フィード

![word-cloud.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3512467/5edf90e2-4528-5f4e-6bf5-2398ecab682f.png)
ちょうど観光バス記念日だったので、観光が入っています。
バスは一般名詞だったので、今回は除外されていました。

#### [原神](https://bsky.app/profile/shimoju.jp/feed/aaadkjp33cr7y)フィード

![word-cloud2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3512467/767a9f97-f6e7-4e12-374b-149e635cc40b.png)
精確な分かち書きができていないため、意味のない単語が多いです。
なんとなくガイアが人気なことはわかります。

#### [買っちゃった！](https://bsky.app/profile/itokonn.bsky.social/feed/aaae55kaqvsla)フィード

![word-cloud3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3512467/bd5a0752-7605-0f86-b5b3-140cbc6f5241.png)
ポチったってワードがポチとして抽出されていそうです。
他に、ちょうど大神が新作の発表と旧作のセールで話題になっていた影響が出ていそうです。

### 当たり判定について

単語の中にも単語が配置されているWordCloudを見たことがあるかと思います。
今回は単語の短形を当たり判定に利用しましたが、WordCloud生成ライブラリの[wordcloud2.js](https://github.com/timdream/wordcloud2.js)では、Canvasをgridに分割した2次元配列で管理し、詳細な判定処理を行っていました。
このような方法を取ることで、単語内の空白や余白にも単語を配置できます。

## （追加実装）文字の回転について

前項の実装では、文字の回転までは実装していませんでした。
若干物足りない見た目のWordCloudが生成されていたので、追加で文字の回転を実装してみます。

まずはフラグを用意し、ランダムで回転する単語を決めます。
その際に、当たり判定を考慮し高さと幅を入れ替えます。

```ts
const isVertical = Math.random() > 0.5;
const textWidth = isVertical ? fontSize : ctx.measureText(d.text).width;
const textHeight = isVertical ? ctx.measureText(d.text).width : fontSize;
```

次に文字を配置します。
回転する単語の場合のみ、canvasを回してから配置します。

```ts
if (!isOverlapping(rect, rects) && isWithinCanvas(rect, canvas)) {
  rects.push(rect);
  ctx.save(); // 状態を保存
  ctx.fillStyle = `rgb(${Math.random() * 240}, ${Math.random() * 240}, ${Math.random() * 240})`;
  ctx.translate(x + boxWidth / 2, y + boxHeight / 2); // Canvasの原点を文字の中心にする
  if (isVertical) ctx.rotate(Math.PI / 2); // 90度回転
  // 縦書きの場合はx, yを入れ替える
  const posX = isVertical ? -textHeight / 2 : -textWidth / 2;
  const posY = isVertical ? -textWidth / 2 : -textHeight / 2;
  ctx.fillText(d.text, posX , posY);
  ctx.restore(); // 保存した状態に戻す
  break;
}
```

x, y座標の配置に悩みましたが、これで当たり判定と同じ位置に配置できます。

このコードで生成したWordCloudがこちらになります。

![word-cloud5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3512467/ef23725a-285d-0faa-f153-56dfcbfec900.png)
![word-cloud6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3512467/97620264-e606-b638-c4d3-cafd950cb00f.png)


ついでにフォントの変更、一般名詞も追加して出力してみました。
前項で掲載したWordCloudより、ぎっしりと敷き詰められた気がします。

## 最後に
今回はBlueSkyのAPIを利用してフィードを取得し、WordCloudを生成する方法を紹介しました。

BlueSkyのAPIは、データを取得して表示するといった基本的な実装を試みてみるのに適しています。
Web開発の初学者にもオススメですので、ぜひ試してみてください。

WordCloudの実装はシンプルですが、計算量の削減や見た目の調整など、まだだ改良の余地があります。
ライブラリ毎に当たり判定の処理や配置のアルゴリズムが異なりますので、興味があれば調べてみてください。

ここまで読んでくださりありがとうございました。

## 参考
- [Bluesky Documentation](https://docs.bsky.app/docs/get-started)
- [kuromojin](https://github.com/azu/kuromojin)
- [wordcloud2.js](https://github.com/timdream/wordcloud2.js)
- [WordCloudとkuromojiを使ったアプリケーション](https://www.design.kyushu-u.ac.jp/~morimoto/teaching/materials/Extra.html)
- [React + kuromoji.js + D3-CloudでWordCloudをブラウザに描画](https://takumon.com/wordcloud-with-kuromoji-d3cloud-react/)
- [きれいな Word Cloud を作るには](https://note.com/takibi333/n/nd9b8fcf4c3cd)

## 一緒に二次元業界を盛り上げていきませんか？
株式会社viviONでは、フロントエンドエンジニアを募集しています。

https://vivion.jp/recruit/

また、フロントエンドエンジニアに限らず、バックエンド・SRE・スマホアプリなど様々なエンジニア職を募集していますので、ぜひ採用情報をご覧ください。
