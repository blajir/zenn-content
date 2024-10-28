---
title: "Shopify の Checkout UI extensions で設定可能なテキスト UI を作る"
emoji: "🛒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["shopify"]
published: true
publication_name: "kauche"
---

## はじめに
Shopify の Checkout UI extensions を使用すると、チェックアウトページの各種項目（商品情報、配送、支払いなど）にバナーやカスタムテキストといった独自の要素を設定できます。これにより、注意事項やプロモーション情報を効率よくユーザーに伝えることが可能です。

この記事では、Checkout UI extensions で作成したアプリのテキスト部分とスタイルを Shopify のチェックアウトエディタから設定できるようにし、使い回しが利く拡張機能を作成する方法と、その際につまずいたことについて書こうと思います。

## Checkout UI extensions とは
Checkout UI extensions とは、Shopify が提供するチェックアウト環境を安全にカスタマイズするための機能に含まれる、「チェックアウトページと注文状況ページにカスタム UI や任意のコンテンツを追加することができる機能」になります。
例えば、チェックアウトページに返品や交換ができないことを示すお知らせを表示したい、といった場合に、Checkout UI extensions でカスタム UI を作成し、アプリとして設置することで実現できます。

:::message
Checkout UI extensions は Shopify Plus プランに加入しているマーチャントのみ利用可能です。
https://shopify.dev/docs/api/checkout-ui-extensions
:::

## 拡張機能を作成する
ここでは前提条件として、Shopify パートナーアカウントは作成済みとします。最終的には下記のような拡張機能を作成します。
![](/images/shopify-checkout-editor-field/2024-10-27-16.55.17.png)
*Shopify のチェックアウトエディタ画面*

### アプリを作成する
まずは下記手順でアプリを構築していきます。

1. `npm install -g @shopify/cli@latest` で Shopify CLI をインストール
2. `shopify app init` でアプリを作成
3. project name を入力
  a. ここでは `kauche-checkout-ui` としています
4. Remix(recommended) を選択
5. TypeScript を選択

### 拡張機能の雛形を作成する
次にアプリ内に拡張機能を作成していきます。

1. `cd {project name}` で作成されたアプリ配下に移動
2. `shopify app generate extension` コマンドで拡張機能を作成
  a. ログインが求められるので、ブラウザで Shopify パートナーアカウント情報でログインする
3. Shopify の新規アプリとして作成するか質問されるので、`Yes, create it as a new app` を選択し、アプリ名を入力
4. 拡張機能のタイプ `Checkout UI` を選択
5. 拡張機能の名前を入力
  a. ここでは `text-field` としています
6. Typescript React を選択

![](/images/shopify-checkout-editor-field/2024-10-27-17.08.35.png =320x)
*extensions配下に作成した拡張機能が生成される*

ここまで実行すると、extensions 配下に拡張機能のアセットが生成されます。
ちなみに同一アプリ内に複数 extension を作成したいときは、2. のコマンドを都度実行すれば可能です。

