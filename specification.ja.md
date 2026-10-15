# ACR テンプレート仕様

## 概要

**ACR（Across Report）** は、JSONをベースにした、プリンタに依存しない帳票テンプレート形式です。

ACRは、レイアウトの記述・描画・出力の生成を分離しています。
同じテンプレートを複数のプラットフォームと出力形式で描画できます。

> *Render once. Output everywhere.*

このドキュメントは、ACRエンジン（`acr_core`）とそのクライアント（`acr-viewer-vscode`、
`acr-viewer`、`acr-cli`）が実際に読んでいるテンプレート形式を記述したものです。
これより前の版にあった、小文字の`x`/`y`/`width`/`height`・`TextBox`コントロール・
`{{ field }}`形式のバインディングを使う（実装されていない）仕様を置き換えます。
`RPX2ACR`変換ツールが出力するテンプレート、および現在のサンプルテンプレートは、すべて
以下の形式に従っています。

### 確認済みの出力形式

| 出力形式 | 説明 |
|--------|-------------|
| PDF | ドキュメント出力 |
| PNG | ピクセル精度のラスタ画像（1ページ1ファイル、ZIPにまとめることも可能） |

SVG・ESC/POS・StarPRNT・SATO・TECといった出力形式は将来の方向性として話題に上ったことは
ありますが、現行エンジンでの実装は確認できていません。これらのフィールドレベルの形式が
確定するまでは、本ドキュメントを根拠にしないでください。

---

## アーキテクチャ

```
テンプレート（JSON）
    ↓
レイアウトエンジン    — セクションの解決、データの結合、位置の計算
    ↓
描画エンジン          — Google Skiaで描画（1ドット精度）
    ↓
出力                  — PDF / PNG
```

### 設計の原則

**プリンタ非依存**
ACRは、プリンタドライバやOSの印刷機能に依存しません。
レイアウトはデバイスに依存しない単位で計算し、出力先の解像度で描画します。

**WYSIWYG精度**
画面上のプレビューは、最終的な印刷結果とピクセル単位で一致します。

**実機不要のプレビュー**
プリンタやドライバがなくても、ピクセル単位で正確なプレビューを確認できます。

**テンプレート形式としてのJSON**
テンプレートは平易なJSONで、描画とは独立して確認・バージョン管理できます。

**Skiaによる描画**
描画エンジンには、ChromeやAndroidにも使われているグラフィックスライブラリGoogle Skiaを
使用しています。

**セクション型の帳票モデル**
ACRは、従来の業務帳票ツールで広く使われているセクション型の構造を採用し、既存の帳票定義
からの移行コストを最小限に抑えています（[design-decisions.ja.md](design-decisions.ja.md)
のADR-003参照）。

---

## 座標系

- **単位:** twip（1/1440インチ。1インチ = 1440 twips）
- **原点:** ページ／セクションの左上
- **X軸:** 右方向が正
- **Y軸:** 下方向が正
- **位置:** コントロールの座標（`Left`/`Top`）は、ページではなく、そのコントロールを
  含むセクションの左上を基準とした相対位置
- 線（Line）コントロールのみ `X1`/`Y1`/`X2`/`Y2` を使用し、これもセクション基準のtwip

---

## テンプレートの構造

テンプレートは、次のルート構造を持つ単一のJSONファイルです。

```json
{
  "ACR": {
    "Meta": { ... },
    "Report": {
      "PageSettings": { ... },
      "Sections": { "Section": [ ... ] }
    }
  }
}
```

レガシーのルートキー`AcReport`も`ACR.Report`の代わりとして受け付けられます
（どちらも同じ`Report`オブジェクトとして解釈されます）。

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `ACR.Meta` | object | | テンプレートのメタ情報（下記）。レイアウト／描画エンジンでは使用されない |
| `ACR.Report.PageSettings` | object | ✓ | 用紙サイズ、向き、余白 |
| `ACR.Report.Sections.Section` | array | ✓ | 帳票セクションの並び順リスト |

### Metaオブジェクト

情報目的のみ。`RPX2ACR`変換時に引き継がれる値で、描画エンジンでは解釈されません。

```json
{
  "Author": "RPX2ACR",
  "CreatedAt": "2026-03-26 15:47:38",
  "LastEditor": "",
  "Source": "RPX",
  "UpdatedAt": "2026-03-26 15:47:38",
  "Version": "1.0"
}
```

---

## PageSettingsオブジェクト

物理的な用紙サイズ・向き・余白を定義します。すべてtwip単位です。

