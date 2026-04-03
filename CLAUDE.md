# Maison Ma Manière — テーマ開発ログ

## プロジェクト概要

**ブランド:** Maison Ma Manière  
**テーマ:** Dawn ベースカスタムテーマ v4.x.x  
**リポジトリ:** https://github.com/Masaki-setsugenn-web3/mmm.git  
**デプロイ:** GitHub main ブランチ → Shopify テーマ自動反映  
**運用ルール:** `main` へのプッシュが即本番反映のため、**push は毎回確認を取ること**

---

## ディレクトリ構成（カスタム部分）

```
sections/
  mmm-snap-detail.liquid    # スナップ詳細ページ（メタオブジェクトテンプレート）
  mmm-snap-list.liquid      # スナップ全件一覧ページ
  mmm-staff-profile.liquid  # スタッフ個別ページ（メタオブジェクトテンプレート）
  mmm-featured-snap.liquid  # トップ等に埋め込むスナップフィーチャーセクション

templates/metaobject/
  staffsnap.json            # StaffSnap メタオブジェクト用テンプレート定義
  staff.json                # Staff メタオブジェクト用テンプレート定義

snippets/
  social-icons.liquid       # SNS アイコン（Instagram SVG を参照元として使用）
  delivery-time.liquid      # 配送時間表示ウィジェット
  desktop-menu.liquid       # デスクトップナビゲーション
  facets.liquid             # コレクションフィルタ
  facets-vertical-top.liquid
  line-add-friend-banner.liquid
  meta-tags.liquid          # OGP / SEO メタタグ

locales/
  en.default.schema.json    # 英語スキーマ翻訳
  ja.schema.json            # 日本語スキーマ翻訳
```

---

## メタオブジェクト設計

### StaffSnap（投稿データ）

| フィールド | 型 | 説明 |
|---|---|---|
| `snap_image` | list.file_reference | スナップ画像（複数枚可） |
| `thumbnail_image` | file_reference | サムネイル用（将来的に廃止検討） |
| `comment` | multi_line_text | コメント |
| `post_date` | date | 投稿日 |
| `related_products` | list.product_reference | 着用商品 |
| `staff_ref` | metaobject_reference → Staff | スタッフ参照 |

### Staff（スタッフ名簿）

| フィールド | 型 | 備考 |
|---|---|---|
| `name` | single_line_text | — |
| `height` | single_line_text | 将来的に `number` 型への変更を推奨 |
| `introduction` | multi_line_text | 自己紹介文 |
| `profile_photo` | file_reference | アバター画像 |
| `instagram_url` | url | オブジェクト型：`.url` でURLを取得 |

### Liquid フィールドアクセスの注意点

```liquid
snap.staff_ref.value           {# Staff メタオブジェクト本体 #}
snap.staff_ref.value.name      {# 文字列（.value 不要） #}
snap.staff_ref.value.profile_photo  {# 画像オブジェクト（.value 不要） #}
snap.snap_image.value          {# 画像の配列（list 型は .value 必須） #}
staff.instagram_url.url        {# URL 文字列（url 型は .url でアクセス） #}
```

---

## ページ URL 設計

```
/pages/staffsnap              全スナップ一覧（mmm-snap-list.liquid）
/pages/staffsnap/{handle}     スナップ詳細（mmm-snap-detail.liquid）
/pages/staff/{handle}         スタッフ個別（mmm-staff-profile.liquid）
```

> **Shopify 管理画面での設定:** コンテンツ → メタオブジェクト → Staff → 「Webページを有効化」を ON にすること。各 Staff エントリに `/pages/staff/{handle}` が自動生成される。

---

## 変更ログ

---

### 2026-04-03 — 初回コミット（ヘルスチェック全修正 ＋ Staff Snap システム構築）

#### A. テーマ全体ヘルスチェック修正

**`snippets/meta-tags.liquid`**
- OGP 画像 URL の `http:` → `https:` 修正（セキュリティ）

**`snippets/delivery-time.liquid`**
- `* { outline: 0; }` → `.dropdown:focus { outline: 0; }` — **重大 a11y バグ修正**（全要素のフォーカスリングを消していた）
- 8 箇所のハードコードカラー → CSS カスタムプロパティ（`rgb(var(--color-base-text))` 等）に置換

**`snippets/desktop-menu.liquid`**
- インラインスタイルから `!important` を 5 箇所削除

