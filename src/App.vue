<template>
  <div class="app">
    <!-- ヘッダー -->
    <header class="header">
      <span class="header-title">{{ config.configJSON.title || 'コンフィグ' }}</span>
      <button class="btn-close-header" @click="closeConfig">✕</button>
    </header>

    <!-- メイン -->
    <div class="main">
      <!-- サイドバー（タブ） -->
      <aside class="sidebar">
        <button
          v-for="tab in config.configJSON.tabs"
          :key="tab.id"
          class="tab-btn"
          :class="{ active: activeTabId === tab.id }"
          @click="activeTabId = tab.id"
        >
          {{ tab.label }}
        </button>
      </aside>

      <!-- コンテンツエリア -->
      <section class="content">
        <Transition name="tab-fade" mode="out-in">
          <div :key="activeTabId" class="tab-content">
            <div v-if="activeTab" class="field-card">
              <template v-for="fieldDef in activeTab.fields" :key="fieldDef.key">
                <div v-if="showField(activeTab, fieldDef)"
                  class="field-row"
                  :class="{ error: errors[fieldDef.key] }"
                >
                  <label class="field-label">
                    {{ fieldDef.label }}
                    <span v-if="fieldDef.required" class="required">*</span>
                  </label>

                  <!-- toggle -->
                  <div v-if="fieldDef.type === 'toggle'" class="toggle-group">
                    <button
                      v-for="opt in fieldDef.options" :key="opt.value"
                      class="toggle-btn"
                      :class="{ selected: fieldDef.value === opt.value }"
                      @click="handleChange(fieldDef, opt.value)"
                    >{{ opt.label }}</button>
                  </div>

                  <!-- switch -->
                  <button v-else-if="fieldDef.type === 'switch'"
                    class="toggle-switch" :class="{ on: fieldDef.value }"
                    @click="handleChange(fieldDef, !fieldDef.value)"
                  >
                    <span class="toggle-track"><span class="toggle-knob"></span></span>
                    <span class="toggle-label">{{ fieldDef.value ? 'ON' : 'OFF' }}</span>
                  </button>

                  <!-- portal-select -->
                  <select v-else-if="fieldDef.type === 'portal-select'"
                    :value="fieldDef.value"
                    @change="e => handlePortalChange(fieldDef, e.target.value)"
                  >
                    <option value="">-- ポータルを選択 --</option>
                    <option v-for="p in portalNames" :key="p" :value="p">{{ p }}</option>
                  </select>

                  <!-- field-select -->
                  <select v-else-if="fieldDef.type === 'field-select'"
                    v-model="fieldDef.value"
                    @change="markDirty"
                  >
                    <option value="">{{ fieldDef.required ? '-- 選択してください --' : '-- なし --' }}</option>
                    <option v-for="f in activeFields" :key="f" :value="f">{{ f }}</option>
                  </select>

                  <!-- number -->
                  <input v-else-if="fieldDef.type === 'number'"
                    type="number" min="1"
                    :value="fieldDef.value ?? ''"
                    :placeholder="fieldDef.placeholder ?? ''"
                    @input="e => handleChange(fieldDef, e.target.value === '' ? null : Number(e.target.value))"
                  />

                  <!-- color -->
                  <div v-else-if="fieldDef.type === 'color'" class="color-input-group">
                    <input type="color" :value="fieldDef.value"
                      @input="e => syncColor(fieldDef, e.target.value)" />
                    <input type="text" :value="fieldDef.value" maxlength="7"
                      @input="e => syncColor(fieldDef, e.target.value)" />
                  </div>
                </div>
              </template>
            </div>
          </div>
        </Transition>
      </section>
    </div>

    <!-- フッター -->
    <footer class="footer">
      <span class="status-msg" :class="{ warn: isDirty && !hasErrors, error: hasErrors }">
        {{ footerMessage }}
      </span>
      <div class="footer-btns">
        <button class="btn btn-secondary" @click="closeConfig">キャンセル</button>
        <button class="btn btn-primary" @click="saveConfig">保存</button>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'

