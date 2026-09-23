# ACR テンプレート仕様 v1.0

## 概要

ACR（Across Report）は、JSONにもとづく、プリンタに依存しない帳票テンプレート形式を定義します。

ACRは、レイアウトの定義・描画・出力の生成を、それぞれ独立した層に分けています。  
同じテンプレートから、どのプラットフォーム・どの出力先でも同一の結果が得られます。

> *Render once. Output everywhere.*

### 対応する出力先

| 出力先 | 説明 |
|--------|------|
| PDF | 高精度な文書出力 |
| PNG | ピクセル単位で正確なラスター画像 |
| SVG | 拡大縮小可能なベクター出力 |
| ESC/POS | サーマルレシートプリンタ |
| StarPRNT | スター精密のプリンタ |
| SATO | SATOのラベルプリンタ |
| TEC | 東芝テックのプリンタ |

---

## アーキテクチャ

```
テンプレート（JSON）
    ↓
レイアウトエンジン    — セクションの解決、データの結合、位置の計算
    ↓
描画モデル（JSON）    — 中間表現。確認・キャッシュが可能
    ↓
描画エンジン          — Google Skiaで描画（1ドット精度）
    ↓
出力                  — PDF / PNG / ESC/POS / StarPRNT / SATO / TEC
```

### 設計の原則

**プリンタ非依存**  
ACRは、プリンタドライバやOSの印刷機能に依存しません。  
レイアウトはデバイスに依存しない単位で計算し、出力先の解像度で描画します。

**WYSIWYGの保証**  
画面のプレビューは、最終的な印刷結果とピクセル単位で一致します。  
1ドットのずれも許しません。

**実機不要のプレビュー**  
プリンタやドライバがなくても、ピクセル単位で正確なプレビューを確認できます。

**中間モデルとしてのJSON**  
テンプレートも描画モデルもJSONです。  
描画とは独立して、確認・キャッシュ・バージョン管理・転送ができます。

**Skiaによる描画**  
描画エンジンには、ChromeやAndroidにも使われているグラフィックスライブラリGoogle Skiaを使用しています。  
これにより、どのプラットフォームでも一貫した高精度の出力を保証します。

**セクション型の帳票モデル**  
ACRは、業務帳票で広く使われているセクション型のレイアウト構造を採用しています。

---

## 座標系

- **単位:** ドット
- **定義:** 1ドット = 1/DPI インチ  
  例: 203 DPIでは、1インチ = 203ドット
- **原点:** ページの左上
- **X軸:** 右に向かって増加
- **Y軸:** 下に向かって増加
- **位置指定:** すべての座標は絶対位置で指定

---

## テンプレートの構造

テンプレートは、次のルート構造を持つ1つのJSONファイルです。

```json
{
  "version": "1.0",
  "page": { ... },
  "datasource": { ... },
  "sections": [ ... ]
}
```

| フィールド | 型 | 必須 | 説明 |
|------------|----|------|------|
| `version` | string | ✓ | 仕様のバージョン。現在は `"1.0"` |
| `page` | object | ✓ | ページサイズと余白の定義 |
| `datasource` | object | | データ結合の設定 |
| `sections` | array | ✓ | 帳票のセクションを順に並べたリスト |

---

## Page オブジェクト

論理的なページサイズと余白を定義します。

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
|------------|----|------|
| `width` | number | ページの幅（ドット） |
| `height` | number | ページの高さ（ドット） |
| `unit` | string | 常に `"dot"` |
| `dpi` | number | 出力先の解像度（例: 300、203、96） |
| `margin` | object | ページの余白（ドット。top / bottom / left / right） |

**300 DPIでの主な用紙サイズ:**

| 用紙 | 幅（ドット） | 高さ（ドット） |
|------|--------------|----------------|
| A4 | 2480 | 3508 |
| Letter | 2550 | 3300 |
| レシート 80mm | 945 | 可変 |

---

## セクションモデル

ACRは、業務帳票で広く使われているセクション型のレイアウトモデルを使用します。  
セクションは順に処理され、ページに順番に描画されます。

### セクションの種類

| セクション | 描画されるタイミング | 説明 |
|------------|----------------------|------|
| `ReportHeader` | 帳票の最初に1回 | タイトル、ロゴ、帳票の情報 |
| `PageHeader` | 各ページの上部 | 列見出し、ページタイトル |
| `GroupHeader` | 各データグループの開始時（入れ子可） | グループ名、小計の見出し |
| `Detail` | データ1件ごとに1回 | 本体の行 |
| `GroupFooter` | 各データグループの終了時（入れ子可） | グループの小計 |
| `PageFooter` | 各ページの下部 | ページ番号、日付 |
| `ReportFooter` | 帳票の最後に1回 | 総計、署名 |

### セクションの定義

```json
{
  "type": "PageHeader",
  "height": 120,
  "canGrow": false,
  "elements": [ ... ]
}
```

