# AGENTS.md

Link Index の改修・拡張を行うエージェント / 開発者向けガイド。仕様の一次情報は実装（`index.html`）そのもの。利用者向け説明は [README.md](README.md) を参照。

## このプロジェクトの本質

- **単一 HTML ファイルのアプリ**。`index.html` に HTML / CSS / JavaScript をすべて内包する
- **ビルドなし・依存なし・サーバーなし**。npm も bundler も使わない
- **個人運用 / 単一ユーザー**。同時編集・共有・コラボレーションは要件外

## 構成ファイル

| ファイル      | 役割                                                   |
| ------------- | ------------------------------------------------------ |
| `index.html`  | アプリ本体（CSS は `<style>`、JS は末尾の `<script>`） |
| `favicon.svg` | ファビコン                                             |
| `README.md`   | 利用者向けドキュメント                                 |
| `CLAUDE.md`   | `AGENTS.md` へのシンボリックリンク                     |

## 動かす / 確認する

ビルド・テストランナーはない。ブラウザで開いて手動確認する。

```bash
python3 -m http.server 8000
# http://localhost:8000 を開く
```

`file://` でも動くが、クリップボード API 等を使う機能は HTTP 配信での確認を推奨。データはブラウザの localStorage に保存されるため、状態をリセットしたいときは DevTools で `link-index:v1` キーを削除するか、アプリの `Settings ▸ Data ▸ Clear All` を使う。

## アーキテクチャ

`<script>` 内は 3 層に分かれている。新しいコードもこの分離を保つ。

- **state** — 単一の `state` オブジェクトが UI の全状態を持つ（`links` / `groups` / `query` / `tag` / `view` / `editingId` / `formTags` / 各メニューの開閉など）。DOM 参照は `els` に集約
- **render** — `render()` を起点に `renderStats` / `renderTags` / `renderGroups` / `renderBoard` / `renderList` などが `state` から DOM を再構築する。差分更新はせず、毎回 `replaceChildren` で描き直す方針
- **persistence** — `load()` で localStorage から復元、`persist()` で書き戻す。`normalizeLink` / `normalizeUrl` / `normalizeTags` / `cleanGroup` が読み込み・保存時のデータ整形を担う

データを変更する操作の基本形は「`state` を更新 → `persist()` → `render()`」。

## 守るべき不変条件（重要）

- **クライアント完結を崩さない** — サーバー通信・外部 fetch・外部スクリプト読み込みを追加しない。title 推定はクリップボードと URL 文字列からのみ行う
- **外部依存を増やさない** — 現状 CDN を含む外部リソースをゼロにしている。フォントも system スタックのみ。ライブラリ追加は単一ファイル・依存なしの方針に反する
- **単一ファイルを保つ** — JS / CSS を別ファイルに分割しない
- **localStorage スキーマはバージョン管理** — キーは `link-index:v1`。保存形式を破壊的に変える場合はキーの version を上げ、`load()` で旧形式を移行する
- **`no group` センチネル** — 未分類は定数 `NO_GROUP`（`"no group"`）で表す。`cleanGroup()` を通して正規化する
- **必須バリデーション** — リンクは title と URL が必須。URL はスキーマ未指定時に `normalizeUrl()` で `https://` を付与する

## 主要なシンボル（index.html 内）

| 名前                                                    | 役割                                              |
| ------------------------------------------------------- | ------------------------------------------------- |
| `STORAGE_KEY` / `NO_GROUP` / `DEFAULT_GROUPS`           | 定数                                              |
| `sampleLinks`                                           | 初回起動時のサンプルデータ                        |
| `state` / `els`                                         | アプリ状態 / DOM 参照                             |
| `load` / `persist` / `normalize*`                       | 永続化・データ整形                                |
| `getFilteredLinks` / `compareLinks`                     | 検索クエリ + タグでの絞り込み・並び順の決定       |
| `enableGridReorder` / `commitLinkOrder`                 | Board カードの D&D 並び替え（`order` / `pinnedOrder`） |
| `commitGroupOrder`                                      | グループ管理モーダルでのグループ D&D 並び替え     |
| `render` ほか `render*`                                 | 描画                                              |
| `openModal` / `closeModal` / `saveForm`                 | リンク追加・編集モーダル                          |
| `openGroupsModal` / `commitGroupRename` / `deleteGroup` | グループ管理（追加・リネーム・削除）              |
| `inferTitleFromUrl` / `parseHtmlLink`                   | title 自動推定（SharePoint 対応含む）             |
| `exportData` / `importData` / `applyImport`             | JSON の入出力                                     |
| `clearData` / `restoreSamples` / `renderStorageUsage`   | Data モーダル（全削除・サンプル復元・使用量表示） |
| `openHelpModal` / `closeHelpModal`                      | Help モーダル（使い方・ショートカット一覧）       |
| `showToast`                                             | 一時的な通知（保存失敗・容量超過など）            |

## コードスタイル

- 既存コードに合わせる: 2 スペースインデント、ダブルクォート、関数宣言ベース、`const`/`let`
- DOM はテンプレート文字列の `innerHTML` ではなく `createElement` + テキストノードで組み立てる（XSS 回避。SVG など固定マークアップのみ `innerHTML` を使用）
- 文字列を画面に出すときは `textContent` を使い、ユーザー入力を HTML として解釈させない

## 仕様上の判断

以下は意図的な設計判断（あえて実装しない決定）。改修時の参考に。

- **最近開いたリンク（実装しない）** — `lastOpenedAt` は `openLink()` で記録するが、Board への「最近開いたリンク」表示は **実装しない方針**。`renderBoard` に Recent セクションを足さないこと（Board 上部に固定表示するのは `renderFavorites` の Pinned のみ）
- **note は編集ダイアログのみ（一覧に出さない）** — note の確認・編集は追加/編集モーダルでのみ行う運用。`renderList` に note 列を**足さない**（ただし全文検索 `getFilteredLinks` の対象には含む）
- **フォント** — system フォントスタックのみを使用し、Google Fonts などの外部フォントは読み込まない（外部依存ゼロを優先）
- **未分類は Inbox 扱い** — `no group` のリンクは List / All セクションの**先頭**に**登録が新しい順**で表示する（仮登録 → すぐグループ設定する運用のため）。`groupSortRank()` が NO_GROUP を常に先頭に置き、`compareFallback()` が未分類のみ createdAt 降順にする
- **並び替えは Board のみ** — リンクの D&D 並び替え UI は Board のグループセクションと Pinned セクションだけに置く。List ビューには並び替え UI を**足さない**（List は手動順で表示されるのみ）。検索・タグ絞り込み中と All セクションでは D&D を無効にする（非表示リンクがあると並び順を確定できないため）。並び順は関連度ソートにせず、絞り込み中も手動順を維持する

## 変更時の確認観点

手動テストの最小セット:

1. リンクの追加 / 編集 / 削除 / 開く / ピン留めが動く
2. 検索・タグ絞り込みが入力即時で反映され、ハイライトされる
3. Board / List の切り替えと、空グループの非表示
4. URL 貼り付け時の title 自動入力（プレーン URL / HTML リンク / SharePoint URL）
5. Board でカードを D&D してグループ内・Pinned 内の並び順が変わり、リロード後も維持される。グループ管理モーダルの D&D でグループ順が変わる
6. Export → Clear All → Import(Replace/Merge) でデータが往復する
7. リロード後も localStorage から状態が復元される
8. 900px / 560px 以下でのレイアウト崩れがない
