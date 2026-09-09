---
title: "Fluent UI Blazor v5 の Button をアクセシビリティ実装から読む"
emoji: "♿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Blazor", "FluentUI", "accessibility", "wcag", "a11y"]
published: false
---

## はじめに

ボタンは UI の中でもかなり身近な部品です。送信する、保存する、削除する、メニューを開く、オン / オフを切り替える。どれも小さな見た目に対して、ユーザーの操作結果は大きくなりがちです。

そのため、ボタンのアクセシビリティでは「見た目がボタンっぽい」だけでは足りません。支援技術から見たときに **role / name / state** が適切に伝わること、キーボードで操作できること、フォーカスが見えること、色や状態の意味が視覚的にも十分に伝わることが必要です♿

本記事では、Microsoft Fluent UI Blazor v5 の `FluentButton` と近い Button 系コンポーネントを、**ソースコードベース**で読みます。コンポーネント側が何を出力してくれているのか、逆に利用者が何を設定・確認すべきなのかを整理します。

:::message
この記事は「Fluent UI Blazor を使えば自動的にすべてのアクセシビリティ要件が満たされる」というための記事ではありません。コンポーネント実装が支えてくれる範囲と、アプリケーション側で判断すべき範囲を分けて見るための記事です。
:::

## 本記事のゴール

この記事では、次の観点を整理します。

- ✅ `FluentButton` が出力する HTML と wrapper の責務を理解する
- ✅ `Loading=true` など、例外になる組み合わせや検証済みのパラメーターを確認する
- ✅ テキストあり / アイコンのみ / loading / disabled / toggle / menu / split / anchor の設定ポイントを整理する
- ✅ `Title`、`aria-label`、`aria-labelledby`、可視テキストの使い分けを確認する
- ✅ `Appearance`、`Color`、`BackgroundColor`、`DefaultValues` 変更時のコントラスト確認ポイントを押さえる
- ✅ 自動検出できることと、人間が確認すべきことを切り分ける

読み終えたときに、Fluent UI Blazor v5 で Button 系コンポーネントを使う際の「最低限ここを見る」というチェックリストを持てる状態を目指します。

## 前提条件 / 調査対象

本記事の前提は次のとおりです。