**`snippets/facets.liquid`**
- `aria-expanded="true"` → `"false"` に修正（2 箇所）
- `onclick` インライン JS → `data-close-facet-drawer` / `data-close-facets-wrapper` に変更
- イベント委譲パターンの `{% javascript %}` ブロックを追加

**`snippets/facets-vertical-top.liquid`**
- 同上（aria-expanded、onclick 廃止）

**`snippets/line-add-friend-banner.liquid`**
- ハードコードカラー（`#000`, `#E00000`, `#FFFFFF`）→ CSS カスタムプロパティ
- `@media (prefers-reduced-motion: reduce)` ブロック追加

**`sections/mini-cart.liquid`**
- クローズボタン 3 箇所に `aria-label` 追加
- `onclick` → `data-close-details` 属性 + `{% javascript %}` イベント委譲

**`sections/main-product-modal.liquid`**
- クイックビュークローズボタンの `onclick` 廃止、`data-close-details` に変更

**`sections/collage.liquid`**
- 動画再生ボタンに `aria-label` 追加

**`sections/hero-movie.liquid` / `add-heromovie-section.liquid` / `add-hero-movie.liquid`**
- `"presets"` が欠落していたためテーマエディタに表示されなかった問題を修正

**`locales/en.default.schema.json` / `locales/ja.schema.json`**
- `sections.mmm_snap_list.filter_all` 翻訳キーを追加

---

#### B. Staff Snap システム — 新規構築

**`sections/mmm-snap-detail.liquid`** — 全面再設計
- ギャラリー形式（メイン画像 ＋ サムネイル列）
- サムネイルクリックでメイン画像を JS で切り替え（`data-src` ＋ Vanilla JS）
- Instagram アイコン：インライン SVG（Feather Icons 系 stroke スタイル）
- スタッフ名・アバター → `/pages/staff/{handle}` へのリンク
- Null セーフティ：全フィールドに `!= blank` チェック
- `snap_image.value` の list 型を for ループで展開（`media_type` 判定バグを撤廃）

**`sections/mmm-snap-list.liquid`** — 構造改善
- JS によるフィルタリング（DOM 非表示）を完全削除
- スタッフアバターナビゲーション行を追加（`<nav>` 要素、`shop.metaobjects.staff.values` から生成）
- 各アバターは `/pages/staff/{handle}` へのリンク
- アバター画像なしの場合は名前の頭文字プレースホルダーを表示

**`sections/mmm-staff-profile.liquid`** — 新規作成
- Staff メタオブジェクトテンプレート用セクション
- プロフィールヘッダー（アバター、名前、身長、自己紹介、Instagram アイコン）
- `staff.name` で snaps をフィルタリングして一覧表示
- 「View All Snaps」ボタン（`/pages/staffsnap` へ戻る）

**`templates/metaobject/staff.json`** — 新規作成
- Staff メタオブジェクトの Webページ機能テンプレート定義
- `mmm-staff-profile` セクションを割り当て

**`sections/mmm-featured-snap.liquid`**
- `media_type` 判定バグ修正（`if target.media_type != 'image'` を削除、for ループのみで取得）

---

## 今後のロードマップ

### Phase 1（優先）
- [ ] Shopify 管理画面で Staff メタオブジェクトの「Webページ機能」を有効化
- [ ] Staff フィールド `height` を `single_line_text` → `number` 型に変更（身長ソートのため）
- [ ] `thumbnail_image` フィールドの廃止検討（`snap_image` の 1 枚目で代替可能）

### Phase 2（スナップ 50 件超えたら）
- [ ] `{% paginate shop.metaobjects.staffsnap.values by 24 %}` でサーバーサイドページネーション実装
- [ ] `image_url` に `widths:` + `sizes:` を追加（Retina 対応 srcset 自動生成）

### Phase 3（本格 SEO 施策）
- [ ] 各スナップ詳細ページに OGP メタタグ追加（`snap.comment` をディスクリプションに）
- [ ] Storefront API 移行検討（大量データ対応）

---

## 開発ルール

- `main` ブランチへの push = 本番反映。**必ず確認を取ってから実行**
- CSS は `mmm-` プレフィックスで名前空間を分離
- カラーは `rgb(var(--color-base-text))` / `rgb(var(--color-base-background))` を使用（ハードコード禁止）
- JS は Vanilla のみ（外部ライブラリ禁止）、`{% javascript %}` タグを使用
- `onclick` インライン JS 禁止 → `data-*` 属性 ＋ イベント委譲
- 新規セクションには必ず `"presets"` を定義すること（テーマエディタ表示のため）
