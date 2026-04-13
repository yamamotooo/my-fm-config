# FileMaker コンフィグ画面　アドオン

## プロジェクト概要

FileMaker Pro 向けアドオンのコンフィグ画面（WebViewer）を Vite + Vue 3 + JavaScript で実装する。

---

## 技術スタック

```
Vite + Vue 3 + JavaScript
vite-plugin-singlefile  ← build時に単一HTMLファイルに出力
```

### ビルド要件
- `vite build` で **単一HTMLファイル**（HTML + CSS + JS インライン）を生成
- `dist/index.html` を FileMaker WebViewer に埋め込む

### ファイル構成
```
src/
  App.vue     ← コンフィグ画面（全実装）
  main.js
  style.css
```

---

## FileMaker ↔ WebViewer 連携仕様

### データの流れ

```
① FileMaker が WV を表示
② FileMaker が 0.1秒後に fm_setConfig で統合データを渡す
③ ユーザーが設定して保存 or キャンセル
   → FileMaker.PerformScriptWithOption("mfs-Callback", json, "0")
```

### Vue 側で公開するグローバル関数

```javascript
// FileMaker → Vue
window.fm_setConfig(json)
```

#### fm_setConfig に渡す JSON スキーマ

```json
{
  "fieldData": {
    "InstanceUUID": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "ConfigJSON": "{ ... }"
  },
  "fieldMetaData": [
    { "name": "FieldName", "result": "text", "type": "normal" }
  ],
  "portalMetaData": {
    "PortalName": [
      { "name": "PortalName::FieldName", "result": "text", "type": "normal" }
    ]
  },
  "recordId": "33"
}
```

- `fieldData.ConfigJSON` : 保存済みのコンフィグ文字列（初回は渡さない）
- `fieldData.InstanceUUID` : アドオンインスタンスの識別子
- `fieldMetaData` : レイアウト上のフィールド一覧（FileMaker API 形式）
- `portalMetaData` : ポータル名をキーとしたフィールド一覧
- `recordId` : FM_Config テーブルのレコードID

### Vue → FileMaker

```javascript
// execute() ラッパー経由で呼び出す
execute('mfs-Callback', JSON.stringify({
  action: '保存' | 'キャンセル',
  data: {
    fieldData: {
      InstanceUUID: '...',
      ConfigJSON: '{ ... }'   // 現在の configJSON を JSON 文字列化したもの
    },
    recordId: '33'
  }
}), 0)
```

### FileMaker 側での処理（mfs-Callback）

```
action = "保存"    → data.fieldData.ConfigJSON を ConfigJSON フィールドに保存
action = "キャンセル" → 何もせずウインドウを閉じる
```

### execute() — FileMaker スクリプト呼び出しラッパー

WebViewer 描画直後は `FileMaker` オブジェクトが未定義のため、
利用可能になるまで最大 3 秒ポーリングする。

```javascript
execute(scriptName, param, option)
// ブラウザ環境では console.log にフォールバック
```

---

## ConfigJSON スキーマ（現行版）

`fields` は**配列**。FileMaker の JSON 関数がキーをアルファベット順に並べ替えるため、
オブジェクトではなく配列にして表示順を保持する。

```json
{
  "title": "アドオン設定画面",
  "targetLayouts": "顧客一覧",
  "targetTableOccurrence": "Customers",
  "tabs": [
    {
      "id": "basic",
      "label": "基本",
      "fields": [
        { "key": "mode",          "value": "record", "required": true,  "type": "toggle",       "label": "モード",             "options": [{ "value": "record", "label": "レコード" }, { "value": "portal", "label": "ポータル" }] },
        { "key": "portalName",    "value": "",       "required": true,  "type": "portal-select","label": "ポータル",           "showWhen": { "field": "mode", "value": "portal" } },
        { "key": "uniqueField",   "value": "",       "required": true,  "type": "field-select", "label": "ユニークフィールド" },
        { "key": "sortOrderField","value": "",       "required": true,  "type": "field-select", "label": "並び順フィールド" },
        { "key": "labelField",    "value": "",       "required": true,  "type": "field-select", "label": "ラベルフィールド" },
        { "key": "showHandle",    "value": true,     "required": true,  "type": "switch",       "label": "ハンドル表示" },
        { "key": "recordLimit",   "value": null,     "required": false, "type": "number",       "label": "レコード上限", "placeholder": "未入力の場合は全件取得" }
      ]
    },
    {
      "id": "options",
      "label": "オプション",
      "fields": [
        { "key": "imageField",     "value": "", "required": false, "type": "field-select", "label": "画像フィールド" },
        { "key": "thumbnailField", "value": "", "required": false, "type": "field-select", "label": "サムネイルフィールド" },
        { "key": "labelField2",    "value": "", "required": false, "type": "field-select", "label": "ラベルフィールド2" },
        { "key": "labelField3",    "value": "", "required": false, "type": "field-select", "label": "ラベルフィールド3" }
      ]
    },
    {
      "id": "theme",
      "label": "テーマ",
      "fields": [
        { "key": "colorMain",   "value": "#0077cc", "required": false, "type": "color", "label": "メインカラー" },
        { "key": "colorAlt",    "value": "#f5f5f5", "required": false, "type": "color", "label": "サブカラー" },
        { "key": "colorAccent", "value": "#ff6600", "required": false, "type": "color", "label": "アクセントカラー" }
      ]
    }
  ]
}
```