### 設定可能なテキスト UI を作る
では本題の設定可能なテキスト UI の作成をしていきます。ここでは、Shopify が Checkout UI extentions のコンポーネントとして提供している [Text コンポーネント](https://shopify.dev/docs/api/checkout-ui-extensions/2024-01/components/titles-and-text/text)を拡張していきます。

@[card](https://shopify.dev/docs/api/checkout-ui-extensions/2024-01/components/titles-and-text/text)

また、Text コンポーネントのドキュメントを読むと、`size` や `emphasis`, `appearance` といったコンポーネントの視覚的なスタイルを変更可能な props を受け取ることが分かりますので、これらもチェックアウトエディタ上から設定できるようにします。

#### `checkout.tsx` に Text コンポーネントをインポートする
`checkout.tsx` を次のように変更します。

```tsx:checkout.tsx
import type { TextProps } from "@shopify/ui-extensions-react/checkout";
import {
  reactExtension,
  Text,
  useSettings,
} from "@shopify/ui-extensions-react/checkout";
import { ExtensionSettings } from "@shopify/ui-extensions/checkout";

interface Settings extends ExtensionSettings {
  description: string;
  appearance: TextProps["appearance"];
  emphasis: TextProps["emphasis"];
  size: TextProps["size"];
}

export default reactExtension("purchase.checkout.block.render", () => (
  <Extension />
));

function Extension() {
  const { description, appearance, emphasis, size } = useSettings<Settings>();
  return (
    <Text
      appearance={appearance}
      size={size}
      emphasis={emphasis}
    >
      {description}
    </Text>
  );
}
```

このコードでは、Text コンポーネントをインストールし、[`useSettings` フック](https://shopify.dev/docs/api/checkout-ui-extensions/2024-10/apis/settings)を使用して、チェックアウトエディタから設定した拡張機能の情報を取得しています。取得した値は Text コンポーネントに渡します。

#### `shopify.extension.toml` で拡張機能の設定をする
次に、`shopify.extension.toml` ファイルを次のように変更します。

```shopify.extension.toml
api_version = "2024-07"

[[extensions]]
type = "ui_extension"
name = "text-field"
handle = "text-field"

[[extensions.targeting]]
module = "./src/Checkout.tsx"
target = "purchase.checkout.block.render"

[extensions.capabilities]
api_access = true

[extensions.settings]
  [[extensions.settings.fields]]
  key = "description"
  type = "single_line_text_field"
  name = "Text description"
  description = "Enter a description for the text."
  [[extensions.settings.fields]]
  key = "appearance"
  type = "single_line_text_field"
  name = "Text appearance"
    [[extensions.settings.fields.validations]]
    name = "choices"
    value = '["accent", "decorative", "subdued", "info", "base", "success", "warning", "critical"]'
  [[extensions.settings.fields]]
  key = "emphasis"
  type = "single_line_text_field"
  name = "Text emphasis"
    [[extensions.settings.fields.validations]]
    name = "choices"
    value = '["italic", "bold"]'
  [[extensions.settings.fields]]
  key = "size"
  type = "single_line_text_field"
  name = "Text size"
    [[extensions.settings.fields.validations]]
    name = "choices"
    value = '["extraSmall", "small", "base", "large", "extraLarge", "medium"]'
```

`[[extensions.targeting]]` でチェックアウト画面内のどこに拡張機能をレンダーするかをコントロールすることができます。ここで指定している `purchase.checkout.block.render` という値は、特定のチェックアウトセクションに関連づけられることなく、チェックアウトエディタ上で指定した場所にレンダリングできます。その他、指定可能なターゲットについては[こちら](https://shopify.dev/docs/api/checkout-ui-extensions/2024-10/extension-targets-overview)をご覧ください。

`[extensions.settings]` 以下に、チェックアウトエディタで設定可能なフィールドを定義します。ここでは、`key` と `type`, `name` という必須のプロパティを設定しています。

- `key`：拡張機能コードから設定値にアクセスするために使用される文字列
- `type`：保存できる情報のタイプ
- `name`：チェックアウトエディタでアプリユーザーに表示される設定名

`[[extensions.settings.fields.validations]]` ではテキストやその他のプロパティに対する入力のバリデーションを定義できます。ここでは、`value` の値のみ選択できるようにバリデーションすることで、Text コンポーネントが取り得る値のみを受け付けます。

その他、設定可能な type や validations.name は[こちら](https://shopify.dev/docs/api/checkout-ui-extensions/2024-10/configuration#settings-definition)をご覧ください。

### アプリをデプロイする
アプリを Shopify のストアにインストールするために、以下のコマンドでパートナーダッシュボードにアプリをデプロイします。

```
npm run deploy
```

デプロイに成功するとパートナー管理画面からアプリが閲覧できるようになります。
![](/images/shopify-checkout-editor-field/2024-10-28-19.22.57.png)
*Shopify パートナー画面にデプロイしたアプリが表示される*

アプリがデプロイできたら、Shopify のストアにアプリをインストールします。パートナーアカウントにインストールしたいストアが紐づいている場合、「アプリをテストする」で開発ストアにインストールすることができますし、複数ストアを持っていてそうでない場合などは「リンクを管理」を押下すると、アプリ配布用の URL が取得できます。

![](/images/shopify-checkout-editor-field/2024-10-28-19.23.13.png)
*アプリ概要ページからインストール方法が確認できる*

## チェックアウトエディタからアプリを設定する
ストアにインストールしたアプリをチェックアウト画面に設定していきます。
Shopify の管理画面から `設定 > チェックアウト > カスタマイズ` で、チェックアウトエディタに移動します。

セクションナビゲーションの最下部にある「アプリブロックを追加」を押下すると、インストールしたアプリに含まれる拡張機能が表示されるので、今回作成した拡張機能 `text-field` を選択します。

![](/images/shopify-checkout-editor-field/2024-10-27-18.51.56.png =240x)
*アプリブロックとして作成した拡張機能を選択できる*

選択すると shopify.extension.toml で定義したフィールドが表示されるので、各種フィールドの値を設定していきます。

![](/images/shopify-checkout-editor-field/2024-10-27-19.00.38.png)
*text-field のフィールドを設定できる*

また、shopify.extension.toml のターゲット設定で任意の箇所にアプリブロックを埋め込めるように設定してあるので、セクションナビゲーション上でドラッグ & ドロップで表示位置を変更することも可能です。

![](/images/shopify-checkout-editor-field/2024-10-28-19.34.36.png)
*ドラッグ & ドロップでアプリブロックの表示位置を変更できる*

## つまずいたポイントと解決法
Checkout UI extensions でアプリを作成するにあたって、つまずいたことや役に立ったものを箇条書きします。

- 複数パートナーアカウントを持っており、開発モードで異なるパートナーアカウントでログインした際に、ストア選択を切り替えられなくなった
  - `shopify auth logout` コマンドで一旦ログアウトする必要がある
- デフォルトのチェックアウト画面要素の非表示
  - デフォルトで表示される要素を非表示にしたい場合がありますが、現在の仕様ではこれは不可能でした([Github Issue](https://github.com/Shopify/ui-extensions/issues/1565))
- 複数人で Checkout UI extensions アプリ開発する場合の `.gitignore` の設定
  - Shopify コミュニティの[この投稿](https://community.shopify.com/c/extensions/app-development-source-control-for-multiple-developers/m-p/2462572/highlight/true)が参考になりました

## 終わりに
本記事では、テキスト表示やスタイルのカスタマイズができるアプリを構築するプロセスとともに、実際の開発で私自身がつまずいたポイントや、コミュニティの情報を参考に解決した事例についてご紹介しました。この記事が同じ課題に直面している方々の一助となり、よりスムーズな開発の手助けになれば幸いです。

Checkout UI extensions は、仕様や機能が今後さらに改善されていく可能性があるので、今後のアップデートにも注目していきたいところです。