| フィールド | 型 | 説明 |
|------------|----|------|
| `type` | string | セクションの種類（上の表を参照） |
| `height` | number | セクションの高さ（ドット） |
| `canGrow` | boolean | 内容に合わせてセクションを伸ばすかどうか |
| `groupKey` | string | グループ化に使うデータ項目（GroupHeader / GroupFooter のみ） |
| `elements` | array | セクション内のコントロールのリスト |

### グループの入れ子

GroupHeader と GroupFooter は入れ子にでき、複数階層のグループを表現できます。

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

コントロールは、セクション内に描画される要素です。

### 共通フィールド

| フィールド | 型 | 必須 | 説明 |
|------------|----|------|------|
| `type` | string | ✓ | コントロールの種類 |
| `x` | number | ✓ | X位置（ドット。セクションの左端から） |
| `y` | number | ✓ | Y位置（ドット。セクションの上端から） |
| `width` | number | ✓ | 幅（ドット） |
| `height` | number | ✓ | 高さ（ドット） |
| `visible` | boolean | | 既定値: `true` |

---

### TextBox

フォントと配置を指定して文字を描画します。

```json
{
  "type": "TextBox",
  "x": 0,
  "y": 0,
  "width": 1200,
  "height": 80,
  "text": "{{ invoiceTitle }}",
  "font": {
    "family": "IPAexMincho",
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
|------------|----|------|
| `text` | string | 文字列。`{{ field }}` によるデータ結合に対応 |
| `font.family` | string | フォント名 |
| `font.size` | number | フォントサイズ（ポイント） |
| `font.bold` | boolean | 太字 |
| `font.italic` | boolean | 斜体 |
| `alignment` | string | `left` / `center` / `right` |
| `verticalAlignment` | string | `top` / `middle` / `bottom` |
| `color` | string | 文字色（16進数） |
| `canGrow` | boolean | 内容に合わせて高さを伸ばす |

---

### Line

直線を描画します。

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
|------------|----|------|
| `x2` | number | 終点のX位置（ドット） |
| `y2` | number | 終点のY位置（ドット） |
| `lineWidth` | number | 線の太さ（ドット） |
| `color` | string | 線の色（16進数） |

---

### Rectangle

塗りつぶし、または枠線の四角形を描画します。

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

### Image

画像を描画します。

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
|------------|----|------|
| `src` | string | ZIPコンテナ内のパス |
| `sizing` | string | `fit` / `fill` / `clip` |

---

### Barcode

バーコードまたはQRコードを描画します。

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
|------------|----|------|
| `data` | string | バーコードのデータ。データ結合に対応 |
| `symbology` | string | `CODE128` / `CODE39` / `EAN13` / `EAN8` / `QR` |
| `showText` | boolean | バーコードの下に文字を表示する |

---

## データ結合

`text` と `data` のプロパティの中で、`{{ fieldName }}` の書き方でデータ項目の値を結合します。

```json
{ "text": "Invoice No: {{ invoiceNumber }}" }
{ "text": "Total: {{ formatCurrency(totalAmount) }}" }
{ "text": "Page {{ pageNumber }} of {{ pageCount }}" }
```

### 組み込み変数

| 変数 | 説明 |
|------|------|
| `pageNumber` | 現在のページ番号 |
| `pageCount` | 総ページ数 |
| `reportDate` | 帳票の作成日 |

---

## ZIPコンテナ形式

ACRのテンプレートは、配布用にZIPアーカイブにまとめることができます。

```
template.acr  (ZIP)
├── template.json   ← テンプレートの本体
├── meta.json       ← テンプレートのメタデータ
├── fonts/          ← 同梱するフォントファイル
└── images/         ← 同梱する画像
```

---

## 描画の流れ

```
1. template.json を読み込む
2. データソースをセクションに結合する
3. グループキーを評価し、セクションの繰り返しを決める
4. 要素の位置を計算する（レイアウトエンジン）
5. 描画モデル（JSON）を生成する
6. Google Skiaで描画する（描画エンジン）
7. 出力先の形式で出力する
```

どの段階でも、プリンタドライバは必要ありません。

---

## 実装言語

ACRは、Skiaのバインディングまたは互換性のある2Dグラフィックスライブラリを持つ言語であれば、どの言語でも実装できます。

| 言語 | 状況 |
|------|------|
| Rust | リファレンス実装（[acr-engine](https://github.com/acrossreport/acr-engine)） |
| C++ | 予定 |
| C# | 予定 |
| WebAssembly | 予定 |

---

## 比較

| 機能 | 従来の帳票ツール | ACR |
|------|------------------|-----|
| セクションモデル | ✓ | ✓ |
| JSONテンプレート | — | ✓ |
| プリンタ非依存 | — | ✓ |
| WYSIWYGの保証 | 一部 | ✓ |
| 実機不要のプレビュー | — | ✓ |

---

## 変更履歴

| バージョン | 日付 | 内容 |
|------------|------|------|
| 1.0 | 2026-02-26 | 初版 |

---

*ACR 仕様 — acrossreport/acr-spec*