// FileMaker スクリプト呼び出しラッパー
// WebViewer描画直後はFileMakerオブジェクトが未定義のため、利用可能になるまでリトライする
function execute(name, param = '', option = 0) {
  const MAX_WAIT_MS = 3000
  const INTERVAL_MS = 50
  let elapsed = 0

  function attempt() {
    if (typeof FileMaker !== 'undefined') {
      FileMaker.PerformScriptWithOption(name, param, String(option))
      return
    }
    elapsed += INTERVAL_MS
    if (elapsed < MAX_WAIT_MS) {
      setTimeout(attempt, INTERVAL_MS)
    } else {
      // ブラウザ開発環境（FileMakerが存在しない）
      console.log('[FM mock]', name, param, option)
    }
  }

  attempt()
}

const config = reactive({
  instanceUUID: '',
  configJSON: {
    title: '',
    targetLayouts: '',
    targetTableOccurrence: '',
    tabs: []
  }
})

const recordId = ref('')

// フィールド一覧
const recordFields = ref([])   // レコードモード用フィールド名
const portalMetaData = ref({}) // { portalName: [fieldName, ...] }
const portalNames = computed(() => Object.keys(portalMetaData.value))

// portal-select タイプのフィールドの値からポータル選択を参照
const selectedPortal = computed(() => {
  for (const tab of config.configJSON.tabs) {
    const f = tab.fields.find(f => f.type === 'portal-select')
    if (f) return f.value
  }
  return ''
})

const activeFields = computed(() => {
  if (selectedPortal.value && portalMetaData.value[selectedPortal.value]) {
    return portalMetaData.value[selectedPortal.value]
  }
  return recordFields.value
})

const activeTabId = ref('')
const isDirty = ref(false)
const errors = reactive({})

const activeTab = computed(() => config.configJSON.tabs.find(t => t.id === activeTabId.value) ?? config.configJSON.tabs[0])
const hasErrors = computed(() => Object.keys(errors).some(k => errors[k]))

const footerMessage = computed(() => {
  if (hasErrors.value) return 'バリデーションエラーがあります。必須項目を入力してください。'
  if (isDirty.value) return '保存されていない変更があります。'
  return '保存する変更はありません。'
})

// showWhen 条件を評価（同タブ内フィールドを参照）
function showField(tab, fieldDef) {
  if (!fieldDef.showWhen) return true
  const ref = tab.fields.find(f => f.key === fieldDef.showWhen.field)
  return ref?.value === fieldDef.showWhen.value
}

// toggle 変更: showWhen で参照されているフィールドが変わったら依存フィールドをリセット
function handleChange(fieldDef, value) {
  if (fieldDef.value === value) return
  fieldDef.value = value
  const triggersReset = config.configJSON.tabs.some(t =>
    t.fields.some(f => f.type === 'portal-select' && f.showWhen?.field === fieldDef.key)
  )
  if (triggersReset) {
    config.configJSON.tabs.forEach(t => {
      t.fields.forEach(f => {
        if (f.type === 'portal-select' || f.type === 'field-select') f.value = ''
      })
    })
  }
  markDirty()
}

// portal-select 変更: field-select をリセット
function handlePortalChange(fieldDef, value) {
  if (fieldDef.value === value) return
  fieldDef.value = value
  config.configJSON.tabs.forEach(t => {
    t.fields.forEach(f => {
      if (f.type === 'field-select') f.value = ''
    })
  })
  markDirty()
}

function syncColor(fieldDef, value) {
  if (/^#[0-9a-fA-F]{0,6}$/.test(value) || value === '') {
    fieldDef.value = value
    markDirty()
  }
}

function markDirty() {
  isDirty.value = true
  Object.keys(errors).forEach(k => delete errors[k])
}