```json
{
  "PaperName": "ユーザー定義のサイズ",
  "PaperSize": 9,
  "PaperWidth": 11906,
  "PaperHeight": 16838,
  "Orientation": 2,
  "TopMargin": 850,
  "BottomMargin": 397,
  "LeftMargin": 680,
  "RightMargin": 567
}
```

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `PaperWidth` | number | 用紙の幅（twip、縦向き基準） |
| `PaperHeight` | number | 用紙の高さ（twip、縦向き基準） |
| `PaperSize` | number | Windowsの`DMPAPER_*`用紙サイズコード |
| `PaperName` | string | 用紙サイズの表示名 |
| `Orientation` | number | `1` = 縦（ポートレート）、`2` = 横（ランドスケープ）。`2`の場合、実効コンテンツ幅・高さは`PaperHeight`×`PaperWidth`（入れ替え）になる |
| `TopMargin` / `BottomMargin` / `LeftMargin` / `RightMargin` | number | ページ余白（twip） |

**注意:** 用紙サイズと異なり、余白はそれぞれ独立したトップレベルのフィールドです。
`margin`のようなネストされたオブジェクトはありません。

---

## セクションモデル

セクションは`ACR.Report.Sections.Section`（配列）に並び、その順序でページに描画されます。
エンジンは`Sections`直下の裸の配列や、`Section`/`Controls`の別形式も受け付けます
（下記「スキーマの許容範囲」参照）。

### 確認済みのセクション種別

| `Type`の値 | 描画タイミング | 説明 |
|--------------|----------|-------------|
| `ACR.PageHeader` | 各ページの先頭 | 列見出し、ページタイトル |
| `ACR.GroupHeader` | 各データグループの先頭（入れ子可） | グループラベル、小計見出し |
| `ACR.Detail` | データレコードごとに1回 | 明細行の本体 |
| `ACR.GroupFooter` | 各データグループの末尾（入れ子可） | グループ小計 |
| `ACR.PageFooter` | 各ページの末尾 | ページ番号、日付 |

`ACR.ReportHeader`・`ACR.ReportFooter`は[design-decisions.ja.md](design-decisions.ja.md)
（ADR-003）で概念としては説明されていますが、これらを使ったサンプルテンプレートは
執筆時点で確認できていません。実際のエンジンで確認するまでは、フィールドレベルの形式は
未確定として扱ってください。

### セクションのフィールド

```json
{
  "Name": "groupheader1",
  "Type": "ACR.GroupHeader",
  "Height": 850,
  "BackColor": 16777215,
  "DataField": "TENPOCD",
  "GroupKeepTogether": 1,
  "KeepTogether": 1,
  "NewPage": 1,
  "RepeatStyle": 4,
  "Control": [ ... ],
  "_tag": "Section"
}
```

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `Name` | string | セクション名。VS Codeプレビューでの表示ラベルとしても使われる |
| `Type` | string | セクション種別（上表参照） |
| `Height` | number | セクションの高さ（twip） |
| `BackColor` | number | 背景色（下記「色の形式」参照） |
| `DataField` | string | グループキーとなるフィールド（`ACR.GroupHeader` / `ACR.GroupFooter`のみ） |
| `GroupKeepTogether`, `KeepTogether`, `NewPage`, `RepeatStyle` | number | 変換元の帳票ツールから引き継がれた、改ページ・グループ制御用のフラグ。正確な意味は本ドキュメントではまだ完全には文書化できていない。既存サンプルの値を参考にすること |
| `rowsPerPage` | number | `ACR.Detail`のみ。`0`は無制限 |
| `Control` | array | セクション内のコントロール一覧（下記） |
| `_tag` | string | 常に`"Section"`。レガシーの判別用フィールドで、そのまま残しておいて問題ない |

### グループの入れ子

`ACR.GroupHeader` / `ACR.GroupFooter`はペアで多段にネストでき、次のような順序で並べます。

```
ACR.PageHeader
ACR.GroupHeader   （外側のグループ）
ACR.GroupHeader   （内側のグループ）
ACR.Detail
ACR.GroupFooter   （内側のグループ）
ACR.GroupFooter   （外側のグループ）
ACR.PageFooter
```

---

## コントロール

コントロールは、セクションの`Control`配列に並ぶ描画要素です。

### スキーマの許容範囲

エンジンは、コントロールを次の3つの形式のいずれでも受け付けます（すべて同じ意味）。

