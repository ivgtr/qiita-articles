## Motion とは

MotionはFramer MotionとMotion Oneが統合されて生まれたアニメーションライブラリです。
元々Framer（ノーコードのWebサイト作成ツール）が提供していたアニメーションライブラリでしたが、作者のMatt Perry氏がFramerから独立したタイミングで、ライブラリも独立しました。

Motionはユーザーの操作や状況に併せたアニメーションを簡単に実装できるのが特徴です。

Framer MotionはReact向けのアニメーションライブラリでしたが、MotionではVanilla JS向けのAPIが提供されています。
Motion Oneの機能がMotionとして提供されている形ですが、Framer MotionはMotion Oneを単純にラップしているだけでもないので、同等の機能を提供するわけではありません。

本記事では、Vanilla JS向けのAPIを使ってUIアニメーションの実装方法を紹介します。

## 環境構築

環境構築はViteのテンプレートを使って行います。以下のコマンドを実行して、プロジェクトを作成します。

```bash
npm create vite@latest my-motion -- --template vanilla
cd my-motion
npm install motion
```

CDNを使って読み込むこともできます。今回はViteを使っていますが、LPなど簡易的なWebサイトの制作ではCDN呼び出しを使っても良さそうです。

```html
<script src="https://cdn.jsdelivr.net/npm/motion@11.11.17/dist/motion.js"></script>
<script>
  const { animate, scroll } = Motion
</script>
```

ESモジュールを直接読み込むこともできます。

```js
import { animate, scroll } from "https://cdn.jsdelivr.net/npm/motion@11.11.17/+esm";
```

:::note
ライブラリはとても軽量で、すべての機能を使ってもgzip圧縮されたサイズは約17kbです。GSAP（約23kb）と比較しても軽量ですし、tree-shakingも効くので、よりサイズは小さくなります。
:::

## バージョン確認

```bash
"motion": "^11.11.17"
```

## Motion の基本的な使い方

もっとも簡単にMotionを使う方法は、`animate`関数を使ってアニメーションを定義することです。

```js
import { animate } from "motion";

// Element(s)
const box = document.getElementById("box");

animate(box, { opacity: 0 }, { duration: 0.5 }); // 0.5秒でopacityを0にする

// Selectors
animate("div", { x: [0, 100] }, { ease: "easeIn" }); // 100px右に移動
```

第一引数にアニメーションを適用する要素、第二引数にアニメーションのプロパティ、第三引数にオプションを定義します。

アニメーションのプロパティには数値を配列で渡すことで、アニメーションの開始と終了の値を指定できます。

オプションからは、アニメーションの挙動を調整できます。この記事ではUI実装にフォーカスするので、設定値の詳細は割愛します。詳しい項目は公式ドキュメントの[animate](https://motion.dev/docs/animate)項を参照してください。

## 便利な関数


### `scroll`

`animate`関数から返される`AnimationPlaybackControls`を渡すことで、スクロール時のアニメーションを実装できます。

```js
import { animate, scroll } from "motion";

const box = document.getElementById("box");
const animation = animate(box, { opacity: 0 });

scroll(animation);
```

アニメーションの進行度は、スクロールの進行度と連動します。第二引数でオプションを渡すことで、連動する対象の要素やスクロールの軸、進行度の範囲を指定できます。

### `inView`

要素がビューポートに入った際、callback関数を実行する便利関数です。実装的にはIntersection Observer APIのラッパーかと思われます。

```js
import { inView } from "motion";

const box = document.getElementById("box");

inView(box, () => {
  console.log("in view");
});
```

`inView`に渡す関数は、通常要素がビューポートに入った際、一度だけ実行されます。このcallback関数内で関数を返すことで、要素がビューポートから外れた際に実行される関数を定義できます。

```js
inView(box, () => {
  console.log("in view");

  return () => {
    console.log("out of view");
  };
});
```

この場合、ビューポートを出入りするたびにcallback関数が実行される様になります。

### `stagger`

複数要素をズラしてアニメーションしたい場合、`stagger`関数を利用することで簡単に実現できます。

```js
import { animate, stagger } from "motion";

animate(
  ".item",
  {
    opacity: [0, 1],
    translateY: [100, 0],
  },
  {
    duration: 0.5,
    type: "spring",
    delay: stagger(0.1),
  }
);
```

### `spring`

`spring`関数を使うことで、スプリングアニメーションを実装できます（ばねのような挙動）



```js
import { animate, spring } from "motion";

animate(div, { x: 100 }, { type: spring });
```

※motion/miniではimportする必要がありますが、motionではデフォルトで使えるアニメーションです（`type: "spring"`としても同じ）



## UIアニメーションの為のMotion

Motionの基本的な使い方を紹介しましたが、これだけでもアニメーション実装で十分な機能が揃っていることがわかりました。
ここからは、実際にUIアニメーションを実装してみます。

### 通知アニメーション

簡単な通知表示にアニメーションを適用してみました。

[CodePen](https://codepen.io/tenorichan/pen/NPKKmzQ)

`animate`関数はPromiseを返してくれるので、ユーザーのアクションに応じたアニメーションを簡単に実装できます。

```js
async delete(id) {
  const targetEl = document.querySelector(`.notify[data-id="${id}"]`);
  await animate(targetEl, { opacity: [1, 0], x: [0, 100] });
  this.notifyArray = this.notifyArray.filter((item) => item.id !== id);
  this.render();
}
```

### ハンバーガーメニュー

ハンバーガーメニューのアニメーションを実装してみました。

[CodePen](https://codepen.io/tenorichan/pen/ExWzZjv)

ここでもPromiseが活用できます。
`stagger`関数を用いることで細かい計算を放棄できる点も嬉しい。

### いいねボタン

`animate`関数でSVGをアニメーションさせることもできます。

[CodePen](https://codepen.io/tenorichan/pen/ExWzZjv)


単純な移動からfillの変更、flubberのような（2点間のパスを補完してくれる）ライブラリを用いることで、モーフィングアニメーションも実現できます。


## まとめ

元々Framer MotionはReactで宣言的に書けるのが強みのライブラリでしたが、MotionのVanilla JS向けAPI（Motion One）はかなり命令的な書き方になっています。VueやSvelteで利用する場合は、宣言的に書けるようなラッパーがないと辛くなりそうです。一方でペライチLPなどをよりリッチにしたい場合には必要十分で、ユーザーの操作や状況に応じたアニメーションを簡単に実装できるのは魅力的です。

今回は（Framer Motionの記事はたくさんあるので）React向けのAPIである`motion/react`については触れませんでした。Reactで利用する場合にMotionは宣言的にシンプルな記述で自由度の高いアニメーションを実装できるので、オススメのライブラリです。

ドキュメントも豊富で、カスタマイズ要素も多くあるので色々な表現に利用できます。よりリッチなアニメーションを実装したい場合には、是非利用してみてください。