// showField を通過したフィールドのみバリデーション
function validate() {
  let valid = true
  config.configJSON.tabs.forEach(tab => {
    tab.fields.forEach(field => {
      if (!showField(tab, field)) return
      if (field.required && (field.value === '' || field.value === null)) {
        errors[field.key] = true
        valid = false
      }
    })
  })
  return valid
}

function buildPayload() {
  return {
    fieldData: {
      InstanceUUID: config.instanceUUID,
      ConfigJSON: JSON.stringify(config.configJSON)
    },
    recordId: recordId.value
  }
}

function callback(action) {
  execute('mfs-Callback', JSON.stringify({ action, data: buildPayload() }), 0)
}

function saveConfig() {
  if (!validate()) return
  callback('保存')
  isDirty.value = false
}

function closeConfig() {
  callback('キャンセル')
}

// FileMaker から渡されるデータを処理する関数
// { fieldData: { InstanceUUID, ConfigJSON }, fieldMetaData, portalMetaData, recordId }
window.fm_setConfig = (json) => {
  const data = typeof json === 'string' ? JSON.parse(json) : json

  if (data.recordId) recordId.value = data.recordId

  if (data.fieldMetaData) {
    recordFields.value = data.fieldMetaData.map(f => f.name)
  }
  if (data.portalMetaData) {
    const portals = {}
    Object.entries(data.portalMetaData).forEach(([name, list]) => {
      portals[name] = list.map(f => f.name)
    })
    portalMetaData.value = portals
  }

  const fieldData = data.fieldData ?? {}
  if (fieldData.InstanceUUID) config.instanceUUID = fieldData.InstanceUUID

  const raw = fieldData.ConfigJSON
  if (raw) {
    const cfg = typeof raw === 'string' ? JSON.parse(raw) : raw
    if (cfg?.tabs) {
      config.configJSON.title = cfg.title ?? ''
      config.configJSON.targetLayouts = cfg.targetLayouts ?? ''
      config.configJSON.targetTableOccurrence = cfg.targetTableOccurrence ?? ''
      config.configJSON.tabs.splice(0, Infinity, ...cfg.tabs)
      activeTabId.value = config.configJSON.tabs[0]?.id ?? ''
      isDirty.value = false
    }
  }
}


