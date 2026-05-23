# ACR テンプレート仕様書 v1.0

## 概要

ACR (Across Report) は、JSON ベースのプリンター非依存帳票テンプレート形式を定義します。

ACR はレイアウト定義・レンダリング・出力生成を明確に分離された層として扱います。  
同一テンプレートが、あらゆるプラットフォームおよび出力先で同一の結果を生成します。

> *一度レンダリングすれば、どこへでも出力できる。*

### 対応出力先

| 出力先 | 説明 |
|--------|------|
| PDF | 高精度ドキュメント出力 |
| PNG | ピクセル精度のラスター画像 |
| SVG | スケーラブルベクター出力 |
| ESC/POS | サーマルレシートプリンター |
| StarPRNT | Star Micronics プリンター |
| SATO | SATO ラベルプリンター |
| TEC | 東芝テック プリンター |

---

## アーキテクチャ

```
テンプレート（JSON）
    ↓
レイアウトエンジン     — セクション解決・データバインド・座標計算
    ↓
描画モデル（JSON）     — 中間表現。検査・キャッシュ・転送が可能
    ↓
描画エンジン           — Google Skia によるレンダリング（1ドット精度）
    ↓
出力                   — PDF / PNG / ESC/POS / StarPRNT / SATO / TEC
```

### 設計原則

**プリンター非依存**  
ACR はプリンタードライバーや OS の印刷サブシステムに依存しない。  
レイアウトはデバイス非依存単位で計算され、目的解像度でレンダリングされる。

**WYSIWYG 保証**  
画面上のプレビューは最終印刷出力とピクセル単位で一致する。  
1ドットの狂いも許容しない。

**実機不要プレビュー**  
物理プリンターやドライバーなしに、完全なピクセル精度のプレビューを実現する。

**JSON 中間描画モデル**  
テンプレートも描画モデルも JSON である。  
レンダリングとは独立して検査・キャッシュ・バージョン管理・転送が可能。

**Skia ベースのレンダリング**  
描画エンジンには Google Skia を採用。Chrome・Android と同一のグラフィックスライブラリ。  
全プラットフォームで一貫した高精度出力を保証する。

**ActiveReports 互換セクション構造**  
ACR は ActiveReports と互換性のあるセクションベースのレイアウト構造を採用する。

---

## 座標系

- **単位:** ドット（dot）
- **定義:** 1 dot = 1/DPI インチ  
  例: 203 DPI のプリンターでは 1 インチ = 203 ドット
- **原点:** ページ左上隅
- **X 軸:** 右方向に増加
- **Y 軸:** 下方向に増加
- **位置指定:** すべての座標は絶対座標

---

## テンプレート構造

テンプレートは以下のルート構造を持つ単一の JSON ファイルである。

