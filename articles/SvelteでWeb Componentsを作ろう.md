## 🔎 Web Componentsって何？

Web Componentsは、再利用可能なカスタム要素を作成するための一連の技術です。

主に「Custom elements」「Shadow DOM」「HTML templates」から成っており。それらを組み合わせてカスタム要素を作成します。

これらの技術はW3Cで標準化されており、コンポーネント化されたカスタム要素はライブラリに依存せず利用できます。

SwiperやMaterial-UIなどのライブラリは、Web Componentsとして提供されていることがあります。

## 👍️ Web Componentsを利用するメリット

Web Componentsを使うことで、以下のようなメリットがあります。

- ライブラリに依存しない
  - Web Componentsはライブラリに依存せず利用できる
  - ライブラリの流行り廃りに依存しないので将来性がある
- 再利用性が高い
  - 依存性が低いため、他のプロジェクトでも利用しやすい
- 互換性が高い
  - Web Componentsは標準化されているため、ブラウザ間の互換性が高い
  - ポリフィルを使うことで古いブラウザでも利用できる
- 保守性が高い
  - Web Componentsはカプセル化されているため、他のコンポーネントに影響を与えにくい
  - コンポーネントごとにファイルを分割することで保守性が高まる

## 🚶 最小限の実装でWeb Componentsを作る

Web Componentsの実装を理解するために、最小限の実装でWeb Componentsを作ってみます。

以下の例では、`<hello-world>`というカスタム要素を作成します。

```html:index.html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Hello World</title>
</head>
<body>
  <hello-world></hello-world>
  <script>
    // カスタム要素を定義
    customElements.define('hello-world', class extends HTMLElement {
      constructor() {
        super();
        this.textContent = 'Hello, World!';
      }
    });
  </script>
</body>
</html>
```

1. `HTMLElement`を継承したclassを用意
2. `constructor`メソッドでカスタム要素の初期化
3. `this.textContent`でカスタム要素にテキストを追加
4. `customElements.define`メソッドでカスタム要素とclassを紐付ける

以上の手順で、`Hello, World!`と表示されるだけの`<hello-world>`コンポーネントを作成できます。

このように、Web Componentsは簡単な実装でカスタム要素を作成できます。一方で動的な値を扱ったり複雑なUIを実装する場合に、これらの生APIを用いて実装をするとコードが複雑になることがあります。

そのため、Web Componentsをより簡単に実装するために、フレームワークやライブラリを利用するのが一般的です。

代表的なライブラリとしては、[Lit](https://lit.dev/)や[Stencil](https://stenciljs.com/)があります。どちらもAngularのような書き心地で、classベースのAPIでデコレータを用いてWeb Componentsを開発できます。

他にも、オススメの選択肢として[Svelte](https://svelte.dev/)を用いたWeb Components開発があります。

## Svelteで作るWeb Components

Svelteは、VueやReactと比べて記述量が少なく、シンプルな構文でWebアプリケーションを作成できるフレームワーク（もしくはコンパイラー）です。

Svelteで記述したコンポーネントはWeb Componentsとして出力できます。また、Svelteのコアランタイムは非常に軽量で、バンドルサイズも小さいため、Web Componentsとして提供する際にも適しています。

ここからは、Svelteを使ってWeb Componentsを作成する手順を紹介します。

### 環境準備

セットアップはViteのテンプレートを使って行います。以下のコマンドを実行して、プロジェクトを作成します。

```bash
npm create vite@latest my-web-components -- --template svelte-ts
cd my-web-components
```

### コンポーネントを作成

今回はViteでテンプレートを利用した際に存在する`src/lib/Counter.svelte`を使って、Web Componentsを作成します。

このコンポーネントは、クリック数を記録する変数があり、クリックされるたびにカウントアップします。

```svelte:src/lib/Counter.svelte
<script lang="ts">
  let count: number = 0
  const increment = () => {
    count += 1
  }
</script>

<button on:click={increment}>
  count is {count}
</button>
```

SvelteでWeb Componentsを作る場合、`<svelte:options>`タグを追加し、customElement属性でコンポーネントのタグ名を指定します。

```svelte:src/lib/Counter.svelte
<svelte:options customElement="my-counter" />

<script lang="ts">
  let count: number = 0
  const increment = () => {
    count += 1
  }
</script>

<button on:click={increment}>
  count is {count}
</button>
```

SvelteコンポーネントをWeb Componentsとして扱う為のコンポーネント側の設定はこれで終わりです。

### Web Componentsとして出力

次に、ビルド設定を変更します。

アプリを構成する諸々の設定は不要なので、`src/main.ts`の記述を削除して、`Counter.svelte`のみをエクスポートするようにします。

```ts:src/main.ts
export * from "./lib/Counter.svelte";
```

次に`svelte.config.js`の`compilerOptions.customElement`を`true`に設定します。

```js:svelte.config.js
import { vitePreprocess } from "@sveltejs/vite-plugin-svelte";

export default {
	preprocess: vitePreprocess(),
	compilerOptions: {
		customElement: true,
	},
};
```

最後にViteライブラリモードを使ってWeb Componentsを単一のjsファイルとして出力します。

`vite.config.ts`を以下のように設定します。

```ts:vite.config.ts
import { defineConfig } from 'vite'
import { svelte } from '@sveltejs/vite-plugin-svelte'

export default defineConfig({
	build: {
		lib: {
			entry: "./src/main.ts",
			name: "my-components",
			fileName: (format) => `my-components.${format}.js`,
		},
	},
  plugins: [svelte()],
})
```

これで、`npm run build`を実行すると、以下のファイルが出力されます。

- dist/my-components.es.js
- dist/my-components.umd.js

### Web Componentsを利用

前項で出力したjsファイルをHTMLファイルで読み込み、Web Componentsとして利用してみます。

```html:index.html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + Svelte + TS</title>
  </head>
  <body>
    <my-counter></my-counter>
    <script src="/dist/my-components.umd.js"></script>
  </body>
</html>
```

`<my-counter>`タグをHTMLに追加し、`my-components.umd.js`を読み込むことで、Svelteで作成したWeb Componentsを利用できます。

この状態で`npm run dev`を実行して開発サーバーを起動すると、`count is 0`と表示されるボタンが表示されます。また、ボタンをクリックすると、カウントが増えていくことが確認できます。

正しく`Counter.svelte`が表示されていることを確認できました。

再利用性を確かめるために、`my-components.umd.js`の内容をそのまま貼り付けたcodepenも用意しました。

[URL]

## まとめ

今回はSvelteを使ってWeb Componentsを作成する手順を紹介しました。

作成したWeb Componentsをnpmパッケージとして公開することで、他のプロジェクトでも簡単に利用できます。また、cdn経由で読み込むことで、単一のHTML上からでも簡単にWeb Componentsを利用できます。

他にも、SwiperやMaterial-UIなどのライブラリもWeb Componentsとして提供されていることがありますので、ぜひ活用してみてください。

## 参考

- [Web Components](https://developer.mozilla.org/ja/docs/Web/Web_Components)
- [Svelte](https://svelte.dev/)
- [Lit](https://lit.dev/)
- [Stencil](https://stenciljs.com/)
