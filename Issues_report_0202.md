# 中文輸入法與 Enter 鍵衝突問題修正報告

**日期**：2026年2月2日  
**問題編號**：IME-ENTER-001  
**狀態**：✅ 已解決

---

## 問題摘要

在使用中文輸入法（IME）時，組字確認用的 Enter 鍵會意外觸發消息發送，導致：
1. 空消息被發送到後端
2. 用戶選擇的問答模式（如「數據問答」）被重置為默認的「智能問答」
3. 用戶輸入的文本和選擇的數據源丟失

---

## 根本原因分析

### 1. 全局快捷鍵綁定錯誤

**位置**：`web/src/views/chat/index.vue`

```typescript
// ❌ 錯誤的綁定方式
const enterCommand = keys.Enter  // 綁定到所有 Enter 鍵
const enterCtrl = keys.Enter      // 綁定到所有 Enter 鍵
```

**問題**：這兩個變量都綁定到了 `keys.Enter`，導致任何 Enter 按鍵（包括中文輸入法的組字確認 Enter）都會觸發 Vue 的 deep watcher，進而調用 `handleCreateStylized()` 且不帶參數。

**正確做法**：
```typescript
// ✅ 正確的綁定方式
const enterCommand = keys['Command+Enter']  // Mac: Cmd+Enter
const enterCtrl = keys['Ctrl+Enter']        // Windows: Ctrl+Enter
```

### 2. 缺少輸入法組字狀態追蹤

**位置**：`web/src/views/chat/default-page.vue`

原始代碼僅依賴 `KeyboardEvent.isComposing` 屬性，但在某些瀏覽器/輸入法組合中，`compositionend` 事件和 `keydown` 事件的觸發順序不一致，導致：
- `isComposing` 在 `keydown` 之前被設為 false
- 組字確認的 Enter 被誤認為正常 Enter
- 消息被意外發送

---

## 解決方案

### 1. 修正快捷鍵綁定

**文件**：`web/src/views/chat/index.vue`

```typescript
// 修改前
const keys = useMagicKeys()
const enterCommand = keys.Enter
const enterCtrl = keys.Enter

// 修改後
const keys = useMagicKeys()
const enterCommand = keys['Command+Enter']
const enterCtrl = keys['Ctrl+Enter']
```

### 2. 實現三層輸入法防護機制

**文件**：`web/src/views/chat/default-page.vue`

#### 第一層：狀態追蹤
```typescript
const isComposingInput = ref(false)
const lastCompositionEndTime = ref(0)

const handleCompositionStart = () => {
  isComposingInput.value = true
}

const handleCompositionEnd = () => {
  lastCompositionEndTime.value = Date.now()
  
  // 延遲 150ms 清除組字狀態，確保時序正確
  setTimeout(() => {
    isComposingInput.value = false
  }, 150)
}
```

#### 第二層：多重檢查
```typescript
const handleEnter = (e?: KeyboardEvent) => {
  // 檢查 1：輸入法組字中
  if (e?.isComposing || isComposingInput.value) {
    return
  }

  // 檢查 2：輸入法剛完成（200ms 內）
  const timeSinceCompositionEnd = Date.now() - lastCompositionEndTime.value
  if (timeSinceCompositionEnd < 200 && lastCompositionEndTime.value > 0) {
    return
  }

  // 檢查 3：Shift+Enter 換行
  if (e && e.shiftKey) {
    return
  }
  
  // ... 其他檢查
}
```

#### 第三層：事件冒泡阻止
```vue
<n-input
  @keydown.enter="(e) => {
    e.preventDefault();
    e.stopPropagation();  // 阻止事件冒泡
    handleEnter(e);
  }"
  @compositionstart="handleCompositionStart"
  @compositionend="handleCompositionEnd"
/>
```

### 3. 增加提交鎖機制

防止重複提交：
```typescript
const isSubmitting = ref(false)
const lastSubmitTime = ref(0)

const handleEnter = (e?: KeyboardEvent) => {
  // 防止 500ms 內重複提交
  if (isSubmitting.value || (Date.now() - lastSubmitTime.value < 500)) {
    return
  }
  
  // ... 執行發送
  isSubmitting.value = true
  lastSubmitTime.value = Date.now()
  
  emit('submit', { ... })
  
  // 500ms 後解除鎖定
  setTimeout(() => {
    isSubmitting.value = false
  }, 500)
}
```

### 4. 參數鎖定機制

確保發送時的 mode 和 datasource 不被改變：
```typescript
// 在 emit 前鎖定參數
const modeToSend = selectedMode.value?.value || 'COMMON_QA'
const datasourceIdToSend = selectedDatasource.value?.id

emit('submit', {
  text: inputValue.value,
  mode: modeToSend,
  datasource_id: datasourceIdToSend,
})
```

### 5. 父組件防護加強

**文件**：`web/src/views/chat/index.vue`