| 項目 | 内容 |
|------|------|
| 🧩 対象ライブラリ | Microsoft Fluent UI Blazor v5 |
| 🌿 対象 upstream | [microsoft/fluentui-blazor `dev-v5`](https://github.com/microsoft/fluentui-blazor/tree/dev-v5) |
| 🔎 確認 commit | [`fff57c781b6e99513883b55277f58db94d100fcc`](https://github.com/microsoft/fluentui-blazor/tree/fff57c781b6e99513883b55277f58db94d100fcc) |
| 📅 確認日 | 2026-09-08 時点の commit として確認 |
| 📦 Web Components | `src/Core.Scripts/package.json` で `@fluentui/web-components` `^3.1.3` を確認 |
| 👀 主な対象 | `FluentButton` / `FluentAnchorButton` / `FluentMenuButton` / `FluentToggleButton` / `FluentSplitButton` |

以降では、**執筆時点で dev-v5 の上記 commit を確認**した内容として記述します。将来の v5 リリースや Web Components 側の更新で実装が変わる可能性があるため、実際の採用時には該当バージョンのソースと生成 DOM を再確認してください。

主に参照した一次情報は次のとおりです。

- [FluentButton.razor](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Button/FluentButton.razor)
- [FluentButton.razor.cs](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Button/FluentButton.razor.cs)
- [FluentAnchorButton.razor](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Button/FluentAnchorButton.razor)
- [FluentMenuButton.razor](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Button/FluentMenuButton.razor)
- [FluentToggleButton.razor](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Button/FluentToggleButton.razor)
- [FluentSplitButton.razor](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Button/FluentSplitButton.razor)
- [FluentComponentBase.cs](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Components/Base/FluentComponentBase.cs)
- [DefaultValues.cs](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Infrastructure/DefaultValues.cs)
- [ButtonAppearance.cs](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core/Enums/ButtonAppearance.cs)
- [Fluent UI Blazor Button documentation](https://www.fluentui-blazor.net/Button)

では、まず `FluentButton` が実際にどのような HTML を出力するのかから見ていきます。

## FluentButton が出力する HTML と wrapper の責務

`FluentButton.razor` を見ると、Blazor コンポーネントは最終的に `<fluent-button>` という Web Component を出力しています。

一部を抜粋すると、アクセシビリティに関係する属性は次のように渡されています。

```razor
<fluent-button
    icon-only="@(IconOnly || (ChildContent is null && Label is null))"
    disabled="@(Disabled || Loading)"
    disabled-focusable="@(DisabledFocusable || Loading)"
    role="@(string.IsNullOrEmpty(Title) ? null : "button")"
    aria-label="@Title"
    title="@Title"
    appearance="@(Appearance == ButtonAppearance.Default ? null : Appearance.ToAttributeValue())"
    @attributes="@AdditionalAttributes">
    @Label
    @ChildContent
</fluent-button>
```

ここで大事なのは、wrapper がすべてを魔法のように判断するのではなく、**いくつかの判断を Blazor 側で属性に変換し、最終的な振る舞いは Web Component とブラウザー / 支援技術の組み合わせに委ねる**構造になっている点です。

### Shadow DOM の中で何が実際に動いているのか

ここからは、`<fluent-button>` というホスト要素だけではなく、実際にその中で何が描かれているかを、**upstream の `@fluentui/web-components` ソース**から見ていきます。

`microsoft/fluentui` の `packages/web-components/src/button/button.template.html` では、ボタンの実体が `shadowrootmode="open"` の Shadow DOM 内で生成されることが書かれています。

```html
<f-template name="fluent-button" shadowrootmode="open">
  <template @click="{clickHandler($e)}" @keypress="{keypressHandler($e)}">
    {{styles}}
    <slot name="start" f-ref="{start}"></slot>
    <span class="content" part="content">
      <slot f-slotted="{defaultSlottedContent}"></slot>
    </span>
    <slot name="end" f-ref="{end}"></slot>
  </template>
</f-template>
```

このコードの意味は、単純に「`<fluent-button>` の中にネイティブの `<button>` が直に書かれている」のではなく、**カスタム要素の Shadow DOM 側でスロットと `span.content` が構築され、その中にコンテンツが配置される**という構造です。つまり、Blazor 側の `<fluent-button>` はホストであり、ブラウザーと支援技術が実際に認識する Accessible Tree は、ホスト要素と Shadow DOM の中身をまとめて評価します。

さらに、`packages/web-components/src/button/button.base.ts` では、実際に `button` としてのロールやキーボード動作が定義されています。

```ts
constructor() {
  super();
  this.elementInternals.role = 'button';
}

public keypressHandler(e: KeyboardEvent): boolean | void {
  if (e && this.disabledFocusable) {
    e.stopImmediatePropagation();
    return;
  }

  if (e.key === 'Enter' || e.key === ' ') {
    this.click();
    return;
  }

  return true;
}
```

`BaseButton` の中には `ElementInternals` もあります。これは、**ネイティブの `button` 要素とほぼ同じように form / label / disabled / aria などの状態を支援技術に伝えるための API**です。

```ts
public elementInternals: ElementInternals = this.attachInternals();

public disabledFocusableChanged(previous: boolean, next: boolean): void {
  if (this.elementInternals) {
    this.elementInternals.ariaDisabled = `${!!next}`;
  }
}
```

ここが重要です。`FluentButton` は素の `<button>` を直接レンダリングしているわけではなく、**Custom Element の中で WAI-ARIA と form-associated custom element の規約を使って `button` として振る舞う**設計になっています。`role='button'`、`aria-disabled`、Enter / Space キーの処理、`form` 関連の fallback control などが、Web Component 側で定義されているのです。

この事実を踏まえると、Blazor 側の `Title` などがホスト要素の属性として渡るのは、**Shadow DOM の中の実体に名前を伝える入口の一部**に過ぎません。つまり、

- Blazor: `aria-label="検索"` / `Title` をホストに設定
- Web Component: `role="button"` と `ElementInternals` で semantics を定義
- Shadow DOM: `slot` と `span.content` で実際のテキスト / icon を組み立てる
- ブラウザー: Accessibility Tree で識別し、screen reader が読み上げる

という流れが実際にあります。

この流れを DevTools で確認すると、Elements タブでは `<fluent-button>` が見え、Accessibility タブでは「button」として扱われているのが見えます。実際には、`<button>` そのものが見えているのではなく、`fluent-button` という host が custom element として role を持ち、Internals の情報が結びついている状態です。

:::message
つまり「`<fluent-button>` は、その名の通り Web Component のホストであり、裏で Shadow DOM の中に `slot` と `span.content` を持ち、アクセシビリティ情報は `ElementInternals` と `role` を通じて公開される」という理解が、実装の正確な見取り図です。
:::

| 観点 | wrapper 側の実装 | 利用者が見るべきこと |
|------|------------------|----------------------|
| 🏷️ 名前 | `Title` があれば `aria-label` / `title` を出力 | 可視テキストとアクセシブルネーム（accessible name）がずれないか |
| 🧩 アイコンのみ | `IconOnly` または空コンテンツで `icon-only` | 可視テキストがない場合の名前付け |
| ⏳ Loading | `disabled` / `disabled-focusable` とスピナーを出力 | 処理中であることが文脈上伝わるか |
| 🚫 Disabled | `disabled` / `disabled-focusable` を出力 | 無効理由や復帰条件が分かるか |
| 🎨 見た目 | `appearance`、`style` に色を出力 | コントラストと状態表現が十分か |
| 🧰 任意属性 | `@attributes="@AdditionalAttributes"` | `aria-labelledby` などを重複なく渡す |

`FluentComponentBase` には次のように `AdditionalAttributes` が定義されています。

```csharp
[Parameter(CaptureUnmatchedValues = true)]
public virtual IReadOnlyDictionary<string, object>? AdditionalAttributes { get; set; }
```

つまり、`aria-labelledby`、`aria-describedby`、`aria-controls`、`aria-haspopup` など、コンポーネントの専用パラメーターになっていない属性も呼び出し側から渡せます。ただし、`Title` は `aria-label` と `title` に変換されるため、同じ意味の属性を複数ルートで指定すると意図がぶれます。**1 つのボタンに対して accessible name の決め方を 1 つに寄せる**のが安全です。

ここでいうアクセシブルネーム（accessible name）は、支援技術が UI 要素を識別するための短い名前です。たとえば「保存する」ボタンなら、見えている「保存する」という文字列がそのまま名前になるのが自然です。`aria-label` や `aria-labelledby` は強力ですが、可視テキストと別の名前を与えると、スクリーンリーダーや音声入力の利用者にとって分かりにくくなる場合があります。

次に、利用パターンごとに「何を必ず設定すべきか」を整理します。

## パターン別: 必須設定と確認ポイント

Button 系コンポーネントは見た目が似ていても、意味は少しずつ違います。通常ボタン、リンク、トグル、メニュー、スプリットボタンでは、支援技術に伝えたい role / name / state が変わります。

### まず全体像を表で見る

| パターン | 使うコンポーネント | 必須に近い設定 | 確認ポイント |
|----------|--------------------|----------------|--------------|
| 📝 テキストありの操作 | `FluentButton` | `Label` または `ChildContent` | 可視テキストが操作内容を表すか |
| 🎯 アイコンのみ | `FluentButton` | `Title` または `aria-label` / `aria-labelledby` | 可視テキストがないため、name を必ず用意 |
| ⏳ 処理中 | `FluentButton Loading="true"` | 元のボタン名、二重送信防止 | スピナーだけに意味を頼らない |
| 🚫 無効 | `Disabled` | 無効理由の説明 | フォーカス不能で困らないか |
| 🧭 無効だが説明したい | `DisabledFocusable` | `Tooltip` や説明文 | キーボード到達時に理由が分かるか |
| 🔁 トグル | `FluentToggleButton` | 安定したラベル、`Pressed` / `Mixed` | 状態が最終的に支援技術（AT）に伝わるか |
| 📂 メニューを開く | `FluentMenuButton` | ボタン名、必要に応じて ARIA 属性 | `aria-haspopup` / `aria-expanded` の最終 DOM |
| ✂️ 主操作 + メニュー | `FluentSplitButton` | `Label` と `Title` | 主操作とメニュー操作の名前が分かれるか |
| 🔗 遷移 | `FluentAnchorButton` | `Href` とリンク名 | 操作ではなく移動として妥当か |

ここからは、それぞれのパターンを少し具体的に見ていきます。

### テキストありの FluentButton

通常のボタンでは、可視テキストをそのままアクセシブルネームにするのが基本です。Microsoft Learn の ARIA Label Error でも、ボタンの修正方法としてまず **button caption text** が挙げられています。

- [ARIA Label Error - Microsoft Learn](https://learn.microsoft.com/windows/win32/winauto/aria-label?WT.mc_id=DT-MVP-5004827#description)

良い例です。

```razor
<FluentButton Appearance="ButtonAppearance.Primary"
              OnClick="@SaveAsync">
    保存する
</FluentButton>
```

`Label` を使う場合も同じ考え方です。

```razor
<FluentButton Label="保存する"
              Appearance="ButtonAppearance.Primary"
              OnClick="@SaveAsync" />
```

避けたい例は、可視テキストと `Title` で別の意味を持たせてしまうケースです。

```razor
@* 避けたい例: 見えている文言と支援技術に渡す名前がずれる *@
<FluentButton Title="削除する"
              OnClick="@SaveAsync">
    保存する
</FluentButton>
```

`FluentButton` では `Title` が `aria-label` にも使われます。WAI-ARIA Authoring Practices の naming guidance では、`button` のように子孫コンテンツから名前を取れる role に `aria-label` を付けると、子孫コンテンツ由来の名前を置き換える点が警告されています。

- [WAI-ARIA APG: Names and Descriptions](https://www.w3.org/WAI/ARIA/apg/practices/names-and-descriptions/)

`Title` は単なる tooltip 用の文字列ではなく、`aria-label` にもなる。ここを意識しておくと、可視テキストと支援技術向けの名前の不一致を避けやすくなります。

### アイコンだけのボタン

`FluentButton.razor.cs` の `IconOnly` パラメーターの XML コメントには、アイコンのみの場合は `Title` または `aria-label` で アクセシブルネームを提供するように書かれています。実装上も、`IconOnly` が `true`、または `ChildContent` と `Label` がどちらもない場合に `icon-only` 属性が出ます。

良い例です。

```razor
<FluentButton IconOnly="true"
              Title="検索"
              OnClick="@SearchAsync">
    <FluentIcon Value="@(new Icons.Regular.Size20.Search())" />
</FluentButton>
```

`aria-labelledby` を使って、画面上の別テキストと関連付ける方法もあります。

```razor
<span id="search-button-label" class="visually-hidden">検索</span>

<FluentButton IconOnly="true"
              aria-labelledby="search-button-label"
              OnClick="@SearchAsync">
    <FluentIcon Value="@(new Icons.Regular.Size20.Search())" />
</FluentButton>
```

避けたい例です。

```razor
@* 避けたい例: 可視テキストもアクセシブルネームもない *@
<FluentButton IconOnly="true"
              OnClick="@SearchAsync">
    <FluentIcon Value="@(new Icons.Regular.Size20.Search())" />
</FluentButton>
```

アイコンの形から意味を推測できる人もいますが、スクリーンリーダーや音声入力では「何のボタンか」を名前で扱います。アイコンだけのボタンでは、`Title`、`aria-label`、`aria-labelledby` のいずれかで自然な名前を用意することを必須として扱うのが安全です。

### Loading のボタン

`FluentButton` の `Loading` は、公式ドキュメントでも「disabled + spinner（無効化 + スピナー）」として説明されています。ソース上でも `disabled="@(Disabled || Loading)"`、`disabled-focusable="@(DisabledFocusable || Loading)"` が出力されます。

アイコンがない `Loading` では、`AddTag` により外側に `span.loading-button` が付き、別の `<fluent-spinner size="tiny" />` が重ねて表示されます。テストの [verified snapshot](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/tests/Core/Components/Button/FluentButtonTests.FluentButton_Loading.verified.razor.html)（検証済みスナップショット）でも、次のような形が確認できます。

```html
<span class="loading-button">
  <fluent-button disabled="" disabled-focusable="" blazor:onclick="x">My button</fluent-button>
  <fluent-spinner size="tiny"></fluent-spinner>
</span>
```

良い例です。

```razor
<FluentButton Appearance="ButtonAppearance.Primary"
              Loading="@isSaving"
              OnClick="@SaveAsync">
    保存する
</FluentButton>
```

Loading 中はスピナーが出ますが、スピナーだけに意味を頼らない方が安全です。フォーム全体で「保存中です」のメッセージを出す、必要に応じて `aria-live` 領域で状態を知らせるなど、画面の文脈に合わせて補助しましょう。

```razor
<FluentButton Appearance="ButtonAppearance.Primary"
              Loading="@isSaving"
              aria-describedby="save-status"
              OnClick="@SaveAsync">
    保存する
</FluentButton>

<p id="save-status" aria-live="polite">
    @(isSaving ? "保存中です。" : "入力内容を保存できます。")
</p>
```

:::message
`FluentButton.razor.cs` の `OnClickHandlerAsync` は `Disabled` のとき `OnClick` を呼ばない実装です。一方、`Loading` はマークアップでは `disabled` になりますが、ハンドラー側の条件は `Disabled` のみです。ブラウザーイベント経路の細部は Web Component / ブラウザーに依存するため、重要な処理ではサーバー側・アプリ側でも二重送信を防ぐ設計にしておくと安心です。
:::

### Disabled と DisabledFocusable

`Disabled` は操作不可の状態を表します。`FluentButton` のソースでは `disabled` 属性に渡され、`OnClickHandlerAsync` でも `Disabled` のときは `OnClick` が呼ばれません。

```razor
<FluentButton Disabled="true">
    保存する
</FluentButton>
```

ただし、無効な理由が分からないボタンはユーザーにとって不親切です。特に「なぜ押せないのか」「どうすれば押せるのか」が重要な場合は、説明を近くに置くか、`DisabledFocusable` と tooltip / 説明文を組み合わせます。

```razor
<FluentButton DisabledFocusable="true"
              Tooltip="必須項目を入力すると保存できます"
              aria-describedby="save-disabled-reason">
    保存する
</FluentButton>

<p id="save-disabled-reason">
    必須項目を入力すると保存できます。
</p>
```

`DisabledFocusable` は、公式ドキュメントの例にもあるように「無効だがフォーカス可能」な状態です。キーボード利用者にも無効理由を伝えたいときに検討できます。

### FluentToggleButton

`FluentToggleButton` は `FluentButton` を継承し、`<fluent-toggle-button>` を出力します。ソースでは `mixed` と `pressed` 属性が出力され、クリック時に `Pressed = !Pressed` してから `base` のハンドラーを呼びます。

```razor
<fluent-toggle-button
    mixed="@Mixed"
    pressed="@Pressed"
    @onclick="@OnClickHandlerAsync">
    @Label
    @ChildContent
</fluent-toggle-button>
```

WAI-ARIA APG の button pattern では、toggle button は `aria-pressed` によって支援技術へ状態を伝えると説明されています。

- [WAI-ARIA APG: Button Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/button/)

一方、Fluent UI Blazor v5 の wrapper は `aria-pressed` を直接出しているのではなく、`pressed` / `mixed` を Web Component に渡しています。最終的にアクセシビリティ API にどう反映されるかは、`@fluentui/web-components`、ブラウザー、支援技術の組み合わせで確認する領域です。

ラベルは状態で変えず、状態は `Pressed` に寄せる方が分かりやすいです。

```razor
<FluentToggleButton Pressed="@isMuted"
                    OnClick="@ToggleMuteAsync">
    ミュート
</FluentToggleButton>
```

避けたい例です。

```razor
@* 避けたい例: ラベル自体が状態で変わるため、toggle の意味が揺れやすい *@
<FluentToggleButton Pressed="@isMuted"
                    OnClick="@ToggleMuteAsync">
    @(isMuted ? "ミュート解除" : "ミュート")
</FluentToggleButton>
```

また、`FluentButton.razor.cs` の `OnParametersSet` では、`FluentToggleButton` に `Loading=true` が指定された場合に `ArgumentException("FluentToggleButton does not support Loading")` が投げられます。toggle で処理中状態が必要な場合は、別の表示や状態管理を設計しましょう。

### FluentMenuButton

`FluentMenuButton` は `FluentButton` を継承し、`<fluent-menu-button>` を出力します。`FluentMenu` の中に置かれた場合は `slot="trigger"` が出力されます。

```razor
<fluent-menu-button
    role="@((Menu != null || !string.IsNullOrEmpty(Title)) ? "button" : null)"
    aria-label="@Title"
    title="@Title"
    slot="@(Menu != null ? @FluentSlot.Trigger : null)"
    @attributes="@AdditionalAttributes">
    @Label
    @ChildContent
</fluent-menu-button>
```

公式 docs でも、`FluentMenuButton` には `FluentButton` の practice が適用されると説明されています。

- [Fluent UI Blazor MenuButton documentation](https://www.fluentui-blazor.net/Button/MenuButton)

WAI-ARIA APG の menu button pattern では、メニューを開くボタンは `role="button"`、`aria-haspopup="menu"` または `true`、表示状態に応じた `aria-expanded` を持つ、と整理されています。

- [WAI-ARIA APG: Menu Button Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/menu-button/)

Fluent UI Blazor wrapper のソースだけを見ると、`aria-haspopup` や `aria-expanded` を明示的に出しているわけではありません。ここは Web Component / `FluentMenu` との連携で最終 DOM とアクセシビリティツリーを確認したい部分です。

```razor
<FluentMenu>
    <FluentMenuButton Title="その他の操作">
        その他
    </FluentMenuButton>
    <FluentMenuList>
        <FluentMenuItem>複製</FluentMenuItem>
        <FluentMenuItem>削除</FluentMenuItem>
    </FluentMenuList>
</FluentMenu>
```

必要な場合は `AdditionalAttributes` で ARIA 属性を渡せますが、Web Component が同じ属性を管理している場合に競合しないよう、最終 DOM を確認してから指定してください。

### FluentSplitButton

`FluentSplitButton` は、`<FluentMenu split="true">` の中に主操作用の `FluentButton` と、メニューを開く `FluentMenuButton` を生成します。ソース上は次の構造です。

```razor
<FluentMenu split="true" @attributes=@AdditionalAttributes>
    <FluentButton slot="primary-action">
        @Label
    </FluentButton>

    <FluentMenuButton aria-label="@Title" />

    @ChildContent
</FluentMenu>
```

ここでは `Label` が主操作側、`Title` がメニューを開く側のアクセシブルネームとして重要になります。つまり、`FluentSplitButton` では **主操作の名前**と**メニューを開く操作の名前**を分けて考える必要があります。

良い例です。

```razor
<FluentSplitButton Appearance="ButtonAppearance.Primary"
                   Label="保存する"
                   Title="保存オプションを開く"
                   OnClick="@SaveAsync"
                   OnMenuClick="@OnSaveMenuClick">
    <FluentMenuList>
        <FluentMenuItem Id="save-copy">コピーとして保存</FluentMenuItem>
        <FluentMenuItem Id="export-pdf">PDF として出力</FluentMenuItem>
    </FluentMenuList>
</FluentSplitButton>
```

避けたい例です。

```razor
@* 避けたい例: メニュー側の名前が空になりやすい *@
<FluentSplitButton Label="保存する"
                   OnClick="@SaveAsync">
    <FluentMenuList>
        <FluentMenuItem>コピーとして保存</FluentMenuItem>
    </FluentMenuList>
</FluentSplitButton>
```

`FluentSplitButton` の `AdditionalAttributes` は外側の `FluentMenu` に渡されます。内側のメニュー開閉ボタンに任意の属性を細かく渡したい場合は、`Title` で足りるか、別構成にした方がよいかを検討してください。

### FluentAnchorButton

リンク風の Button 系コンポーネントとして、v5 ソースでは `FluentAnchorButton` を確認しました。

`FluentAnchorButton` は `<fluent-anchor-button>` を出力し、`href`、`target`、`rel`、`referrerpolicy`、`force-load` などを渡します。

```razor
<fluent-anchor-button
    href="@Href"
    rel="@Rel"
    target="@Target.ToAttributeValue()"
    referrerpolicy="@ReferrerPolicy"
    role="@(string.IsNullOrEmpty(Title) ? null : "link")"
    aria-label="@Title"
    force-load="@(ForceLoad ? "true" : null)"
    @attributes="@AdditionalAttributes">
    @Label
    @ChildContent
</fluent-anchor-button>
```

使い分けの軸はシンプルです。

| 目的 | 推奨 |
|------|------|
| 🔗 ページ遷移・外部 URL を開く | `FluentAnchorButton` |
| ⚙️ 保存・削除・ダイアログ表示などの操作 | `FluentButton` |

WAI-ARIA APG の button pattern でも、ボタンとリンクは異なる機能として説明されています。見た目を揃えたい場合でも、移動なら `link`、操作なら `button` という意味の一致を優先しましょう。

```razor
<FluentAnchorButton Href="/settings"
                    Appearance="ButtonAppearance.Outline">
    設定画面へ移動
</FluentAnchorButton>

<FluentButton Appearance="ButtonAppearance.Primary"
              OnClick="@OpenSettingsDialog">
    設定を変更
</FluentButton>
```

ここまでで role / name / state の入口を整理しました。次は、例外になる組み合わせと検証済みのパラメーターをまとめます。

### パラメータ検証・例外になる組み合わせ

Button 系のパラメーターは多いですが、wrapper 側で明示的に検証しているものと、Web Component へそのまま渡すものがあります。ソース上で確認できる範囲では、次の点を押さえると実装時の迷いが減ります。

| 観点 | ソース上の扱い | 実装時の注意 |
|------|----------------|--------------|
| 🚫 `FluentToggleButton Loading="true"` | `OnParametersSet` で `ArgumentException` | toggle では別の処理中表現を設計する |
| 🎯 `FormTarget` | `_self` / `_blank` / `_parent` / `_top` 以外は `ArgumentException` | フォーム送信先の意図を明確にする |
| 🧩 `ButtonAppearance` / `Shape` / `Size` | nullable で、無効な enum 値は `ToAttributeValue()` 側で空になり得る | 値を動的生成する場合はテストで確認する |
| 🧰 `AdditionalAttributes` | 未一致属性をまとめて出力 | `aria-*` の重複や競合は利用者側で避ける |

たとえば、`FormTarget` は次の値に寄せます。

```razor
<FluentButton Type="ButtonType.Submit"
              FormTarget="_self">
    送信する
</FluentButton>
```

`FluentToggleButton` で `Loading` を使いたい場合は、例外を避けるだけでなく、支援技術に処理中状態がどう伝わるかも別途設計します。

```razor
<FluentToggleButton Pressed="@isMuted"
                    aria-describedby="mute-status"
                    OnClick="@ToggleMuteAsync">
    ミュート
</FluentToggleButton>

<p id="mute-status" aria-live="polite">
    @(isUpdating ? "状態を更新しています。" : "音声のオン / オフを切り替えます。")
</p>
```

次は、見た目に直結する色とコントラストを見ます。

## 色とコントラスト: Appearance と独自 Color / BackgroundColor

`ButtonAppearance` は、v5 ソース上で次の値を持ちます。

| 値 | ソース上の説明 | 実装時の見方 |
|----|----------------|--------------|
| 🎛️ `Default` | default style | 標準の中立的な操作 |
| ◻️ `Outline` | background styling を除去 | 補助的な操作 |
| 🔵 `Primary` | primary action として強調 | 画面内の主操作 |
| 🌫️ `Subtle` | hover / focus まで背景に溶け込む | 軽い操作 |
| 透明 `Transparent` | background と border styling を除去 | ツールバーなどの軽い操作 |

公式 Button docs でも、`Primary` は最も重要な操作に使い、minor actions が多い場合は `Outline` / `Subtle` / `Transparent` を使う考え方が示されています。

- [Fluent UI Blazor Button documentation](https://www.fluentui-blazor.net/Button)

一方、`FluentButton.razor.cs` には `BackgroundColor` と `Color` があり、どちらも `Appearance` を上書きする説明になっています。実装では `style` に `background-color` と `color` が追加されます。

```razor
<FluentButton BackgroundColor="#005A9E"
              Color="#FFFFFF"
              OnClick="@SaveAsync">
    保存する
</FluentButton>
```

独自色を使うときは、Fluent の design token が持っている前提から外れます。ここで WCAG のコントラスト基準を確認します。

> The 3:1 and 4.5:1 contrast ratios referenced in this success criterion are intended to be treated as threshold values. When comparing the computed contrast ratio to the Success Criterion ratio, the computed values should not be rounded.
>
> — [Understanding WCAG 2.2: Contrast (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html)

通常テキストは 4.5:1、大きいテキストは 3:1 が基準です。さらに UI コンポーネントの境界、状態、フォーカスインジケーターなど、テキストではない視覚情報には 1.4.11 Non-text Contrast の 3:1 が関係します。

> Unless the control is inactive, any visual information provided that is necessary for a user to identify that a control is present and how to operate it must have a minimum 3:1 contrast ratio with the adjacent colors.
>
> — [Understanding WCAG 2.2: Non-text Contrast](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html)

:::message alert
`BackgroundColor` / `Color` を指定した時点で、「Fluent がよしなにしてくれるはず」という前提から一歩外れます。文字色、背景色、hover、focus、pressed、disabled、周辺背景との 3:1 / 4.5:1 を、実際の画面で確認してください。
:::

## DefaultValues を変えると、すべての画面の前提が変わる

`FluentComponentBase` のコンストラクターでは、`configuration?.DefaultValues.ApplyDefaults(this)` が呼ばれています。`DefaultValues` では、コンポーネント型ごとに既定値を登録し、インスタンス生成時に反映できます。

たとえば、全体の `FluentButton` を Primary に寄せる設定は可能です。

```csharp
builder.Services.AddFluentUIComponents(config =>
{
    config.DefaultValues
        .For<FluentButton>()
        .Set(p => p.Appearance, ButtonAppearance.Primary);
});
```

ただし、これは画面全体の意味を変えます。`Primary` が多すぎると、どれが主操作なのか分かりにくくなります。さらに配色の前提も変わるため、コントラスト確認も「その画面だけ」ではなく「既定値が効くすべての画面」に広がります。

DefaultValues は便利ですが、アクセシビリティの観点ではグローバルな設計判断として扱うのがよさそうです。

## 支援技術から見る role / name / state

WCAG 2.2 の 4.1.2 Name, Role, Value は、UI コンポーネントの name と role が programmatically determined でき、状態や値が支援技術に伝わることを求めています。

> For all user interface components ... the name and role can be programmatically determined; states, properties, and values that can be set by the user can be programmatically set; and notification of changes to these items is available to user agents, including assistive technologies.
>
> — [WCAG 2.2: 4.1.2 Name, Role, Value](https://www.w3.org/TR/WCAG22/#name-role-value)

Fluent UI Blazor の Button 系では、ざっくり次のように考えると整理しやすいです。

| 要素 | 自動で入りやすいもの | 利用者が明示すべきもの |
|------|----------------------|------------------------|
| 🧩 role | Web Component 側の既定 role、`Title` 指定時の `role` | 見た目と機能がずれる場合の再検討 |
| 🏷️ name | 可視テキスト、`Title` → `aria-label` | アイコンのみ、split のメニュー側 |
| 🔁 state | `disabled`、`pressed`、`mixed` などの属性 | 状態名の自然さ、状態変化の伝わり方 |
| 🔗 relation | 既定では限定的 | `aria-describedby`、`aria-controls`、`aria-labelledby` |

`AdditionalAttributes` を使えば、次のように説明文と関連付けられます。

```razor
<FluentButton aria-describedby="delete-help"
              Appearance="ButtonAppearance.Outline"
              OnClick="@DeleteAsync">
    削除する
</FluentButton>

<p id="delete-help">
    削除後は管理者に依頼しないと復元できません。
</p>
```

`aria-labelledby` を使う例です。

```razor
<h2 id="export-heading">レポート出力</h2>

<FluentButton aria-labelledby="export-heading export-button-text"
              OnClick="@ExportAsync">
    <span id="export-button-text">CSV を作成する</span>
</FluentButton>
```

ただし、可視テキストがある通常ボタンに、別文言の `aria-label` をむやみに付けるのは避けます。音声入力では「画面に見えている名前で操作する」ケースもあります。見えている名前と支援技術上の名前がずれると、ユーザーが操作対象を見つけにくくなります。

名前は「自然言語として適切か」も大事です。`btn1`、`実行`、`OK` だけで文脈が足りない場合は、画面文脈や説明文と合わせて見直しましょう。

## キーボード / フォーカス / ターゲットサイズ

WCAG 2.2 の 2.1.1 Keyboard は、機能がキーボードインターフェースで操作できることを求めています。

> All functionality of the content is operable through a keyboard interface without requiring specific timings for individual keystrokes...
>
> — [WCAG 2.2: 2.1.1 Keyboard](https://www.w3.org/TR/WCAG22/#keyboard)

`<fluent-button>` や `<fluent-anchor-button>` 自体の基本的なキーボード操作は Web Component 側に期待できる範囲です。ただし、アプリケーションとしては次を確認する必要があります。

| 観点 | 確認内容 |
|------|----------|
| ⌨️ Tab 順 | ボタンに自然な順序で到達できるか |
| Enter / Space | ボタン操作がキーボードで実行できるか |
| 🧭 メニュー | `FluentMenuButton` で開閉後のフォーカス移動が自然か |
| 👀 Focus Visible | キーボードフォーカスが見えるか |
| 📏 Target Size | 小さい icon-only ボタンが 24 × 24 CSS px 以上、または例外に合うか |

WCAG 2.2 の 2.4.7 Focus Visible は、キーボードフォーカスが見えることを求めています。

> Any keyboard operable user interface has a mode of operation where the keyboard focus indicator is visible.
>
> — [WCAG 2.2: 2.4.7 Focus Visible](https://www.w3.org/TR/WCAG22/#focus-visible)

また、2.5.8 Target Size (Minimum) は、ターゲットが少なくとも 24 × 24 CSS px であること、または例外に該当することを求めます。

> The requirement is for targets to be at least 24 by 24 CSS pixels in size.
>
> — [Understanding WCAG 2.2: Target Size (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html)

Fluent UI Blazor の `Size` は `Small` / `Medium` / `Large` を持ちます。とはいえ、実際のターゲットサイズは、周辺レイアウト、padding、CSS の上書き、ブラウザー表示倍率にも影響されます。特にツールバーの icon-only ボタンは、実画面で測るのが確実です。

```razor
<FluentButton IconOnly="true"
              Size="ButtonSize.Medium"
              Title="フィルターを開く"
              OnClick="@OpenFilterAsync">
    <FluentIcon Value="@(new Icons.Regular.Size20.Filter())" />
</FluentButton>
```

`Size.Small` を使う場合は、隣接するボタンとの間隔も含めて確認しましょう。小さく見える UI は、マウスだけでなくタッチやスイッチデバイスでも誤操作につながりやすいです。

## 自動検出できること・人間確認が必要なこと

アクセシビリティは自動テストと相性がよい部分もありますが、すべてを自動化できるわけではありません。Button 系で分けると、次のようになります。

| 分類 | 自動検出しやすいこと | 人間確認が必要なこと |
|------|----------------------|----------------------|
| 🏷️ name | icon-only で name が空、`aria-label` 欠落 | 名前が文脈に合っているか |
| 🧩 role | role 不足、誤った role の一部 | ボタンとリンクの意味が機能と合うか |
| 🔁 state | `disabled`、`aria-*` の欠落の一部 | 状態変化が理解しやすいか |
| 🎨 contrast | 文字色 / 背景色の計算可能な違反 | hover / focus / pressed の意味が色だけに依存していないか |
| ⌨️ keyboard | Tab 到達や Enter / Space の一部 | 操作順が自然か、メニュー内移動が期待通りか |
| 📏 target | bounding box の機械測定 | 例外適用が妥当か、実利用で押しやすいか |

たとえば Playwright と axe-core で「アクセシブルネームが空のボタン」を検出することはできます。一方で、「`検索` という名前がこの画面で十分か」「`その他` というメニュー名で操作内容が伝わるか」は、人間が業務文脈を見て判断する領域です。

簡単な確認観点をコードコメントとして残すなら、次のようなチェックリストにできます。

```csharp
// Button accessibility checklist:
// - Icon-only buttons have Title, aria-label, or aria-labelledby.
// - Visible text and accessible name do not conflict.
// - Loading state is not represented by spinner alone.
// - Custom Color / BackgroundColor combinations pass contrast requirements.
// - Toggle buttons expose state correctly in the final accessibility tree.
// - Menu buttons are keyboard operable and expose menu state as expected.
// - Split buttons have separate names for primary action and menu trigger.
```

自動テストで入口を固めつつ、最後はキーボード操作、スクリーンリーダー、ブラウザー DevTools の Accessibility pane で確認する。この組み合わせが現実的だと感じています。

## 実装レビュー用チェックリスト

最後に、実装レビューでそのまま使える形にまとめます。自動テストで入口を押さえつつ、自然言語と操作文脈は人間が確認する前提です。

| 確認対象 | 自動検出しやすいこと | 人間が確認すること |
|----------|----------------------|--------------------|
| 🏷️ 名前 | name が空の icon-only ボタン | 文言が業務文脈に合っているか |
| 👀 可視テキスト | `aria-label` との不一致候補 | 見えている文言と読み上げ名がずれていないか |
| 🔁 状態 | `disabled` / `pressed` / `mixed` の属性 | 状態変化や復帰条件が理解できるか |
| 🎨 色 | 計算可能なコントラスト違反 | hover / focus / pressed が色だけに依存していないか |
| ⌨️ キーボード | Tab 到達、Enter / Space 操作の一部 | メニュー内の移動や戻り先が自然か |
| 📏 サイズ | bounding box の 24 × 24 CSS px 未満 | 例外適用が妥当か、実際に押しやすいか |
| 🧰 属性 | `aria-*` の欠落や重複候補 | Web Component が管理する属性と競合していないか |

特にレビューで見落としやすいのは、**アイコンだけのボタン**、**スプリットボタンのメニュー側の名前**、**独自色を入れたボタンのフォーカス表示**です。ここは自動チェックだけで完了扱いにせず、最終 DOM とアクセシビリティツリーを見て確認するのがおすすめです。

## まとめ

Fluent UI Blazor v5 の Button 系コンポーネントは、`<fluent-button>` などの Web Component に対して、Blazor wrapper が `disabled`、`icon-only`、`aria-label`、`pressed`、`mixed`、`@attributes` などを橋渡しする構造になっています。

この記事で特に押さえておきたい点は次のとおりです。

- `FluentButton` は `Title` を `aria-label` / `title` として出力する
- 可視テキストがある通常ボタンでは、むやみに別の `aria-label` を付けない
- アイコンだけのボタンでは、`Title` または `aria-label` / `aria-labelledby` を必ず用意する
- `Loading` は `disabled` + スピナーになるが、処理中の意味は画面文脈でも補う
- `FluentToggleButton` は `pressed` / `mixed` を Web Component に渡すため、最終的なアクセシビリティツリーを確認する
- `FluentSplitButton` は主操作の `Label` とメニュー側の `Title` を分けて考える
- `Color` / `BackgroundColor` / `DefaultValues` を変えたら、コントラストと意味の前提を再確認する
- 自動検出できる欠落と、人間が判断すべき自然言語・操作文脈を分ける

コンポーネントライブラリは、アクセシビリティの土台をかなり支えてくれます。ただし、最終的に「このボタンは何をするのか」「この状態はユーザーに伝わるのか」「この色と文言で迷わないか」を決めるのは、アプリケーションを作る私たちです。

Fluent UI Blazor の実装を読みながら、wrapper が担う範囲と利用者が担う範囲を分けておくと、レビューでも実装でも迷いが減ります。小さなボタンほど、丁寧に名前・状態・色・フォーカスを確認していきたいですね。

## 参考リンク

- [microsoft/fluentui-blazor dev-v5 commit `fff57c781b6e99513883b55277f58db94d100fcc`](https://github.com/microsoft/fluentui-blazor/tree/fff57c781b6e99513883b55277f58db94d100fcc)
- [Fluent UI Blazor Button documentation](https://www.fluentui-blazor.net/Button)
- [Fluent UI Blazor MenuButton documentation](https://www.fluentui-blazor.net/Button/MenuButton)
- [Fluent UI Blazor ToggleButton documentation](https://www.fluentui-blazor.net/Button/ToggleButton)
- [Fluent UI Blazor SplitButton documentation](https://www.fluentui-blazor.net/Button/SplitButton)
- [Fluent UI Blazor AnchorButton documentation](https://www.fluentui-blazor.net/Button/AnchorButton)
- [Core.Scripts package.json（`@fluentui/web-components` 依存）](https://github.com/microsoft/fluentui-blazor/blob/fff57c781b6e99513883b55277f58db94d100fcc/src/Core.Scripts/package.json)
- [WCAG 2.2: 1.4.3 Contrast (Minimum)](https://www.w3.org/TR/WCAG22/#contrast-minimum)
- [WCAG 2.2: 1.4.11 Non-text Contrast](https://www.w3.org/TR/WCAG22/#non-text-contrast)
- [WCAG 2.2: 2.1.1 Keyboard](https://www.w3.org/TR/WCAG22/#keyboard)
- [WCAG 2.2: 2.4.7 Focus Visible](https://www.w3.org/TR/WCAG22/#focus-visible)
- [WCAG 2.2: 2.5.8 Target Size (Minimum)](https://www.w3.org/TR/WCAG22/#target-size-minimum)
- [WCAG 2.2: 4.1.2 Name, Role, Value](https://www.w3.org/TR/WCAG22/#name-role-value)
- [WAI-ARIA APG: Button Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/button/)
- [WAI-ARIA APG: Menu Button Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/menu-button/)
- [WAI-ARIA APG: Names and Descriptions](https://www.w3.org/WAI/ARIA/apg/practices/names-and-descriptions/)
- [ARIA Label Error - Microsoft Learn](https://learn.microsoft.com/windows/win32/winauto/aria-label?WT.mc_id=DT-MVP-5004827#description)