```json
"Control": [ ... ]
"Controls": [ ... ]
"Controls": { "Control": [ ... ] }
```

### 共通フィールド

| フィールド | 型 | 必須 | 説明 |
|-------|------|----------|-------------|
| `Type` | string | ✓ | コントロール種別（下記） |
| `Name` | string | ✓ | コントロール名。`ACR.Field`に表示用の`Text`がない場合、`[Name]`の形でプレースホルダ表示される |
| `Left` | number | ✓ | X位置（twip、セクション左上からの相対位置。`X`も使用可） |
| `Top` | number | ✓ | Y位置（twip、セクション左上からの相対位置。`Y`も使用可） |
| `Width` | number | ✓ | 幅（twip） |
| `Height` | number | ✓ | 高さ（twip） |
| `Style` | string | | CSSライクな宣言のリスト（下記「Style」参照） |
| `_tag` | string | | 常に`"Control"`。レガシーの判別用フィールド |

### 確認済みのコントロール種別

#### `ACR.Label`

データに依存しない固定テキスト（列見出しなど）。

```json
{
  "Type": "ACR.Label",
  "Name": "lblcampany",
  "Text": "会社名：",
  "Style": "font-family: ＭＳ ゴシック; font-size: 9.75pt; font-weight: bold",
  "Left": 0, "Top": 28.34646, "Width": 1133.858, "Height": 226.7716
}
```

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `Text` | string / number | 表示する文字列そのもの |

#### `ACR.Field`

データに紐づくテキスト。描画時の実際の値は`DataField`で指定したデータソースの
フィールドから解決されます。`Text`があっても、それはVS Codeプレビュー上で表示される
デザイン時のサンプル値にすぎず、実際の出力には使われません。

```json
{
  "Type": "ACR.Field",
  "Name": "txtunitcost",
  "DataField": "UNITCOST",
  "OutputFormat": "¥#,##0",
  "CurrencyCulture": "ja-JP",
  "Multiline": 0,
  "Style": "font-family: MS UI Gothic; font-size: 8.25pt; text-align: right; vertical-align: middle",
  "Left": 10771.65, "Top": 28.34646, "Width": 1020.473, "Height": 255.118
}
```

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `DataField` | string | このコントロールが紐づくデータソースのフィールド名 |
| `Text` | string / number | デザイン時のサンプル値のみ（描画には使われない） |
| `OutputFormat` | string | バインドされた値に適用する.NET形式の書式文字列（例: `"yyyy/MM/dd"`, `"¥#,##0"`, `"#,##0.0"`） |
| `CurrencyCulture` | string | `OutputFormat`と組み合わせて使う通貨書式用のロケール（例: `"ja-JP"`） |
| `Multiline` | number | `0` = 1行、`1` = コントロールの範囲内で折り返す |
| `SummaryType`, `SummaryRunning` | number | 集計・連番値用コントロール（ページ番号、総ページ数など）。正確な列挙値の意味は本ドキュメントではまだ完全には文書化できていない。既存サンプルの値を参考にすること |

まだデータレコードに紐づいていない段階（レイアウト計算前のページ番号など）では、
`Text`に初期値・既定値として使われるリテラルを持たせることもできます。

#### `ACR.Shape`

塗りつぶし・枠線のある矩形。行や列の背景として使われることが多い。

```json
{
  "Type": "ACR.Shape",
  "Name": "shape2",
  "BackColor": 14474460,
  "BackStyle": 1,
  "LineColor": 16777215,
  "RoundingRadius": 9.999999,
  "Left": 0, "Top": 283.4646, "Width": 15590.55, "Height": 283.4646
}
```

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `BackColor` | number | 塗りつぶし色（下記「色の形式」参照） |
| `BackStyle` | number | `0` = 透明（塗りつぶしなし）、`1` = `BackColor`で塗りつぶし |
| `LineColor` | number | 枠線の色 |
| `LineStyle` | number | `1` = 実線（既定）、`2` = 破線、`3` = 点線 |
| `RoundingRadius` | number | 角丸の半径（twip）。省略または`0`で角丸なし |

#### `ACR.Line`

直線。`Left`/`Top`/`Width`/`Height`ではなく独自の座標指定を使います。

```json
{
  "Type": "ACR.Line",
  "Name": "line1",
  "X1": 0, "Y1": 1417.323,
  "X2": 15590.55, "Y2": 1417.323,
  "LineColor": 0,
  "LineWeight": 1,
  "LineStyle": 1
}
```