```json
{
  "version": "1.0",
  "page": { ... },
  "datasource": { ... },
  "sections": [ ... ]
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `version` | string | ✓ | 仕様バージョン。現在は `"1.0"` |
| `page` | object | ✓ | ページサイズとマージンの定義 |
| `datasource` | object | | データバインディング設定 |
| `sections` | array | ✓ | 帳票セクションの順序付きリスト |

---

## ページオブジェクト

論理ページサイズとマージンを定義する。

```json
{
  "width": 2480,
  "height": 3508,
  "unit": "dot",
  "dpi": 300,
  "margin": {
    "top": 118,
    "bottom": 118,
    "left": 118,
    "right": 118
  }
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `width` | number | ページ幅（ドット） |
| `height` | number | ページ高さ（ドット） |
| `unit` | string | 常に `"dot"` |
| `dpi` | number | 目的解像度（例: 300, 203, 96） |
| `margin` | object | ページマージン（ドット）: top / bottom / left / right |

**300 DPI における主要用紙サイズ:**

| 用紙 | 幅（dot） | 高さ（dot） |
|------|-----------|------------|
| A4 | 2480 | 3508 |
| Letter | 2550 | 3300 |
| レシート 80mm | 945 | 可変 |

---

## セクションモデル

ACR は ActiveReports と互換性のあるセクションベースのレイアウトモデルを採用する。  
セクションは定義順に処理され、ページに順次レンダリングされる。

### セクション種別

| セクション | 出力タイミング | 用途 |
|------------|--------------|------|
| `ReportHeader` | 帳票の先頭に1回 | タイトル・ロゴ・帳票メタ情報 |
| `PageHeader` | 各ページの先頭 | 列見出し・ページタイトル |
| `GroupHeader` | 各グループの先頭（ネスト可） | グループラベル・小計ヘッダー |
| `Detail` | データレコードごとに繰り返し | 明細行 |
| `GroupFooter` | 各グループの末尾（ネスト可） | グループ小計 |
| `PageFooter` | 各ページの末尾 | ページ番号・日付 |
| `ReportFooter` | 帳票の末尾に1回 | 総合計・署名欄 |

### セクション定義

```json
{
  "type": "PageHeader",
  "height": 120,
  "canGrow": false,
  "elements": [ ... ]
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `type` | string | セクション種別（上表参照） |
| `height` | number | セクション高さ（ドット） |
| `canGrow` | boolean | コンテンツに合わせて高さを拡張するか |
| `groupKey` | string | グループキーとなるデータフィールド名（GroupHeader / GroupFooter のみ） |
| `elements` | array | セクション内のコントロールリスト |

### グループのネスト

GroupHeader と GroupFooter はネストして多階層グループを表現できる。

```json
[
  { "type": "GroupHeader", "groupKey": "department", "elements": [...] },
  { "type": "GroupHeader", "groupKey": "category",   "elements": [...] },
  { "type": "Detail",                                 "elements": [...] },
  { "type": "GroupFooter", "groupKey": "category",   "elements": [...] },
  { "type": "GroupFooter", "groupKey": "department",  "elements": [...] }
]
```

---

## コントロール

コントロールはセクション内の描画要素である。

### 共通フィールド

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `type` | string | ✓ | コントロール種別 |
| `x` | number | ✓ | X 座標（ドット）: セクション左端からの距離 |
| `y` | number | ✓ | Y 座標（ドット）: セクション上端からの距離 |
| `width` | number | ✓ | 幅（ドット） |
| `height` | number | ✓ | 高さ（ドット） |
| `visible` | boolean | | 表示/非表示。デフォルト: `true` |

---

### TextBox（テキストボックス）

フォントと配置を指定してテキストを描画する。

```json
{
  "type": "TextBox",
  "x": 0,
  "y": 0,
  "width": 1200,
  "height": 80,
  "text": "{{ invoiceTitle }}",
  "font": {
    "family": "IPAex明朝",
    "size": 24,
    "bold": true,
    "italic": false
  },
  "alignment": "center",
  "verticalAlignment": "middle",
  "color": "#000000",
  "canGrow": true
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `text` | string | テキスト内容。`{{ フィールド名 }}` でデータバインド可能 |
| `font.family` | string | フォントファミリー名 |
| `font.size` | number | フォントサイズ（ポイント） |
| `font.bold` | boolean | 太字 |
| `font.italic` | boolean | 斜体 |
| `alignment` | string | `left` / `center` / `right` |
| `verticalAlignment` | string | `top` / `middle` / `bottom` |
| `color` | string | テキスト色（16進カラーコード） |
| `canGrow` | boolean | コンテンツに合わせて高さを拡張するか |

---

### Line（線）

直線を描画する。

```json
{
  "type": "Line",
  "x": 0,
  "y": 118,
  "x2": 2244,
  "y2": 118,
  "lineWidth": 2,
  "color": "#000000"
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `x2` | number | 終点 X 座標（ドット） |
| `y2` | number | 終点 Y 座標（ドット） |
| `lineWidth` | number | 線の太さ（ドット） |
| `color` | string | 線の色（16進カラーコード） |

---

### Rectangle（矩形）

塗りつぶしまたは枠線付きの矩形を描画する。

```json
{
  "type": "Rectangle",
  "x": 0,
  "y": 0,
  "width": 2244,
  "height": 120,
  "lineWidth": 1,
  "borderColor": "#000000",
  "fillColor": "#F0F0F0"
}
```

---

### Image（画像）

画像アセットを描画する。

```json
{
  "type": "Image",
  "x": 0,
  "y": 0,
  "width": 300,
  "height": 300,
  "src": "images/logo.png",
  "sizing": "fit"
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `src` | string | ZIP コンテナ内のパス |
| `sizing` | string | `fit`（アスペクト比維持）/ `fill`（全体塗りつぶし）/ `clip`（クリップ） |

---

### Barcode（バーコード）

バーコードまたは QR コードを描画する。

```json
{
  "type": "Barcode",
  "x": 100,
  "y": 300,
  "width": 600,
  "height": 120,
  "data": "{{ orderCode }}",
  "symbology": "CODE128",
  "showText": true
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `data` | string | バーコードデータ。データバインド可能 |
| `symbology` | string | `CODE128` / `CODE39` / `EAN13` / `EAN8` / `QR` |
| `showText` | boolean | バーコード下部に可読テキストを表示するか |

---

## データバインディング

`text` および `data` プロパティ内で `{{ フィールド名 }}` 構文を使用してデータをバインドする。

```json
{ "text": "請求書番号: {{ invoiceNumber }}" }
{ "text": "合計: {{ formatCurrency(totalAmount) }}" }
{ "text": "{{ pageNumber }} / {{ pageCount }} ページ" }
```

### 組み込み変数

| 変数 | 説明 |
|------|------|
| `pageNumber` | 現在のページ番号 |
| `pageCount` | 総ページ数 |
| `reportDate` | 帳票生成日 |

---

## ZIP コンテナ形式

ACR テンプレートは配布のために ZIP アーカイブとしてパッケージ化できる。

```
template.acr  (ZIP)
├── template.json   ← メインテンプレート定義
├── meta.json       ← テンプレートメタデータ
├── fonts/          ← 埋め込みフォントファイル
└── images/         ← 埋め込み画像アセット
```

---

## レンダリングモデル

```
1. template.json を読み込む
2. datasource をセクションにバインドする
3. グループキーを評価 → セクション繰り返し回数を決定する
4. 要素の座標を計算する（レイアウトエンジン）
5. 描画モデル（JSON）を生成する
6. Google Skia でレンダリングする（描画エンジン）
7. 目的の出力形式に出力する
```

いかなる段階においてもプリンタードライバーは不要である。

---

## 実装言語

ACR は Skia バインディングまたは互換 2D グラフィックスライブラリを持つ任意の言語で実装できる。

| 言語 | 状況 |
|------|------|
| Rust | 参照実装（[acr-engine](https://github.com/acrossreport/acr-engine)） |
| C++ | 計画中 |
| C# | 計画中 |
| WebAssembly | 計画中 |

---

## ActiveReports との互換性

| 機能 | ActiveReports | ACR |
|------|---------------|-----|
| セクションモデル | ✓ | ✓（互換） |
| JSON テンプレート | — | ✓ |
| プリンター非依存 | — | ✓ |
| WYSIWYG 保証 | 部分的 | ✓ |
| 実機不要プレビュー | — | ✓ |

---

## 変更履歴

| バージョン | 日付 | 内容 |
|-----------|------|------|
| 1.0 | 2026-02-26 | 初版リリース |

---

*ACR 仕様書 — acrossreport/acr-spec*