### フィールドの共通プロパティ

| プロパティ | 必須 | 説明 |
|---|---|---|
| `key` | ✓ | フィールド識別子 |
| `value` | ✓ | 現在値 |
| `required` | ✓ | バリデーション対象か |
| `type` | ✓ | UIコンポーネント種別（下記） |
| `label` | ✓ | 表示ラベル |
| `options` | toggle のみ | `[{ value, label }]` |
| `showWhen` | 任意 | `{ field: key, value: 値 }` — 条件付き表示 |
| `placeholder` | number のみ | 入力欄のプレースホルダー |

### サポートする type

| type | UI |
|---|---|
| `toggle` | ボタン2択（options で選択肢定義） |
| `switch` | ON/OFF トグル |
| `field-select` | fieldMetaData からのドロップダウン |
| `portal-select` | portalMetaData からのドロップダウン（showWhen で表示制御） |
| `number` | 数値入力 |
| `color` | カラーピッカー + HEX テキスト入力 |

### 動的レンダリングの設計思想

- Vue 側はフィールド名・タブ構造を一切知らない
- **ConfigJSON のスキーマを変えるだけで別のコンフィグ画面として使いまわせる**
- `showWhen` でフィールド間の依存表示制御が可能
- mode=portal 切替時は `portal-select` / `field-select` を自動リセット

---

## UIレイアウト

```
┌─────────────────────────────────────────┐
│ ヘッダー（タイトル + ✕ボタン）              │
├──────────┬──────────────────────────────┤
│          │                              │
│ サイドバー │  コンテンツエリア              │
│ （タブ）  │  （選択中タブの設定フォーム）    │
│          │                              │
├──────────┴──────────────────────────────┤
│ フッター（ステータス | キャンセル | 保存）   │
└─────────────────────────────────────────┘
```

### バリデーション
- 保存時に `required: true` かつ `showField` を通過したフィールドが空 → 保存ブロック
- `showWhen` 条件に合致しない非表示フィールドはバリデーションをスキップ

---

## デザインシステム

shadcn/ui のデザイン言語を参考にした CSS 変数ベースのスタイル。
`:root` の変数を変更するだけで全体の見た目を調整できる。

```css
:root {
  /* カラー */
  --background:         #ffffff;
  --foreground:         #09090b;
  --muted:              #f4f4f5;
  --muted-foreground:   #71717a;
  --border:             #e4e4e7;
  --input:              #e4e4e7;
  --primary:            #18181b;
  --primary-foreground: #fafafa;
  --secondary:          #f4f4f5;
  --accent:             #f4f4f5;
  --destructive:        #ef4444;
  --ring:               #18181b;
  --sidebar-bg:         #fafafa;
  --radius:             6px;

  /* フォントサイズ 大・中・小 */
  --font-size-lg:       22px;   /* ヘッダータイトル */
  --font-size-md:       14px;   /* 標準テキスト全般 */
  --font-size-sm:       12px;   /* 予約 */

  /* フォントウェイト 極太・太い・普通 */
  --font-weight-heavy:  700;    /* ヘッダータイトル（予約） */
  --font-weight-bold:   500;    /* タブラベル（予約） */
  --font-weight-normal: 400;    /* その他全般（予約） */
}
```

---

## FileMaker テーブル定義（参考）

テーブル名: `FM_Config`

| フィールド名 | 型 |
|---|---|
| z_作成TS | タイムスタンプ |
| z_作成者 | テキスト |
| z_修正TS | タイムスタンプ |
| z_修正者 | テキスト |
| z_削除フラグ | 数字 |
| z_01 | テキスト |
| OnWindowTransaction | テキスト |
| __PK_ID | テキスト |
| InstanceUUID | テキスト |
| ConfigJSON | テキスト |

---

## 開発

```bash
npm run dev    # 開発サーバー
npm run build  # dist/index.html を生成
```

### debugMode フラグ（src/App.vue）

```javascript
const debugMode = true   // 開発時: test データを fm_setConfig に流す
const debugMode = false  // 本番ビルド時（デフォルト）
```

`debugMode = true` にすると `onMounted` 時にテストデータが自動投入される。
ビルド前に必ず `false` に戻すこと。