// 開発用モックデータ（本番埋め込み時は debugMode = false にする）
const debugMode = false
const test = {
  fieldData: {
    InstanceUUID: 'XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX',
    ConfigJSON: JSON.stringify({
      title: 'アドオン設定画面',
      targetLayouts: '',
      targetTableOccurrence: '',
      tabs: [
        {
          id: 'basic', label: '基本',
          fields: [
            { key: 'mode',          value: 'record',    required: true,  type: 'toggle',       label: 'モード',             options: [{ value: 'record', label: 'レコード' }, { value: 'portal', label: 'ポータル' }] },
            { key: 'portalName',    value: '',          required: true,  type: 'portal-select',label: 'ポータル',           showWhen: { field: 'mode', value: 'portal' } },
            { key: 'uniqueField',   value: '__PK_ID',   required: true,  type: 'field-select', label: 'ユニークフィールド' },
            { key: 'sortOrderField',value: 'SortOrder', required: true,  type: 'field-select', label: '並び順フィールド' },
            { key: 'labelField',    value: 'Name',      required: true,  type: 'field-select', label: 'ラベルフィールド' },
            { key: 'showHandle',    value: true,        required: true,  type: 'switch',       label: 'ハンドル表示' },
            { key: 'recordLimit',   value: null,        required: false, type: 'number',       label: 'レコード上限', placeholder: '未入力の場合は 100 件取得' }
          ]
        },
        {
          id: 'options', label: 'オプション',
          fields: [
            { key: 'imageField',     value: 'Photo',     required: false, type: 'field-select', label: '画像フィールド' },
            { key: 'thumbnailField', value: 'Thumbnail', required: false, type: 'field-select', label: 'サムネイルフィールド' },
            { key: 'labelField2',    value: 'Email',     required: false, type: 'field-select', label: 'ラベルフィールド2' },
            { key: 'labelField3',    value: '',          required: false, type: 'field-select', label: 'ラベルフィールド3' }
          ]
        },
        {
          id: 'theme', label: 'テーマ',
          fields: [
            { key: 'colorMain',   value: '#0077cc', required: false, type: 'color', label: 'メインカラー' },
            { key: 'colorAlt',    value: '#f5f5f5', required: false, type: 'color', label: 'サブカラー' },
            { key: 'colorAccent', value: '#ff6600', required: false, type: 'color', label: 'アクセントカラー' }
          ]
        }
      ]
    })
  },
  fieldMetaData: [
    { name: '__PK_ID',   result: 'text',      type: 'normal' },
    { name: 'Name',      result: 'text',      type: 'normal' },
    { name: 'Email',     result: 'text',      type: 'normal' },
    { name: 'Category',  result: 'text',      type: 'normal' },
    { name: 'SortOrder', result: 'number',    type: 'normal' },
    { name: 'Photo',     result: 'container', type: 'normal' },
    { name: 'Thumbnail', result: 'container', type: 'normal' }
  ],
  portalData: {},
  portalMetaData: {
    'Contacts': [
      { name: 'Contacts::__PK_ID',   result: 'text',   type: 'normal' },
      { name: 'Contacts::Name',      result: 'text',   type: 'normal' },
      { name: 'Contacts::SortOrder', result: 'number', type: 'normal' },
      { name: 'Contacts::Email',     result: 'text',   type: 'normal' }
    ],
    'Orders': [
      { name: 'Orders::__PK_ID',   result: 'text',   type: 'normal' },
      { name: 'Orders::Title',     result: 'text',   type: 'normal' },
      { name: 'Orders::SortOrder', result: 'number', type: 'normal' },
      { name: 'Orders::Amount',    result: 'number', type: 'normal' }
    ]
  },
  recordId: '33',
  modId: '0'
}

onMounted(() => {
  if (debugMode) {
    window.fm_setConfig(typeof test === 'string' ? test : JSON.stringify(test))
  }
})
</script>

<style>
/* ── shadcn/ui CSS 変数 ────────────────────── */
:root {
  --background:         #ffffff;
  --foreground:         #09090b;
  --muted:              #f4f4f5;
  --muted-foreground:   #71717a;
  --border:             #e4e4e7;
  --input:              #e4e4e7;
  --primary:            #18181b;
  --primary-foreground: #fafafa;
  --secondary:          #f4f4f5;
  --secondary-foreground: #18181b;
  --accent:             #f4f4f5;
  --accent-foreground:  #18181b;
  --destructive:        #ef4444;
  --ring:               #18181b;
  --sidebar-bg:         #fafafa;
  --sidebar-border:     #e4e4e7;
  --radius:             6px;

  /* フォントサイズ 大・中・小 */
  --font-size-lg:       22px;   /* ヘッダータイトル */
  --font-size-md:       14px;   /* 標準テキスト全般 */
  --font-size-sm:       12px;   /* 予約 */

  /* フォントウェイト 極太・太い・普通 */
  --font-weight-heavy:  700;    /* ヘッダータイトル */
  --font-weight-bold:   500;    /* タブラベル */
  --font-weight-normal: 400;    /* その他全般 */
}

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  font-size: var(--font-size-md);
  background: var(--background);
  color: var(--foreground);
  height: 100vh;
  overflow: hidden;
  -webkit-font-smoothing: antialiased;
}

#app { height: 100vh; }
</style>

<style scoped>
/* ── レイアウト ─────────────────────────────── */
.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: var(--background);
}