```typescript
const handleSubmitFromDefaultPage = (payload) => {
  // 檢查 1：payload 存在
  if (!payload) return
  
  // 檢查 2：必須有文本或文件
  if (!payload.text?.trim() && !businessStore.file_list.length) return
  
  // 檢查 3：mode 白名單驗證
  const validModes = ['COMMON_QA', 'DATABASE_QA', 'FILEDATA_QA', 'REPORT_QA']
  if (!payload.mode || !validModes.includes(payload.mode)) return
  
  // 檢查 4：DATABASE_QA 和 REPORT_QA 必須有 datasource_id
  if ((payload.mode === 'DATABASE_QA' || payload.mode === 'REPORT_QA') 
      && !payload.datasource_id) {
    window.$ModalMessage?.error?.('數據問答需要選擇數據源')
    return
  }
  
  // ... 繼續處理
}
```

### 6. 全局 Watcher 加入 showDefaultPage 檢查

```typescript
watch(() => enterCommand.value, () => {
  if (!isMacos || notUsingInput.value) return
  if (stylizingLoading.value) return
  
  // 🔒 如果顯示 default page，不處理
  if (showDefaultPage.value) return
  
  if (!enterCommand.value) {
    handleCreateStylized()
  }
}, { deep: true })
```

---

## 修改文件清單

1. **`web/src/views/chat/default-page.vue`**
   - 新增輸入法狀態追蹤（`isComposingInput`, `lastCompositionEndTime`）
   - 新增 `handleCompositionStart` 和 `handleCompositionEnd` 函數
   - 強化 `handleEnter` 函數的檢查邏輯
   - 新增提交鎖機制
   - 新增參數鎖定機制
   - 修改輸入框事件綁定（加入 `@compositionstart` 和 `@compositionend`）

2. **`web/src/views/chat/index.vue`**
   - 修正快捷鍵綁定（`enterCommand` 和 `enterCtrl`）
   - 強化 `handleSubmitFromDefaultPage` 的參數驗證
   - 為兩個 watcher 加入 `showDefaultPage` 檢查
   - 為主聊天頁面的輸入框和發送按鈕加入 `showDefaultPage` 檢查

---

## 測試結果

### 測試場景 1：中文輸入法組字

**操作步驟**：
1. 選擇「數據問答」模式
2. 選擇數據庫「my_project」
3. 使用中文輸入法輸入「查詢所有表」
4. 按第一次 Enter（組字確認）

**預期結果**：✅ 不發送消息，等待用戶第二次 Enter

**實際結果**：✅ 通過

---

### 測試場景 2：正常發送

**操作步驟**：
1. 選擇「數據問答」模式
2. 選擇數據庫「my_project」
3. 輸入「查詢所有表」（組字完成）
4. 按第二次 Enter（發送）

**預期結果**：
- ✅ 消息發送成功
- ✅ mode 為 `DATABASE_QA`
- ✅ datasource_id 為 `1`
- ✅ 文本為「查詢所有表」

**實際結果**：✅ 通過

---

### 測試場景 3：空值阻止

**操作步驟**：
1. 不輸入任何內容
2. 按 Enter

**預期結果**：✅ 不發送消息

**實際結果**：✅ 通過

---

### 測試場景 4：快捷鍵

**操作步驟**：
1. 在主聊天頁面輸入文本
2. 按 Ctrl+Enter（Windows）或 Cmd+Enter（Mac）

**預期結果**：✅ 快速發送消息

**實際結果**：✅ 通過

---

## 防護機制總結

| 層級 | 檢查項目 | 位置 | 作用 |
|-----|---------|------|------|
| **0** | 快捷鍵綁定修正 | index.vue | 防止普通 Enter 觸發全局 watcher |
| **1** | `isComposing` 檢查 | default-page.vue | 標準瀏覽器支持 |
| **2** | `isComposingInput` 追蹤 | default-page.vue | 更可靠的狀態追蹤 |
| **3** | 時間差檢查（200ms） | default-page.vue | 防止時序競爭問題 |
| **4** | 空值檢查 | default-page.vue | 防止空消息發送 |
| **5** | 提交鎖（500ms） | default-page.vue | 防止重複提交 |
| **6** | 參數鎖定 | default-page.vue | 確保 mode 不被改變 |
| **7** | 父組件驗證 | index.vue | Mode 白名單、必填檢查 |
| **8** | showDefaultPage 檢查 | index.vue | 防止頁面切換時誤觸發 |

---

## 相關資源

### Web 標準
- [Composition Events](https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent)
- [KeyboardEvent.isComposing](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/isComposing)

### Vue 相關
- [Event Modifiers](https://vuejs.org/guide/essentials/event-handling.html#event-modifiers)
- [VueUse - useMagicKeys](https://vueuse.org/core/useMagicKeys/)

---

## 後續建議

1. **單元測試**：為 `handleEnter` 和 `handleSubmitFromDefaultPage` 添加單元測試
2. **E2E 測試**：添加針對中文輸入法的端到端測試
3. **監控**：在生產環境添加錯誤監控，追蹤空消息發送的頻率
4. **文檔**：更新開發文檔，記錄中文輸入法處理的最佳實踐

---

## 結論

通過修正快捷鍵綁定和實現多層次的輸入法防護機制，成功解決了中文輸入法與 Enter 鍵的衝突問題。現在系統能夠正確區分：
- 輸入法組字確認的 Enter（不發送）
- 正常的 Enter（發送消息）
- 快捷鍵 Cmd/Ctrl+Enter（快速發送）

所有測試場景均已通過，用戶選擇的問答模式和數據源在發送過程中保持不變。