| フィールド | 型 | 説明 |
|-------|------|-------------|
| `X1`, `Y1` | number | 始点（twip、セクション基準） |
| `X2`, `Y2` | number | 終点（twip、セクション基準） |
| `LineColor` | number | 線の色（下記「色の形式」参照） |
| `LineWeight` | number | 線の太さ（twip。実効最小値は1） |
| `LineStyle` / `BorderStyle` | number | `1` = 実線（既定）、`2` = 破線、`3` = 点線 |

### 参照はされているが未確認のコントロール種別

VS Code拡張のデータフィールド機能（`list_data_fields`）は、紐づくフィールドを
`'Text' \| 'Barcode' \| 'Image' \| 'Group'`として分類しており、これは`ACR.Barcode`や
`ACR.Image`（または類似の名前）のコントロール種別がエンジン内に存在することを示唆して
います。執筆時点ではこれらを含むサンプルテンプレートが手元になく、フィールドレベルの
JSON形式はこのドキュメントでは確認できていません。実際のテンプレートやエンジンの
ソースで確認しないまま、（旧版の草案仕様にあった`data`/`symbology`/`src`/`sizing`の
ような）形式を前提にしないでください。

---

## Style

`Style`は、`;`区切りの`プロパティ: 値`のリストで、インラインCSSの`style`属性に近い形式
です。プロパティ名は大文字小文字を区別せずに解釈されます。描画／プレビューエンジンが
解釈するのは次のプロパティのみです。

| プロパティ | 値 | 対象 |
|----------|--------|------------|
| `font-family` | フォント名 | Label / Field |
| `font-size` | 数値 + `pt` | Label / Field |
| `font-weight` | `normal` / `bold` | Label / Field |
| `font-style` | `normal` / `italic` | Label / Field |
| `text-align` | `left` / `center` / `right` | Label / Field |
| `vertical-align` | `top` / `middle` / `bottom` | Label / Field |

`ddo-char-set`などの宣言が含まれることがありますが（変換元の帳票ツールから引き継がれた
もの）、エンジンでは解釈されません。

例: `"font-family: ＭＳ ゴシック; font-size: 9.75pt; font-weight: bold; text-align: center; vertical-align: middle"`

---

## 色の形式

色（`BackColor`、`LineColor`）は、`0xRRGGBB`を表す単一の整数です。
例: `16777215` = `0xFFFFFF` = 白、`0` = `0x000000` = 黒。これは単純なRGB整数であり、
Windowsの OLE 色形式（`0x00BBGGRR`、BGR順）**ではありません**。

---

## データバインディング

関数呼び出しのようなバインディング式をテキストに埋め込む形式とは異なり、ACRは
`Text`にバインディング式を書き込みません。代わりに：

- `ACR.Field`コントロールの`DataField`が、描画値の取得元となるデータソースの
  フィールド名を指定します。
- `ACR.GroupHeader` / `ACR.GroupFooter`セクションの`DataField`が、グループの区切りを
  検出するためのフィールド名を指定します。
- `ACR.Field`上のリテラルな`Text`は、あくまでデザイン時のプレースホルダです。

データソース自体の形式（JSON構造、エンジンへの渡し方、`OutputFormat`/
`CurrencyCulture`と書式文字列の対応関係）は本ドキュメントの対象外であり、エンジンで
確認がとれ次第、別途文書化すべきです。

---

## 描画モデル

```
1. テンプレートJSONを読み込む（ACR.Report、またはレガシーのAcReport）
2. PageSettingsを解決する（Orientation == 2 のとき幅・高さを入れ替え）
3. データソースのレコードをDetail / GroupHeader / GroupFooterセクションに結合する
4. 各グループセクションのDataFieldでグループの区切りを解決する
5. 各セクションのControl一覧をGoogle Skiaで描画する（1ドット/twip精度）
6. PDFまたはPNGへ出力する
```

いずれの段階でもプリンタドライバは不要です。

---

## 実装言語

| 言語 | 状況 |
|----------|--------|
| Rust | リファレンス実装（`acr_core`、非公開リポジトリ） |

---

## 変更履歴

| バージョン | 日付 | 変更内容 |
|---------|------|-------------|
| 1.0 | 2026-02-26 | 初版リリース（未実装の草案スキーマ） |
| 2.0 | 2026-10-15 | `RPX2ACR`が実際に出力し`acr_core` / `acr-viewer-vscode`が読んでいる形式に合わせて全面改訂 |

---

*ACR Specification — acrossreport/acr-spec*