/* ── ヘッダー ───────────────────────────────── */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: var(--background);
  padding: 0 16px;
  height: 52px;
  flex-shrink: 0;
}
.header-title {
  font-size: var(--font-size-lg);
  font-weight: 600;
  color: var(--foreground);
  letter-spacing: -0.01em;
}
.btn-close-header {
  background: transparent;
  border: none;
  color: var(--muted-foreground);
  cursor: pointer;
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--radius);
  font-size: var(--font-size-md);
  transition: background 0.15s, color 0.15s;
}
.btn-close-header:hover {
  background: var(--accent);
  color: var(--foreground);
}

/* ── メイン ─────────────────────────────────── */
.main {
  display: flex;
  flex: 1;
  overflow: hidden;
}

/* ── サイドバー ─────────────────────────────── */
.sidebar {
  width: 180px;
  background: var(--sidebar-bg);
  display: flex;
  flex-direction: column;
  padding: 12px 8px;
  gap: 1px;
  flex-shrink: 0;
}
.tab-btn {
  background: transparent;
  border: none;
  padding: 7px 10px;
  text-align: left;
  border-radius: var(--radius);
  cursor: pointer;
  font-family: inherit;
  font-size: var(--font-size-md);
  color: var(--muted-foreground);
  font-weight: 400;
  width: 100%;
  transition: background 0.15s, color 0.15s;
}
.tab-btn:hover {
  background: var(--accent);
  color: var(--accent-foreground);
}
.tab-btn.active {
  background: var(--accent);
  color: var(--accent-foreground);
  font-weight: 500;
}

/* ── コンテンツ ─────────────────────────────── */
.content {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  background: var(--background);
}
.content::-webkit-scrollbar { width: 6px; }
.content::-webkit-scrollbar-track { background: transparent; }
.content::-webkit-scrollbar-thumb {
  background: var(--border);
  border-radius: 3px;
}

