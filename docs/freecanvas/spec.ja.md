# ACR Free Canvas Template Specification (Draft v0.1)

## 1. 概要

Free Canvasは、既存のセクション型（ActiveReports RPX/RDLX互換）テンプレートと並存する、絶対座標ベースのレイアウトモードである。
Free Canvasは以下の用途を主目的とする。

- 帳票運用現場で既存の紙帳票をスキャン/撮影し、そのままレイアウトを再現する（pngbackパイプライン経由のOCR/罫線検出結果インポート）
- セクション（ページヘッダ/詳細/フッタ等）の概念に縛られない自由配置のラベル・受領書・任意フォーマット文書

セクション型と同じ「JSON中間描画モデル → Skiaバックエンド変換」パイプラインを共有し、出力段階（PDF/PNG/ESC-POS/ZPL/SBPL/TPCL）に差異はない。差異はテンプレートの**入力JSON構造**のみに限定される。

## 2. 座標系・単位

セクション型テンプレートの座標系定義をそのまま踏襲する（独自定義を導入しない）。

- 単位: mm（内部的には1/100mm精度の整数値で保持し、丸め誤差を排除）
- 原点: ページ左上 (0, 0)
- 軸方向: X軸=右方向正、Y軸=下方向正
- ページサイズ: `page.width_mm` / `page.height_mm`（既存のページ定義キーを流用）

pngback (`png_to_json.py` / `layout_analyze.py`) は px→mm変換を担当し、`--dpi`と`--paper-size`指定時のスケール計算をFree Canvas側では前提としない。Free Canvas側は常に**変換済みmm値**のみを受け取る。

> 未解決事項: pngback側で「PNGがページ全体をカバーしていない（トリミング済み）」場合の原点ズレ問題（pngback.md記載の既知issue）。この問題が解消されるまで、Free CanvasインポートJSONには `origin_offset_mm: {x, y}` を任意フィールドとして持たせ、ズレを明示的に記録できるようにする（暫定対応案、要協議）。

## 3. トップレベル構造

```json
{
  "schema_version": "freecanvas-0.1",
  "canvas": {
    "page": { "width_mm": 210.0, "height_mm": 297.0 },
    "origin_offset_mm": { "x": 0.0, "y": 0.0 }
  },
  "elements": [ ... ],
  "source": {
    "type": "pngback_ocr",
    "generated_by": "acrpng2json",
    "original_image": "001.png",
    "dpi": 96
  }
}
```

- `schema_version`: Free Canvas専用スキーマであることを明示するバージョン文字列（セクション型JSONの `schema_version` と混同させない）
- `source`: pngback由来かどうかのトレーサビリティ情報（任意、手動作成テンプレートでは省略可）

## 4. 要素配置モデル

セクション型のように「行/バンド」に要素が属するのではなく、各要素が**ページ座標を直接持つ**。

```json
{
  "id": "el_0001",
  "type": "text",
  "x_mm": 12.5,
  "y_mm": 20.0,
  "width_mm": 60.0,
  "height_mm": 6.0,
  "z_index": 0,
  "rotation_deg": 0.0,
  "confidence": 0.94,
  "props": { ... }
}
```

共通フィールド:

| フィールド | 説明 |
|---|---|
| `id` | テンプレート内一意ID |
| `type` | `text` / `line` / `rect` / `image` / `barcode` 等（既存セクション型の要素タイプ定義を再利用） |
| `x_mm`, `y_mm` | 要素バウンディングボックス左上座標 |
| `width_mm`, `height_mm` | バウンディングボックスサイズ |
| `z_index` | 重なり順（省略時は配列順） |
| `rotation_deg` | 回転角（省略時0） |
| `confidence` | OCR/罫線検出由来の場合の信頼度（0.0〜1.0、手動作成時は省略） |
| `props` | 要素タイプ別のプロパティ（既存セクション型の各要素定義に準拠） |

**方針**: `type`と`props`の内部スキーマはセクション型で定義済みのものをそのまま流用し、Free Canvas側で独自の描画プロパティ体系を新設しない。これにより中間描画コマンド生成ロジックの共通化を維持する（実装の重複・分岐を避ける）。

## 5. 罫線（ruled-line）要素

pngbackの罫線検出結果は `type: "line"` および `type: "rect"` 要素として表現する。

```json
{
  "id": "el_0002",
  "type": "line",
  "x_mm": 10.0, "y_mm": 30.0,
  "width_mm": 190.0, "height_mm": 0.0,
  "props": { "stroke_width_mm": 0.2, "orientation": "horizontal" }
}
```

- `orientation` はレンダリング時の最適化ヒントであり、`width_mm`/`height_mm`から実質導出可能なため必須ではない
- 表（テーブル構造）の罫線は個別の`line`要素の集合として表現し、テーブルとしての論理構造（セル結合等）は本v0.1では扱わない（将来拡張: `type: "table"`）

## 6. セクション型との共存規則

1. 1テンプレートファイルは Free Canvas と セクション型 のどちらか一方のモードで、混在は不可（`schema_version`で判別）
2. Free CanvasテンプレートをAcrDesignerで編集後、セクション型へのコンバートは非対応（一方向: pngback → Free Canvas → 手動微修正が基本フロー。セクション型への逆変換は範囲外）
3. 印刷/プレビューの精度保証（1ドットも狂わないWYSIWYG）はFree Canvasにも同一基準で適用する。座標変換・丸め処理はセクション型と共通コードパスを使う

## 7. 未確定事項（要協議）

- `origin_offset_mm` の扱い（暫定案、pngback側のissue解消後に再検討）
- テーブル構造（セル結合を含む罫線の論理グルーピング）
- `confidence`値の閾値によるAcrDesigner上でのUI警告表示の要否
- acr-specリポジトリでの公開範囲（JSON中間モデルのスキーマ自体は公開方針と一致するが、pngback連携部分をどこまで公開するか）
