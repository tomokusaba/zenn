---
name: primary-source-verification
description: 技術記事や文章中の主張・コード・API 仕様・数値を「一次情報」で検証する手順とフォーマットを提供するスキル。ファクトチェック、出典確認、要出典タグ付けを依頼されたときに参照する。
---

# 一次情報ファクトチェック スキル

このスキルは、文章中の技術的主張を **一次情報（primary source）** で検証するための定義・手順・出力テンプレートを提供します。Zenn 記事のレビューや、外部記事のファクトチェック依頼を受けたときに必ず参照してください。

---

## 1. 一次情報の定義

検証時は、以下の **一次情報** を優先的に参照します。

| カテゴリ | 例 |
|----------|----|
| 📘 公式ドキュメント | `learn.microsoft.com`、`docs.python.org`、`developer.mozilla.org`、`kubernetes.io/docs` |
| 📦 公式リポジトリ | GitHub の公式 org（`github.com/dotnet`, `github.com/microsoft`, `github.com/openai` 等）の README / ソース / リリースノート |
| 📰 公式ブログ・告知 | `devblogs.microsoft.com`、`openai.com/blog`、`aws.amazon.com/blogs/aws`、製品リリースノート、変更ログ |
| 📜 標準仕様 | RFC（IETF）、W3C 勧告、ECMA、ISO、Unicode、JIS |
| 🎓 学術論文 | arXiv、ACM、IEEE、Nature 等の査読済み論文 |
| 🧪 実機・実コードでの再現 | 自分で動かして確認できるコマンド・スクリプト・出力 |

### 二次情報は依拠しない

以下は **検証の根拠にしない**（参考としての言及は可）。

- 個人ブログ、Qiita、Zenn、Medium、note の他者記事
- Stack Overflow、Reddit、X（Twitter）の投稿
- LLM（ChatGPT、Claude、Copilot 等）の出力そのもの
- Wikipedia（出典として挙げられている一次情報まで遡って検証する）
- 二次配信のニュース記事（必ず元の発表元を当たる）

---

## 2. 検証手順

1. **主張の抽出**: 対象文章を読み、検証対象となる「事実主張」「コード」「API シグネチャ」「数値」「日付・バージョン」を箇条書きで列挙する。意見・所感は対象外。
2. **一次情報の特定**: 各主張について、上記カテゴリのうち最も適切な一次情報の URL を特定する。
   - Microsoft 系は [`microsoft-docs`](../microsoft-docs/SKILL.md) スキルの `microsoft_docs_search` / `microsoft_docs_fetch` を最優先で使う。
   - OSS の挙動は GitHub リポジトリのソース（特定のタグ / コミット）を確認する。
   - 仕様は RFC / W3C を直接参照する。
3. **該当箇所の引用**: 一次情報のうち、主張を裏付ける**最小限の一文**を引用する。引用は原文ママ（翻訳した場合はその旨を明記）。
4. **判定**: 以下のいずれかを付ける。

   | 判定 | 意味 |
   |------|------|
   | ✅ 確認済 | 一次情報と完全に一致 |
   | ⚠️ 要修正 | 表現の揺れ・古い情報・言い過ぎ・不正確だが部分的に正しい |
   | ❌ 誤り | 一次情報と矛盾、または存在しない API / 機能 |
   | ❓ 出典不明 | 一次情報を発見できず検証不能。要出典タグを推奨 |

5. **修正案の提示**: ⚠️ / ❌ には**書き換え案**を添える。

---

## 3. 出力フォーマット

検証結果は次の Markdown 表で返します。

```markdown
## ファクトチェック結果

| # | 判定 | 主張（記事中の表現） | 一次情報 | 引用 / 補足 | 修正案 |
|---|------|----------------------|----------|------------|--------|
| 1 | ✅ | .NET 8 は LTS である | [.NET and .NET Core release lifecycle](https://learn.microsoft.com/dotnet/core/releases-and-support?WT.mc_id=DT-MVP-5004827) | "8.0 ... LTS ... 3 years" | — |
| 2 | ⚠️ | `HttpClient` は毎回 new するのが推奨 | [HttpClient guidance](https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines?WT.mc_id=DT-MVP-5004827) | 実際は `IHttpClientFactory` 推奨 | `IHttpClientFactory` 経由で取得する旨に書き換え |
| 3 | ❌ | Azure Functions は Java 21 をサポートしない | [Azure Functions Java versions](https://learn.microsoft.com/azure/azure-functions/functions-reference-java?WT.mc_id=DT-MVP-5004827) | Java 21 GA 済み | "Java 21 もサポート" に修正 |
| 4 | ❓ | この機能は内部で gRPC を使用 | — | 公式ドキュメントに記述なし | 出典が無いなら削除 or 推測である旨を明記 |

### サマリ
- 確認済: N 件 / 要修正: N 件 / 誤り: N 件 / 出典不明: N 件
- ブロッカー（公開前に必ず直すべき項目）: #2, #3
```

### 注意事項

- 表が長くなる場合は、判定ごと（❌ → ⚠️ → ❓ → ✅）にセクションを分けてもよい。
- 引用 URL は **必ず実際にアクセスして存在を確認**する。LLM の知識から URL を生成しない。
- Microsoft 系ドメイン（`learn.microsoft.com`, `docs.microsoft.com`, `devblogs.microsoft.com`, `techcommunity.microsoft.com`）には MVP トラッキングパラメータ `?WT.mc_id=DT-MVP-5004827` を付与する（[`write-zenn-article` §5.1](../write-zenn-article/SKILL.md) の規約に従う）。
- バージョン依存の主張は **「いつ時点の情報か」を必ず併記**（例: 「2026 年 4 月時点」）。

---

## 4. ツール優先順位

| 用途 | 第一選択 | フォールバック |
|------|----------|----------------|
| Microsoft / Azure / .NET 系 | `microsoft_docs_search` / `microsoft_docs_fetch`（[`microsoft-docs`](../microsoft-docs/SKILL.md) スキル） | `fetch_webpage` で `learn.microsoft.com` を直接取得 |
| GitHub のコード / Issue / Release | `github_repo` ツール | `fetch_webpage` で raw / blob URL を直接取得 |
| 一般 Web の公式ドキュメント | `fetch_webpage` | Web 検索 → 公式ドメインに絞り込み |
| OSS の挙動再現 | ローカルでコマンド実行（`run_in_terminal` / `execution_subagent`） | コンテナで再現 |

---

## 5. やってはいけないこと

- 一次情報を確認せずに「✅ 確認済」を付ける
- LLM の事前知識のみで判定する（必ず URL を引く）
- 検索結果のスニペットだけで判定する（リンク先全文を読む）
- 二次情報（Qiita 等）を一次情報として扱う
- 翻訳された一次情報のみで判定する（原文がある場合は原文も確認）