/* ── タブ切替フェード ───────────────────────── */
.tab-content {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.tab-fade-enter-active { transition: opacity 0.15s ease, transform 0.15s ease; }
.tab-fade-leave-active { transition: opacity 0.1s ease; }
.tab-fade-enter-from   { opacity: 0; transform: translateY(4px); }
.tab-fade-leave-to     { opacity: 0; }

/* ── フィールドカード ───────────────────────── */
.field-card {
  background: var(--background);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  overflow: hidden;
}

/* ── フィールド行 ───────────────────────────── */
.field-row {
  display: flex;
  align-items: center;
  padding: 10px 14px;
  gap: 16px;
  border-bottom: 1px solid var(--border);
}
.field-row:last-child { border-bottom: none; }

.field-row.error .field-label { color: var(--destructive); }
.field-row.error select,
.field-row.error input[type="text"],
.field-row.error input[type="number"] {
  border-color: var(--destructive);
  box-shadow: 0 0 0 2px #ffffff, 0 0 0 4px var(--destructive);
}

.field-label {
  width: 148px;
  flex-shrink: 0;
  font-size: var(--font-size-md);
  font-weight: 500;
  color: var(--foreground);
}
.required { color: var(--destructive); margin-left: 2px; }

/* ── フォーム要素 ───────────────────────────── */
select, input[type="text"], input[type="number"] {
  flex: 1;
  max-width: 280px;
  height: 32px;
  padding: 0 10px;
  border: 1px solid var(--input);
  border-radius: var(--radius);
  font-family: inherit;
  font-size: var(--font-size-md);
  background: var(--background);
  color: var(--foreground);
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
  -webkit-appearance: none;
  appearance: none;
}
select {
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6'%3E%3Cpath d='M1 1l4 4 4-4' stroke='%2371717a' stroke-width='1.5' fill='none' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 8px center;
  padding-right: 26px;
  cursor: pointer;
}
select:focus,
input[type="text"]:focus,
input[type="number"]:focus {
  border-color: var(--ring);
  box-shadow: 0 0 0 2px #ffffff, 0 0 0 4px var(--ring);
}

/* placeholder */
input::placeholder { color: var(--muted-foreground); }

/* ── トグルボタン2択 ────────────────────────── */
.toggle-group { display: flex; gap: 4px; }
.toggle-btn {
  height: 32px;
  padding: 0 14px;
  border: 1px solid var(--input);
  border-radius: var(--radius);
  background: var(--background);
  cursor: pointer;
  font-family: inherit;
  font-size: var(--font-size-md);
  color: var(--muted-foreground);
  font-weight: 400;
  transition: background 0.15s, color 0.15s, border-color 0.15s;
}
.toggle-btn.selected {
  background: var(--primary);
  border-color: var(--primary);
  color: var(--primary-foreground);
  font-weight: 500;
}
.toggle-btn:not(.selected):hover {
  background: var(--accent);
  color: var(--accent-foreground);
}

/* ── ON/OFFスイッチ ─────────────────────────── */
.toggle-switch {
  display: flex;
  align-items: center;
  gap: 8px;
  background: none;
  border: none;
  padding: 0;
  cursor: pointer;
  font-family: inherit;
  font-size: var(--font-size-md);
  font-weight: 500;
  color: var(--muted-foreground);
}
.toggle-switch.on .toggle-label { color: var(--foreground); }
.toggle-track {
  position: relative;
  width: 40px;
  height: 22px;
  background: var(--input);
  border-radius: 11px;
  flex-shrink: 0;
  transition: background 0.2s ease;
}
.toggle-switch.on .toggle-track { background: var(--primary); }
.toggle-knob {
  position: absolute;
  top: 2px;
  left: 2px;
  width: 18px;
  height: 18px;
  background: #ffffff;
  border-radius: 50%;
  box-shadow: 0 1px 3px rgba(0,0,0,0.15), 0 1px 2px rgba(0,0,0,0.06);
  transition: transform 0.2s ease;
}
.toggle-switch.on .toggle-knob { transform: translateX(18px); }

/* ── カラーピッカー ─────────────────────────── */
.color-input-group { display: flex; align-items: center; gap: 8px; }
.color-input-group input[type="color"] {
  width: 32px;
  height: 32px;
  padding: 2px;
  border: 1px solid var(--input);
  border-radius: var(--radius);
  cursor: pointer;
  flex: none;
  background: var(--background);
  -webkit-appearance: none;
  appearance: none;
  overflow: hidden;
}
.color-input-group input[type="color"]::-webkit-color-swatch-wrapper { padding: 0; }
.color-input-group input[type="color"]::-webkit-color-swatch { border: none; border-radius: 4px; }
.color-input-group input[type="text"] {
  max-width: 96px;
  font-family: 'SF Mono', ui-monospace, 'Fira Code', monospace;
  letter-spacing: 0.05em;
}

/* ── フッター ───────────────────────────────── */
.footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 16px;
  height: 52px;
  background: var(--background);
  flex-shrink: 0;
}
.status-msg {
  font-size: var(--font-size-md);
  color: var(--muted-foreground);
}
.status-msg.warn  { color: #b45309; font-weight: 500; }
.status-msg.error { color: var(--destructive); font-weight: 500; }

.footer-btns { display: flex; gap: 6px; }
.btn {
  display: inline-flex;
  align-items: center;
  height: 32px;
  padding: 0 14px;
  border-radius: var(--radius);
  font-family: inherit;
  font-size: var(--font-size-md);
  font-weight: 500;
  cursor: pointer;
  border: 1px solid transparent;
  transition: background 0.15s, color 0.15s, border-color 0.15s, box-shadow 0.15s;
  white-space: nowrap;
}
.btn:focus-visible {
  outline: none;
  box-shadow: 0 0 0 2px #ffffff, 0 0 0 4px var(--ring);
}
.btn-secondary {
  background: var(--secondary);
  color: var(--secondary-foreground);
  border-color: var(--border);
}
.btn-secondary:hover { background: var(--border); }

.btn-primary {
  background: var(--primary);
  color: var(--primary-foreground);
}
.btn-primary:hover { background: #27272a; }
</style>
