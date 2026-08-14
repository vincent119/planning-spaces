ˋ# Frontend Design

## 技術選型

- **框架**：React 19.2+，TypeScript
- **路由**：React Router v6
- **Server State**：TanStack Query（API 快取、自動刷新）
- **Client State**：React Context（當前使用者、當前專案、語系、主題）
- **表單**：React Hook Form + Zod 驗證
- **報表**：ECharts（`echarts-for-react`）
- **UI 元件**：Material UI v9（`@mui/material`）+ MUI X Data Grid Community（`@mui/x-data-grid`，MIT）
- **國際化**：i18next + react-i18next（支援繁中 / 簡中 / 英文）
- **打包**：Vite

版本規範：`react`、`react-dom`、`@types/react`、`@types/react-dom` 需使用 19.x 相容版本；新增前端套件前需確認 peer dependencies 支援 React 19。

---

## 頁面結構

```text
/login                          登入頁（帳密 + OIDC 入口）
/dashboard                      全域儀表板（跨專案彙總）
/projects                       專案列表
/projects/:id                   專案儀表板
/projects/:id/tickets           Ticket 列表（篩選 / 搜尋）
/projects/:id/tickets/new       建立 Ticket
/tickets/:id                    Ticket 詳情
/projects/:id/reports           報表中心（範本列表 + 設計器）
/projects/:id/jira              Jira 匯入與統計報表
/admin/users                    用戶管理（Admin）
/admin/roles                    角色管理（Admin）
/admin/menus                    選單管理（Admin）
/admin/groups                   群組管理（權限群組 / 成員 / 選單權限矩陣，Admin）
/admin/projects                 專案管理（Admin）
/admin/logs                     日誌查詢（Admin）
/admin/schedulers               排程管理（Admin）
/admin/system                   全域設定（Admin）
/settings/profile               個人設定
/settings/mfa                   MFA 裝置管理
/auth/mfa/setup                 強制 MFA 裝置新增頁
```

Ticket 列表操作欄提供複製、查看與編輯，順序為 `ContentCopyIcon`、`VisibilityIcon`、`EditIcon`。查看導向 `/tickets/:id`；編輯導向 `/tickets/:id?edit=1`，由 Ticket 詳情頁載入資料後自動開啟既有編輯 Dialog。

Ticket 列表採 MUI Data Grid 受控伺服器分頁，前端頁碼從 `0` 起算，呼叫 API 時轉為從 `1` 起算。分頁 query key 變更並載入下一頁期間，TanStack Query 必須使用 `keepPreviousData` 保留上一筆成功資料，使 `rows` 與 `rowCount` 不會暫時清空；篩選條件實際變更時才由既有 handler 將頁碼重設為第一頁。

Ticket 列表欄位順序需在「班別」與「指派人員」之間加入「開單人員」。資料沿用列表 API 的 `creator_full_name`、`creator_username` 與 `created_by`，顯示優先順序為全名、帳號、建立者 ID；長文字以省略顯示並提供 tooltip，不新增後端 API 呼叫。

Ticket 列表搜尋區需新增「開單人員」下拉篩選。選項使用 Ticket metadata API 的 `creators`，內容為目前專案未刪除 Ticket 實際出現過的 `created_by` 去重清單，不得沿用指派人員使用的專案成員查詢；值使用 `id`。前端以 `creator_id` query parameter 呼叫 Ticket 列表 API；條件需納入 TanStack Query key、有效篩選判斷及分頁重設流程，後端以 `tickets.created_by` 套用參數化查詢。

Ticket 列表欄位順序需在「指派人員」與「更新時間」之間加入「外部單號」。資料沿用列表 API 既有的 `external_ref`，不新增後端 API 呼叫或資料庫欄位。欄位標題使用 Ticket 語系的「外部單號」；空字串、`null` 或 `undefined` 使用列表既有空值顯示函式，非空值以單行省略顯示並提供完整文字 tooltip。此欄位只負責顯示，不在本次範圍新增搜尋、排序或複製列內容。

調整後相關欄位順序：

```text
班別 → 開單人員 → 指派人員 → 外部單號 → 更新時間 → 操作
```

複製列資訊使用 `navigator.clipboard.writeText`，只複製該列溝通用欄位，不包含 Ticket ID、附件 / 貼圖、內部 storage key 或帶 ID 的詳情連結。建議格式：

```text
標題：{title}
狀態：{status_label}
優先級：{priority}
事件類型：{ticket_type_name}
子專案：{sub_project_name}
資訊來源：{ticket_resource_name}
班別：{creator_duty_shift_name}
指派人員：{assignee_name}
更新時間：{updated_at}
```

複製成功顯示 `ticket.copy_success` toast；失敗顯示 `ticket.copy_failed` toast。

---

## 元件架構

```text
src/
├── components/           # 共用元件
│   ├── TicketCard/
│   ├── StatusBadge/
│   ├── PriorityBadge/
│   ├── SLACountdown/     # SLA 倒數計時
│   ├── MarkdownEditor/   # 留言 Markdown 編輯器
│   └── AttachmentUpload/ # 圖片上傳（拖曳 / 貼上截圖）
├── features/
│   ├── auth/             # 登入、MFA、OIDC
│   ├── ticket/           # Ticket CRUD、狀態流轉、留言
│   ├── project/          # 專案 / 子專案管理
│   ├── report/           # 報表設計器、範本執行
│   ├── jira/             # Jira CSV 匯入、統計報表
│   ├── dashboard/        # 儀表板
│   └── admin/            # 使用者、系統設定、排程
├── hooks/                # 全域共用 hooks
├── locales/              # i18n 翻譯檔
│   ├── zh-TW/
│   ├── zh-CN/
│   └── en/
├── theme/                # MUI 主題定義
│   └── index.ts          # lightTheme / darkTheme / glassEffect
├── utils/
├── types/
├── i18n.ts               # i18next 初始化
└── App.tsx
```

---

## 狀態管理

### Server State（TanStack Query）

```typescript
// 範例：Ticket 列表查詢
const { data, isLoading } = useQuery({
  queryKey: ['tickets', projectId, filters],
  queryFn: () => ticketApi.list({ projectId, ...filters }),
  staleTime: 30_000,  // 儀表板資料 30 秒自動刷新
});
```

### Client State（React Context）

```typescript
interface AppContext {
  currentUser: User | null;
  currentProject: Project | null;
  setCurrentProject: (p: Project) => void;
  locale: 'zh-TW' | 'zh-CN' | 'en';
  setLocale: (l: 'zh-TW' | 'zh-CN' | 'en') => void;
  themeMode: 'light' | 'light-glass' | 'dark' | 'dark-glass';
  setThemeMode: (m: 'light' | 'light-glass' | 'dark' | 'dark-glass') => void;
}
```

---

## Material UI / MUI X 使用規範

套件：`@mui/material`、`@mui/icons-material`、`@mui/x-data-grid`、`@emotion/react`、`@emotion/styled`。

- 所有 UI 元件優先使用 MUI 原生元件，不自行實作已有的功能（Dialog、Snackbar 等）
- Ticket 列表與管理後台表格優先使用 MUI X Data Grid Community（`@mui/x-data-grid`，MIT）；除非明確需要 Pro / Premium 功能，否則不引入 `@mui/x-data-grid-pro` 或 `@mui/x-data-grid-premium`
- 若需要多欄排序、多條件篩選、欄位 pinning、row grouping、Excel export 等進階功能，需先確認 MUI X Pro / Premium 商業授權
- 所有 Table、Data Grid 與資料列表必須接入 i18n；MUI X Data Grid 必須透過共用 helper 產生 `localeText`，覆蓋欄位選單、排序、篩選、欄位管理、分頁、空資料、載入與錯誤等內建文字，避免顯示英文預設文案
- 客製化樣式透過 `sx` prop 或 `styled()`，禁止直接寫 inline `style`
- 顏色、間距、字型一律引用 `theme.palette` / `theme.spacing` / `theme.typography`，不寫魔數
- Icon 統一使用 `@mui/icons-material`，命名與語意一致
- MUI v9 使用新版 `Grid` API：不使用 `item`、`xs`、`sm`、`md`、`lg`、`xl` props，改用 `size`

### 全域 Toast 提示規範

前端所有使用者觸發的新增、修改、刪除、批量操作、立即執行、登入 / 登出、語系 / 主題切換、MFA 操作與業務狀態變更，成功與失敗都必須顯示 Toast。Toast 以 MUI `Snackbar` + `Alert` 實作，統一由共用通知元件或 hook 管理，不在各頁面重複手寫樣式。

- 成功：`severity="success"`，文案描述已完成的動作，例如「密碼已更新」、「使用者已新增」
- 失敗：`severity="error"`，優先顯示後端 `message`；若後端未提供訊息，依錯誤碼使用 i18n 預設文案
- 警告：`severity="warning"`，用於部分成功、不可逆操作前置限制或非阻斷式問題
- 位置：桌機預設右上角，手機預設上方置中；不得遮擋表單、Dialog 主要按鈕或正在輸入的欄位
- 行為：自動關閉並允許手動關閉；連續操作需排隊或覆蓋為最新訊息，但不得靜默吞掉失敗訊息
- 文案：所有 Toast 文案都需放入 i18n namespace，不得在元件內硬編碼長句

---

## 國際化（i18n）

套件：`i18next`、`react-i18next`、`i18next-browser-languagedetector`。

### 支援語系

| 語系 | locale key | 說明 |
| ------ | ----------- | ------ |
| 繁體中文 | `zh-TW` | 預設語系 |
| 簡體中文 | `zh-CN` | |
| 英文 | `en` | |

### 目錄結構

```text
src/
└── locales/
    ├── zh-TW/
    │   ├── common.json      # 通用（按鈕、狀態、優先級等）
    │   ├── ticket.json      # Ticket 相關文字
    │   ├── project.json
    │   ├── report.json
    │   ├── auth.json
    │   └── admin.json
    ├── zh-CN/
    │   └── ...              # 同結構
    └── en/
        └── ...              # 同結構
```

### 初始化

```typescript
// src/i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'zh-TW',
    supportedLngs: ['zh-TW', 'zh-CN', 'en'],
    defaultNS: 'common',
    detection: {
      order: ['localStorage', 'navigator'],
      caches: ['localStorage'],
    },
    interpolation: { escapeValue: false },
  });

export default i18n;
```

### 使用方式

```typescript
// 元件內
const { t } = useTranslation('ticket');
<span>{t('status.open')}</span>

// 切換語系（存入 localStorage 自動持久化）
const { i18n } = useTranslation();
i18n.changeLanguage('en');
```

### 翻譯 key 命名規則

- 使用 `namespace.category.key` 三層結構
- 狀態類：`status.open` / `status.in_progress` / `status.resolved`
- 優先級：`priority.p1` / `priority.p2`
- 動作類：`action.create` / `action.edit` / `action.delete`
- 錯誤類：`error.required` / `error.max_length`

### Table / Grid 內建文案

所有 table、grid 與 MUI X Data Grid 的顯示文字都必須使用目前語系，不只包含資料欄位，也包含元件內建 UI。

- Data Grid 需建立共用 `getDataGridLocaleText(t)` 或等價 helper，集中維護 `localeText`
- 覆蓋範圍需包含欄位選單（排序、篩選、隱藏欄位、管理欄位）、分頁（Rows per page、目前範圍）、空資料、載入、錯誤、欄位管理與工具列文字
- 各頁面不得只翻譯欄位標題而忽略 MUI X 內建選單文字
- 語系切換時，`localeText` 應隨 `useTranslation()` 重新計算並即時反映在已掛載的 Data Grid 上
- 若使用 MUI `Table` 自行組合分頁或選單，分頁與選單文字也必須使用同一組 i18n key

---

## 主題設計

### 淺色 / 深色模式

使用 MUI `createTheme` 建立四套 theme，透過 `ThemeProvider` 動態切換。偏好設定存入 `localStorage`，初始值跟隨系統（`prefers-color-scheme`，預設對應 `dark-glass`）。

| theme key | 名稱 | 說明 |
| ----------- | ------ | ------ |
| `light` | 淺色 | 純白背景，無毛玻璃 |
| `light-glass` | 淺色毛玻璃 | 淺色漸層背景 + 毛玻璃元件 |
| `dark` | 深色 | 純深色背景，無毛玻璃 |
| `dark-glass` | 暗色毛玻璃 | 深色漸層背景 + 毛玻璃元件（預設） |

```typescript
// src/theme/index.ts
import { createTheme } from '@mui/material/styles';

const glassEffect = (opacity = 0.7) => ({
  backdropFilter: 'blur(12px) saturate(180%)',
  WebkitBackdropFilter: 'blur(12px) saturate(180%)',
  backgroundColor: `rgba(255, 255, 255, ${opacity})`,
  border: '1px solid rgba(255, 255, 255, 0.3)',
});

const glassDarkEffect = (opacity = 0.4) => ({
  backdropFilter: 'blur(12px) saturate(180%)',
  WebkitBackdropFilter: 'blur(12px) saturate(180%)',
  backgroundColor: `rgba(18, 18, 18, ${opacity})`,
  border: '1px solid rgba(255, 255, 255, 0.08)',
});

// 淺色（無毛玻璃）
export const lightTheme = createTheme({
  palette: { mode: 'light', primary: { main: '#1976d2' } },
});

// 淺色毛玻璃
export const lightGlassTheme = createTheme({
  palette: {
    mode: 'light',
    primary: { main: '#1976d2' },
    background: { default: 'linear-gradient(135deg, #e3f0ff 0%, #f5e6ff 100%)', paper: 'rgba(255,255,255,0.7)' },
  },
  components: {
    MuiPaper:  { styleOverrides: { root: { ...glassEffect(),      borderRadius: 12 } } },
    MuiCard:   { styleOverrides: { root: { ...glassEffect(),      borderRadius: 12 } } },
    MuiAppBar: { styleOverrides: { root: { ...glassEffect(0.8),   boxShadow: 'none' } } },
    MuiDrawer: { styleOverrides: { paper: { ...glassEffect(0.85) } } },
    MuiDialog: { styleOverrides: { paper: { ...glassEffect(0.9),  borderRadius: 16 } } },
  },
});

// 深色（無毛玻璃）
export const darkTheme = createTheme({
  palette: { mode: 'dark', primary: { main: '#90caf9' } },
});

// 暗色毛玻璃（預設）
export const darkGlassTheme = createTheme({
  palette: {
    mode: 'dark',
    primary: { main: '#90caf9' },
    background: { default: 'linear-gradient(135deg, #0a0a1a 0%, #1a0a2e 100%)', paper: 'rgba(18,18,18,0.4)' },
  },
  components: {
    MuiPaper:  { styleOverrides: { root: { ...glassDarkEffect(),      borderRadius: 12 } } },
    MuiCard:   { styleOverrides: { root: { ...glassDarkEffect(),      borderRadius: 12 } } },
    MuiAppBar: { styleOverrides: { root: { ...glassDarkEffect(0.6),   boxShadow: 'none' } } },
    MuiDrawer: { styleOverrides: { paper: { ...glassDarkEffect(0.7) } } },
    MuiDialog: { styleOverrides: { paper: { ...glassDarkEffect(0.8),  borderRadius: 16 } } },
  },
});

export const themeMap = {
  'light':       lightTheme,
  'light-glass': lightGlassTheme,
  'dark':        darkTheme,
  'dark-glass':  darkGlassTheme,
} as const;

export type ThemeMode = keyof typeof themeMap;
```

### 毛玻璃效果規範

| 元件 | 淺色 opacity | 深色 opacity | blur |
| ------ | ------------- | ------------- | ------ |
| AppBar / Drawer | 0.80 / 0.85 | 0.60 / 0.70 | 12px |
| Card / Paper | 0.70 | 0.40 | 12px |
| Dialog | 0.90 | 0.80 | 12px |

- 背景需有漸層色或圖片才能讓毛玻璃效果顯現，`body` 設定漸層背景
- `backdrop-filter` 在 Safari 需加 `-webkit-` 前綴
- 低效能裝置（`prefers-reduced-motion: reduce`）降級為純色背景，移除 blur

### ThemeProvider 整合

```typescript
// src/App.tsx
import { ThemeProvider, CssBaseline } from '@mui/material';
import { themeMap } from './theme';

export function App() {
  const { themeMode } = useAppContext();

  return (
    <ThemeProvider theme={themeMap[themeMode]}>
      <CssBaseline />
      <RouterProvider router={router} />
    </ThemeProvider>
  );
}
```

### 切換按鈕

主題與語系切換整合在 AppBar 右側 Avatar 下拉選單的「設定」子面板中（見下方 AppBar 設計），不在 Navbar 單獨放按鈕。

---

## 主頁面佈局設計

### 整體結構

```text
┌──────────────────────────────────────────────────────────────┐
│ AppBar（毛玻璃）                                              │
│ [≡] Logo / 專案名稱              🔔³  [AD] 系統管理員  ▼    │
├────────────┬─────────────────────────────────────────────────┤
│            │                                                  │
│  Sidebar   │  主內容區（Content）                             │
│  （收合式）│                                                  │
│            │                                                  │
│  ⚙ 系統管理 │                                                  │
│  📋 Tickets │                                                  │
│  📊 報表    │                                                  │
│  ...       │                                                  │
│            │                                                  │
│  [←] 收合  │                                                  │
└────────────┴─────────────────────────────────────────────────┘
```

### AppBar

- 毛玻璃效果（同主題設計規範）
- 左側：Sidebar 收合 `IconButton`（`MenuIcon` / `MenuOpenIcon`）+ 系統 Logo + 當前專案名稱
- 右側：
  - 通知鈴鐺 `IconButton`（`NotificationsIcon`）+ 未讀數 `Badge`（紅色）
  - 使用者 `Avatar`（顯示姓名首字母，青藍色背景）+ 姓名文字
  - 點擊 Avatar 區塊展開 `Menu`（深色毛玻璃，圓角 `12px`）：

```typescript
// Avatar 下拉選單項目
const userMenuItems = [
  {
    key: 'profile',
    icon: <PersonIcon />,
    label: t('nav.profile'),   // 個人資料
    path: '/settings/profile',
    color: 'inherit',
  },
  {
    key: 'settings',
    icon: <SettingsIcon />,
    label: t('nav.settings'),  // 設定（展開子面板，不跳頁）
    expandable: true,
    color: 'inherit',
  },
  { divider: true },
  {
    key: 'logout',
    icon: <LogoutIcon />,
    label: t('nav.logout'),    // 登出
    action: handleLogout,
    color: 'error.main',       // 紅色
  },
];
```

「設定」項目點擊後，在同一 `Menu` 內展開子面板（不關閉 Menu，不跳頁），子面板包含：

#### **主題選擇**

```typescript
// PaletteIcon + 「主題」文字 + 當前主題名稱 + 展開箭頭
// 展開後顯示 4 個選項，當前選中項青藍色背景 highlight
const themeOptions: { value: ThemeMode; label: string }[] = [
  { value: 'light',       label: t('theme.light') },        // 淺色
  { value: 'light-glass', label: t('theme.light_glass') },  // 淺色毛玻璃
  { value: 'dark',        label: t('theme.dark') },         // 深色
  { value: 'dark-glass',  label: t('theme.dark_glass') },   // 暗色毛玻璃
];

// 選中項樣式
const selectedItemSx = {
  backgroundColor: 'rgba(33, 150, 243, 0.25)',
  borderRadius: 1,
  '&:hover': { backgroundColor: 'rgba(33, 150, 243, 0.35)' },
};
```

#### **語系選擇**

```typescript
// LanguageIcon + 當前語系名稱（不顯示「語系」文字標籤，直接顯示當前值）
// 展開後顯示 3 個選項，當前選中項同樣 highlight
const localeOptions = [
  { value: 'zh-TW', label: '繁體中文' },
  { value: 'zh-CN', label: '简体中文' },
  { value: 'en',    label: 'English' },
];
```

兩個選項展開時使用巢狀 `Popover` 或 inline 展開列表（`Collapse`），選擇後立即套用並存入 `localStorage`，不需要儲存按鈕。

```typescript
// Menu 樣式（深色毛玻璃）
const menuPaperSx = {
  mt: 1.5,
  minWidth: 160,
  borderRadius: 2,
  backdropFilter: 'blur(16px) saturate(180%)',
  WebkitBackdropFilter: 'blur(16px) saturate(180%)',
  backgroundColor: 'rgba(28, 28, 35, 0.92)',
  border: '1px solid rgba(255,255,255,0.08)',
  boxShadow: '0 8px 32px rgba(0,0,0,0.4)',
  '& .MuiMenuItem-root': {
    gap: 1.5,
    py: 1.2,
    px: 2,
    borderRadius: 1,
    mx: 0.5,
    '&:hover': { backgroundColor: 'rgba(255,255,255,0.08)' },
  },
};
```

> 登出選項文字與 icon 使用 `color: 'error.main'`（紅色），與其他選項做視覺區隔。`Divider` 分隔登出與上方選項。

### Sidebar

- 寬度：展開 `240px`，收合 `64px`，動畫 `transition: width 0.2s ease`
- 收合時只顯示 icon，展開時顯示 icon + 文字
- 底部放收合切換箭頭按鈕（`ChevronLeftIcon` / `ChevronRightIcon`）
- 選單項目依角色顯示（Admin 才看到系統管理群組）

#### **群組展開行為**

- 「系統管理」為可展開群組，點擊標題列切換展開 / 收合（`Collapse`）
- 群組標題右側顯示 `ExpandLessIcon`（展開中）/ `ExpandMoreIcon`（已收合）
- 群組標題本身有青藍色半透明背景（`rgba(33,150,243,0.15)`），與一般選單項目區隔
- 子選單項目縮排 `pl: 4`（收合狀態下 Sidebar 整體收合時，子項目 tooltip 顯示名稱）
- 當前選中的子項目顯示青藍色背景 highlight（`rgba(33,150,243,0.25)`）

```typescript
// 群組標題樣式
const groupHeaderSx = (isExpanded: boolean) => ({
  backgroundColor: isExpanded
    ? 'rgba(33, 150, 243, 0.15)'
    : 'transparent',
  borderRadius: 2,
  mx: 1,
  mb: 0.5,
  '&:hover': { backgroundColor: 'rgba(33, 150, 243, 0.2)' },
});

// 子選單選中項樣式
const activeItemSx = {
  backgroundColor: 'rgba(33, 150, 243, 0.25)',
  borderRadius: 2,
  mx: 1,
  '&:hover': { backgroundColor: 'rgba(33, 150, 243, 0.35)' },
};
```

```typescript
const menuItems = [
  { key: 'dashboard',  icon: <DashboardIcon />,           label: t('nav.dashboard'),  path: '/dashboard' },
  { key: 'tickets',    icon: <ConfirmationNumberIcon />,   label: t('nav.tickets'),    path: '/projects/:id/tickets' },
  { key: 'reports',    icon: <BarChartIcon />,             label: t('nav.reports'),    path: '/projects/:id/reports' },
  { key: 'jira',       icon: <ImportExportIcon />,         label: t('nav.jira'),       path: '/projects/:id/jira' },
  // Admin only — 展開群組
  {
    key: 'admin',
    icon: <SettingsIcon />,
    label: t('nav.admin'),   // 系統管理
    adminOnly: true,
    expandable: true,
    children: [
      { key: 'admin-users',      icon: <PersonIcon />,         label: t('nav.admin_users'),      path: '/admin/users' },
      { key: 'admin-roles',      icon: <GroupsIcon />,         label: t('nav.admin_roles'),      path: '/admin/roles' },
      { key: 'admin-menus',      icon: <TableChartIcon />,     label: t('nav.admin_menus'),      path: '/admin/menus' },
      { key: 'admin-projects',   icon: <AccountTreeIcon />,    label: t('nav.admin_projects'),   path: '/admin/projects' },
      { key: 'admin-logs',       icon: <ArticleIcon />,        label: t('nav.admin_logs'),       path: '/admin/logs' },
      { key: 'admin-schedulers', icon: <ScheduleIcon />,       label: t('nav.admin_schedulers'), path: '/admin/schedulers' },
      { key: 'admin-global',     icon: <TuneIcon />,           label: t('nav.admin_global'),     path: '/admin/system' },
    ],
  },
];
```

---

## 儀表板統計卡片設計

### 視覺規格

每張卡片採漸層背景色（非毛玻璃），圓角 `16px`，無邊框，投影：

```text
┌─────────────────────────────┐
│  標題文字            [icon] │  ← 右上角圓形半透明 icon 背景
│                             │
│  1,254                      │  ← 大數字，fontSize 2.5rem，fontWeight 700
│                             │
│  📈 +12%    🕐 今日         │  ← 趨勢 + 時間標籤，底部左右排列
└─────────────────────────────┘
```

### OnCall Ticket System 卡片定義

| 卡片 | 漸層色 | Icon | 數值來源 |
| ------ | -------- | ------ | --------- |
| Open Tickets | `#42a5f5 → #1565c0`（藍） | `ConfirmationNumberIcon` | `tickets_open_total` |
| 今日新建 | `#ce93d8 → #7b1fa2`（紫） | `AddCircleIcon` | `increase(tickets_created_total[24h])` |
| 待處理（P1/P2） | `#ffa726 → #e65100`（橘） | `PriorityHighIcon` | open tickets where priority IN (P1,P2) |
| SLA 違反 | `#66bb6a → #2e7d32`（綠） | `WarningAmberIcon` | `increase(sla_breaches_total[$range])` |

> 趨勢百分比：與昨日同時段相比，正值綠色 `TrendingUpIcon`，負值紅色 `TrendingDownIcon`。

### 卡片元件實作

```typescript
interface StatCardProps {
  title: string;
  value: number | string;
  icon: React.ReactNode;
  gradient: [string, string];   // [from, to]
  trend?: number;               // 百分比，正負值
  trendLabel?: string;          // 例：'今日'
}

const cardSx = (gradient: [string, string]) => ({
  background: `linear-gradient(135deg, ${gradient[0]} 0%, ${gradient[1]} 100%)`,
  borderRadius: 4,
  p: 3,
  color: '#fff',
  position: 'relative',
  overflow: 'hidden',
  boxShadow: '0 4px 20px rgba(0,0,0,0.15)',
  // 右上角裝飾圓（大）
  '&::after': {
    content: '""',
    position: 'absolute',
    top: -20,
    right: -20,
    width: 120,
    height: 120,
    borderRadius: '50%',
    background: 'rgba(255,255,255,0.1)',
  },
});

const iconWrapperSx = {
  position: 'absolute',
  top: 16,
  right: 16,
  width: 48,
  height: 48,
  borderRadius: '50%',
  background: 'rgba(255,255,255,0.25)',
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
  zIndex: 1,
};
```

### 儀表板頁面佈局

```typescript
// /dashboard 或 /projects/:id/dashboard
<Grid container spacing={3}>
  {/* 統計卡片列 */}
  <Grid size={{ xs: 12, sm: 6, lg: 3 }}><StatCard ... /></Grid>
  <Grid size={{ xs: 12, sm: 6, lg: 3 }}><StatCard ... /></Grid>
  <Grid size={{ xs: 12, sm: 6, lg: 3 }}><StatCard ... /></Grid>
  <Grid size={{ xs: 12, sm: 6, lg: 3 }}><StatCard ... /></Grid>

  {/* 圖表列 */}
  <Grid size={{ xs: 12, md: 8 }}>
    {/* Ticket 趨勢折線圖（ECharts） */}
  </Grid>
  <Grid size={{ xs: 12, md: 4 }}>
    {/* 優先級分佈 Donut Chart */}
  </Grid>

  {/* 最近 Tickets 列表 */}
  <Grid size={12}>
    {/* MUI X Data Grid Community 或 Table，顯示最近 10 筆 */}
  </Grid>
</Grid>
```

每 30 秒自動刷新（`refetchInterval: 30_000`）。

### 儀表板自適應與版面編輯

現有全域儀表板與專案儀表板已依 `xs`、`sm`、`md`、`lg` 斷點自動換欄。後續版面編輯功能延續響應式佈局，不對整頁套用 CSS `scale`，也不縮小基礎字級來容納內容。

#### 範圍與假設

- 套用頁面：`/dashboard` 與 `/projects/:id`
- 可移動面板：統計卡片、Ticket 趨勢圖、優先級分佈圖、最近 Ticket 列表
- 桌面版提供明確的「編輯版面」模式；一般瀏覽模式不得因點擊或捲動誤觸拖曳
- 拖曳只能由面板標題區的拖曳把手開始；icon-only 控制必須提供 `aria-label` 與 tooltip
- 鍵盤使用者可聚焦拖曳把手，進入移動模式後以前後方向移動面板，並由可感知的狀態文字說明目前位置
- 支援尺寸的面板可在編輯模式切換預先定義的寬度，例如 `1/3`、`1/2`、`2/3`、`full`；統計卡片只支援不破壞內容的尺寸集合
- 小於桌面斷點時固定單欄或既有響應式欄數，自動忽略不適用的桌面座標與寬度；不得在手機版提供自由拖曳或任意像素尺寸
- 第一階段不新增後端 API。版面以 `localStorage` 儲存，key 需包含登入使用者識別、儀表板範圍與專案 ID，避免不同使用者或專案互相覆蓋
- 儲存資料包含 `schemaVersion`、面板順序與允許的尺寸代碼，不保存任意 HTML、樣式字串或無界限座標；資料無效或版本不相容時回復預設版面
- 提供「取消編輯」還原本次尚未儲存的變更，以及「重設版面」回復系統預設值；重設需先確認
- ECharts 容器尺寸變更後必須觸發 resize，Data Grid 必須維持可讀欄寬與既有分頁行為

#### 目標元件結構

```text
features/dashboard/
├── components/
│   ├── DashboardLayout.tsx       # 版面解析、響應式重排與編輯模式
│   ├── DashboardPanel.tsx        # 面板標題、拖曳把手與尺寸控制
│   └── StatCard.tsx
├── hooks/
│   └── useDashboardLayout.ts     # 驗證、載入、儲存與重設 localStorage 版面
└── pages/
    ├── DashboardPage.tsx
    └── ProjectDashboardPage.tsx
```

若需引入拖曳套件，實作前必須確認 React 19 相容性、鍵盤支援、bundle 影響與維護狀態；未確認前不得假設 `react-grid-layout`、`dnd-kit` 或其他套件已存在。

---

#### 視覺規格

全螢幕背景圖 + 中央毛玻璃卡片，參考設計稿：

```text
┌─────────────────────────────────────────────────────┐
│                  （全螢幕背景圖）                    │
│                                                     │
│              ┌──────────────────┐                   │
│              │      登入        │  ← 毛玻璃卡片     │
│              │                  │                   │
│              │ ┌──────────────┐ │                   │
│              │ │帳號或電子郵件 │ │  ← outlined      │
│              │ └──────────────┘ │                   │
│              │ ┌──────────────┐ │                   │
│              │ │ 密碼    👁   │ │  ← 眼睛 toggle   │
│              │ └──────────────┘ │                   │
│              │ ☐ 記住我  忘記密碼│                   │
│              │ [────── 登入 ──] │  ← 漸層按鈕       │
│              │ [── OIDC 登入 ─] │  ← 僅 OIDC 啟用時 │
│              └──────────────────┘                   │
│                                                     │
│                                    [繁體中文 ▼]     │  ← 右下角固定
└─────────────────────────────────────────────────────┘
```

> 右上角的圓形按鈕為瀏覽器插件，非系統功能，不實作。

### 背景

- 全螢幕背景圖（`background-size: cover`，`background-position: center`）
- 預設使用內建風景 / 星空圖片（打包進 `public/bg/login.jpg`）
- 背景圖上疊加半透明深色遮罩（`rgba(0,0,0,0.25)`），確保卡片對比度

### 毛玻璃卡片

```typescript
const cardSx = {
  width: 360,
  px: 4,
  py: 4,
  borderRadius: 3,
  backdropFilter: 'blur(16px) saturate(180%)',
  WebkitBackdropFilter: 'blur(16px) saturate(180%)',
  backgroundColor: 'rgba(255, 255, 255, 0.72)',  // 深色模式：rgba(20,20,30,0.65)
  border: '1px solid rgba(255, 255, 255, 0.4)',
  boxShadow: '0 8px 32px rgba(0,0,0,0.18)',
};
```

### 登入按鈕

漸層藍紫色，hover 時加深：

```typescript
const loginButtonSx = {
  background: 'linear-gradient(90deg, #5b8dee 0%, #a56eff 100%)',
  color: '#fff',
  fontWeight: 600,
  letterSpacing: 2,
  borderRadius: 2,
  py: 1.2,
  '&:hover': {
    background: 'linear-gradient(90deg, #4a7de0 0%, #9460f0 100%)',
  },
};
```

### 表單欄位

- 帳號欄位：`TextField` variant `outlined`，右側 adornment 放 `AccountCircleIcon`
- 密碼欄位：`TextField` type `password`，右側 `IconButton` 切換 `Visibility` / `VisibilityOff`
- 記住我：`FormControlLabel` + `Checkbox`，與「忘記密碼？」`Link` 同行左右對齊
- 所有欄位使用 React Hook Form + Zod 驗證，錯誤訊息透過 i18n key 顯示

### OIDC 登入按鈕

僅在後端 `oidc.enabled = true` 時顯示（前端透過 `GET /api/v1/auth/config` 取得）。樣式為 outlined 次要按鈕，文字 `t('auth.login_with_oidc')`。

### 語系切換（右下角固定）

```typescript
const langSelectSx = {
  position: 'fixed',
  bottom: 24,
  right: 24,
  zIndex: 10,
};
```

`Select` 選項：`zh-TW` → 繁體中文 / `zh-CN` → 简体中文 / `en` → English

### 登入頁 Toast 提示

登入頁所有即時狀態提示統一使用右上角 Toast，參考樣稿：

```text
┌────────────────────────────────────┐
│  ✓  Language switched successfully × │
│  ━━━━━━━━━━━━━                       │
└────────────────────────────────────┘
```

#### 觸發情境

- 語系切換成功：顯示成功 Toast，文字使用 `t('auth.language_switched')`
- 帳密登入失敗：顯示錯誤 Toast，優先使用後端錯誤訊息；無後端訊息時使用 `t('auth.login_failed')`
- 密碼錯誤 / 帳號停用 / 帳號不存在：顯示錯誤 Toast，依 API error code 對應 i18n 文案
- MFA 驗證失敗：顯示錯誤 Toast，使用 `t('auth.mfa_verify_failed')`
- MFA setup 初始化或驗證失敗：顯示錯誤 Toast，保留表單輸入並允許重試
- OIDC 啟動或 callback 失敗：顯示錯誤 Toast，使用 `t('auth.oidc_failed')`

#### 位置與樣式

```typescript
const loginToastSx = {
  position: 'fixed',
  top: 24,
  right: 24,
  zIndex: 1400,
  minWidth: 320,
  maxWidth: 'calc(100vw - 32px)',
  borderRadius: 1,
  backgroundColor: 'rgba(18, 18, 18, 0.94)',
  color: '#fff',
  boxShadow: '0 12px 32px rgba(0,0,0,0.28)',
};
```

- 使用 MUI `Snackbar` + `Alert` 或可重用 `LoginToast` 元件實作
- `anchorOrigin={{ vertical: 'top', horizontal: 'right' }}`
- 成功 icon 使用綠色圓形勾勾，錯誤 icon 使用紅色錯誤圖示
- 右上角提供關閉按鈕
- 顯示時間預設 `3000ms`，hover 時不強制關閉
- 底部進度條顯示剩餘時間；成功使用綠色，錯誤使用紅色
- Toast 不得遮擋登入卡片、MFA 卡片與右下角語系切換
- 手機版維持上方顯示，左右間距 `16px`，寬度 `calc(100vw - 32px)`

### MFA 驗證步驟

帳密驗證成功且後端回傳 `mfa_required: true` 時，卡片內容 in-place 切換為 TOTP 輸入畫面（不跳頁）。

若後端回傳 `mfa_setup_required: true`，代表系統已啟用強制 MFA，且目前用戶尚未綁定任何已驗證 MFA 裝置。前端需導向 `/auth/mfa/setup`，要求用戶完成 MFA 裝置新增後才能進入系統。

#### 視覺規格

```text
┌──────────────────────────────┐
│ ←                            │  ← 左上角返回按鈕，回到帳密畫面並清除 temp_token
│                              │
│         ╭──────────╮         │
│         │  🔒 icon │         │  ← 青色圓形背景 + 白色鎖頭 icon（LockIcon）
│         ╰──────────╯         │
│                              │
│        雙因素驗證             │  ← Typography variant="h6" fontWeight=700
│                              │
│  請輸入您的驗證器應用程式      │  ← 副標，Typography variant="body2" color="text.secondary"
│  所顯示的 6 位數驗證碼        │
│                              │
│  登入帳號：ad***@example.com  │  ← 遮罩顯示，Typography variant="body2"
│                              │
│ ┌── 驗證碼 ─────────────────┐ │
│ │      5  1  9  0  8  7    │ │  ← 大字體、字間距寬、inputMode="numeric"
│ └───────────────────────────┘ │
│                              │
│ [────────── 驗證 ───────────] │  ← 漸層藍色按鈕（同登入按鈕樣式）
│                              │
├──────────────────────────────┤
│  無法使用驗證器？             │  ← 底部灰色區塊（背景略深）
│  您可以使用備用碼登入          │  ← TextButton / Link，點擊切換為備用碼輸入模式
└──────────────────────────────┘
```

#### 帳號遮罩規則

```typescript
// ad***@example.com
function maskEmail(email: string): string {
  const [local, domain] = email.split('@');
  const visible = local.slice(0, 2);
  return `${visible}***@${domain}`;
}

// 非 email 格式（username）：取前 2 字元 + ***
function maskUsername(username: string): string {
  return `${username.slice(0, 2)}***`;
}
```

#### 驗證碼輸入框樣式

```typescript
const otpInputSx = {
  '& input': {
    fontSize: '2rem',
    fontWeight: 700,
    letterSpacing: '0.5em',
    textAlign: 'center',
  },
};

// TextField 屬性
<TextField
  inputProps={{
    inputMode: 'numeric',
    pattern: '[0-9]*',
    maxLength: 6,
    autoComplete: 'one-time-code',  // 觸發瀏覽器 / 簡訊自動填入
  }}
  autoFocus
  sx={otpInputSx}
/>
```

#### 備用碼模式

點擊「使用備用碼登入」後，輸入框切換為備用碼輸入（`type="text"`，格式 `XXXX-XXXX`），按鈕文字不變，仍呼叫同一支 `POST /api/v1/auth/mfa/verify`，payload 帶 `{ type: 'backup_code', code }`。

#### 鎖頭 Icon 樣式

```typescript
const lockIconWrapperSx = {
  width: 72,
  height: 72,
  borderRadius: '50%',
  background: 'linear-gradient(135deg, #7ecef4 0%, #5b8dee 100%)',
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
  mx: 'auto',
  mb: 2,
};
// <LockIcon sx={{ color: '#fff', fontSize: 36 }} />
```

### 強制 MFA 裝置新增頁

當全域設定 `security.mfa.force_enable = true`，且登入成功後偵測到用戶不存在已驗證 MFA 裝置時，前端顯示強制新增 MFA 裝置畫面。此頁不允許略過，只能完成綁定或返回登入頁。

#### 觸發條件

```text
帳密驗證成功
  → 後端回傳 mfa_setup_required: true + temp_token
  → 前端導向 /auth/mfa/setup
  → 完成 QR Code 掃描與 TOTP 驗證
  → 後端核發正式 access_token / refresh_token
  → 導向原本目標頁，預設 /dashboard
```

#### 視覺樣稿

依照提供圖片作為樣稿，使用登入頁相同的全螢幕背景與毛玻璃卡片。卡片寬度桌面版約 `500px`，手機版使用 `calc(100vw - 32px)`，內容置中排列。

```text
┌──────────────────────────────────────────────┐
│ ←                                            │
│                                              │
│                    盾牌 icon                 │
│                                              │
│              需要設定雙因素驗證              │
│      系統要求啟用雙因素驗證以保護您的帳號安全 │
│                                              │
│  1 掃描 QR Code ─ 2 驗證設定 ─ 3 保存備用碼  │
│                                              │
│ 使用您的驗證器應用程式掃描下方 QR Code       │
│                                              │
│                  QR Code                     │
│                                              │
│                或手動輸入密鑰：              │
│       RBI6BLCRN6FM6FLY2XRIAGOVQFK3MXPL [複製]│
│                                              │
│ ┌──────────────────────────────────────────┐ │
│ │ 裝置名稱（可選）                         │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│              [ 我已掃描完成 ]                │
└──────────────────────────────────────────────┘
```

#### 步驟設計

使用 `Stepper` 呈現三個步驟：

| 步驟 | 標題 | 內容 |
| ---- | ---- | ---- |
| 1 | 掃描 QR Code | 顯示 QR Code、手動密鑰、裝置名稱欄位 |
| 2 | 驗證設定 | 輸入驗證器產生的 6 位數 TOTP |
| 3 | 保存備用碼 | 顯示一次性備用碼，要求用戶確認已保存 |

#### 卡片與背景

```typescript
const forcedMfaPageSx = {
  minHeight: '100vh',
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
  px: 2,
  backgroundImage: `url(${loginBackgroundUrl})`,
  backgroundSize: 'cover',
  backgroundPosition: 'center',
};

const forcedMfaCardSx = {
  width: { xs: '100%', sm: 500 },
  maxWidth: 'calc(100vw - 32px)',
  px: { xs: 3, sm: 4 },
  py: 3,
  borderRadius: 2,
  backdropFilter: 'blur(18px) saturate(180%)',
  WebkitBackdropFilter: 'blur(18px) saturate(180%)',
  backgroundColor: 'rgba(255, 255, 255, 0.62)',
  border: '1px solid rgba(255, 255, 255, 0.35)',
  boxShadow: '0 16px 48px rgba(0,0,0,0.24)',
};
```

#### 標題區

```typescript
const shieldIconWrapperSx = {
  width: 72,
  height: 72,
  borderRadius: '50%',
  background: 'linear-gradient(135deg, #ffb74d 0%, #ff9800 100%)',
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
  mx: 'auto',
  mt: 1,
  mb: 2,
  boxShadow: '0 12px 28px rgba(255,152,0,0.35)',
};
// <SecurityIcon sx={{ color: '#fff', fontSize: 36 }} />
```

- 返回按鈕：左上角 `IconButton`，使用 `ArrowBackIcon`，點擊後清除 `temp_token` 並回到 `/login`
- 主標題：`Typography variant="h5"`，`fontWeight={700}`，文字 `需要設定雙因素驗證`
- 副標：`Typography variant="body2"`，`color="text.secondary"`，文字 `系統要求啟用雙因素驗證以保護您的帳號安全`

#### QR Code 區塊

```typescript
const qrCodeBoxSx = {
  width: 196,
  height: 196,
  mx: 'auto',
  p: 1,
  borderRadius: 1,
  backgroundColor: '#fff',
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
};

const manualSecretSx = {
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
  gap: 1,
  px: 2,
  py: 1.5,
  borderRadius: 1,
  backgroundColor: 'rgba(80,80,80,0.22)',
  fontFamily: 'monospace',
  fontWeight: 700,
  letterSpacing: 2,
  overflowWrap: 'anywhere',
};
```

- QR Code 由 `qr_code_url` 渲染，失敗時顯示 Skeleton 與錯誤提示
- 手動密鑰旁提供 `ContentCopyIcon` 按鈕，成功複製後顯示 `Snackbar`
- 裝置名稱欄位為可選，placeholder 使用 `裝置名稱（可選）`
- 步驟 1 主要按鈕文字為 `我已掃描完成`

#### 驗證設定步驟

```text
┌──────────────────────────────────────────────┐
│              輸入 6 位數驗證碼               │
│  請輸入驗證器應用程式目前顯示的驗證碼         │
│                                              │
│ ┌──────────────────────────────────────────┐ │
│ │             1 2 3 4 5 6                  │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│              [ 返回 ] [ 驗證並綁定 ]          │
└──────────────────────────────────────────────┘
```

- 驗證碼欄位沿用登入 MFA 的 `otpInputSx`
- 僅允許 6 位數字，送出前由 Zod 驗證
- 驗證失敗時保留在步驟 2，顯示後端錯誤訊息

#### 保存備用碼步驟

```text
┌──────────────────────────────────────────────┐
│                保存您的備用碼                │
│  備用碼只會顯示一次，請保存於安全位置         │
│                                              │
│  ABCD-EFGH   IJKL-MNOP   QRST-UVWX          │
│  1234-5678   90AB-CDEF   GHIJ-KLMN          │
│                                              │
│  [ ] 我已保存備用碼                           │
│                                              │
│              [ 完成設定 ]                    │
└──────────────────────────────────────────────┘
```

- 若後端尚未支援備用碼，步驟 3 顯示完成確認，不顯示備用碼清單
- `完成設定` 按鈕需在用戶勾選確認後才可點擊
- 完成後清除 `temp_token`，更新目前使用者狀態並導向原目標頁

#### API 流程

1. 進入頁面時呼叫 `POST /api/v1/auth/mfa/setup`，Header 帶 `Authorization: Bearer {temp_token}`
2. 後端回傳 `secret`、`qr_code_url`、`setup_id`
3. 步驟 2 呼叫 `POST /api/v1/auth/mfa/verify`，payload 帶 `{ setup_id, code, device_name, is_setup: true }`
4. 驗證成功後若回傳 `backup_codes`，進入步驟 3；否則直接完成登入流程
5. 完成後儲存正式 token，並導向登入前的 `redirectTo`

#### 元件結構

```text
features/auth/
├── ForcedMfaSetupPage.tsx       # 強制 MFA 裝置新增頁
├── ForcedMfaSetupStepper.tsx    # 三步驟流程
├── MfaQrSetupStep.tsx           # QR Code 與手動密鑰
├── MfaVerifySetupStep.tsx       # TOTP 驗證
├── MfaBackupCodesStep.tsx       # 備用碼保存
└── hooks/
    └── useForcedMfaSetup.ts     # setup / verify mutation 與 temp_token 狀態
```

### 元件結構

```text
features/auth/
├── LoginPage.tsx          # 頁面容器（背景、卡片定位）
├── LoginForm.tsx          # 帳密表單
├── MfaForm.tsx            # TOTP 輸入表單
├── ForcedMfaSetupPage.tsx # 強制 MFA 裝置新增頁
├── OidcButton.tsx         # OIDC 登入按鈕（條件渲染）
└── hooks/
    └── useLogin.ts        # 登入邏輯（useMutation + 狀態機）
```

### 登入流程狀態機

```text
idle → submitting → mfa_required → mfa_submitting → success → redirect
                 ↘ mfa_setup_required → setup_submitting → success → redirect
                 ↘ success → redirect
                 ↘ error → idle
```

### 閒置時間自動登出

登入成功並進入受保護頁面後，前端需透過 `GET /api/v1/auth/session-policy` 取得會話政策。Response 僅使用 `idle_timeout_seconds`，不得改由 `/api/v1/admin/global-settings/:key` 讀取 `security.session.timeout`，也不得顯示設定 key、分類、描述、來源或其他安全設定。

前端需在登入狀態下監聽使用者活動事件：`pointerdown`、`keydown`、`scroll`、`touchstart`、`visibilitychange`。每次有效活動更新最後活動時間，並依 `idle_timeout_seconds` 設定下一次檢查。超過設定秒數無操作時，前端需清除 Access Token、Refresh Token、目前使用者、目前專案與 React Query cache，顯示 `notification.idle_timeout` Toast，並導回 `/login?redirectTo=...`。

`idle_timeout_seconds <= 0` 時停用前端閒置自動登出。API 載入失敗或回傳格式無效時，前端以預設 28800 秒處理；後端 JWT Access Token 與 Refresh Token 到期仍是最終安全邊界。登入頁、MFA setup / verify、SSO callback 與 public-only 頁面不得啟用 idle logout timer。

測試驗收：

- 單元測試需使用 fake timer 覆蓋超過 `idle_timeout_seconds` 後清除 token、目前使用者、目前專案與 React Query cache，並導回登入頁
- 單元測試需覆蓋 `pointerdown`、`keydown`、`scroll`、`touchstart`、`visibilitychange` 會刷新最後活動時間並延後登出
- 單元測試需覆蓋 `idle_timeout_seconds <= 0` 不啟動 timer，API 失敗或回傳格式無效時使用 28800 秒
- 整合或手動驗收需以短 timeout 設定確認受保護頁會自動登出，登入頁、MFA setup / verify、SSO callback 不會啟動 idle logout timer
- 閒置自動登出 Toast 需使用 `notification.idle_timeout`，不得誤用 token 過期的 `notification.session_expired`

---

## 報表設計（需求 14）

> **Phase 3** 實作報表設計器與範本；**Phase 2** 僅基礎查詢；MVP 不包含。
> Report 後續前端頁面、元件、i18n 與 task 以 `.kiro/specs/2026-06-22_11-06_Report` 為準；本段保留舊版設計背景。

使用 `echarts-for-react` 渲染，圖表資料由後端 API 計算後回傳（`ReportChartPayload`），前端負責渲染與 CSV 匯出（14.10）。

### 報表中心頁面（`/projects/:id/reports`）

```text
報表中心
├── 範本列表（TemplateList）          — 專案共用範本，PM+ 可新增 / 編輯 / 刪除
├── 執行範本（TemplateExecute）       — 選時間區間（本週 / 上週 / 本月 / 上月 / 自訂）→ 預覽 + 匯出
├── 報表設計器（ReportDesigner）      — 模式 D：維度 / 指標 / 圖表類型即時預覽
└── 內建月報（BuiltinMonthlyReport）  — 快速選 A / B / C 版型，無需先建範本
```

```text
features/report/
├── ReportCenterPage.tsx
├── TemplateList.tsx
├── TemplateExecute.tsx
├── ReportDesigner.tsx          # 模式 D
├── BuiltinMonthlyReport.tsx    # 模式 A / B / C 快速入口
├── charts/
│   ├── WeeklyBarChart.tsx
│   ├── ModeAIndicatorChart.tsx
│   ├── ModeBTaskStackChart.tsx
│   ├── ModeCPersonSubProjectChart.tsx   # 人員 × 子專案堆疊（OP 月報）
│   └── CustomChart.tsx                  # 模式 D 通用渲染
└── hooks/
    ├── useReportPreview.ts
    └── useReportTemplates.ts
```

### 週報（14.7，Bar Chart）

```text
橫軸：日期（Mon 6/2 ~ Sun 6/8）
縱軸：Ticket 數量
分組：班別（早/中/晚）或個人
```

### 月報模式 A（指標導向，Bar Chart）

```text
每個統計指標獨立一張圖
橫軸：週區間（6/1-6/7、6/8-6/14 ...）+ 總計
縱軸：個人（班別-姓名）
附：明細數字表
```

### 月報模式 B（任務導向，Stacked Bar）

```text
縱軸：Ticket 標題（任務內容）
橫軸：個人
堆疊：Ticket 數量（依任務分色）
附：交叉明細表（作業or問題 × 人員）
```

### 月報模式 C（人員 × 子專案堆疊，Stacked Bar）

對應 OP 月報「運維處理事件數量」（整月單圖）。

```text
標題：2026年0501-0531運維處理事件數量
橫軸：人員（Bing、Ralph、OPHelp ...）
堆疊區段：子專案 / 業務項目（圖例，各項目一色）
縱軸：Ticket 數量
統計口徑：範本可選 created_by 或 ticket_activities.actor
```

ECharts 建議設定：

```typescript
// ModeCPersonSubProjectChart — series 每個 sub_project 一條 stack
{
  title: { text: '2026年0501-0531運維處理事件數量' },
  xAxis: { type: 'category', data: persons },
  yAxis: { type: 'value' },
  series: subProjects.map(sp => ({
    name: sp.label,
    type: 'bar',
    stack: 'total',
    data: countsByPerson[sp.id],
  })),
  legend: { bottom: 0 },
}
```

### 月報模式 D（設計器自訂維度）

報表設計器 UI 欄位（對應 14.1）：

| 欄位 | 選項 |
| ------ | ------ |
| 橫軸 | 日期（日/週/月）、人員、子專案 |
| 縱軸分組 | 班制、個人 |
| 統計指標 | Ticket 數量 ×（事件類型 / 資訊來源 / 子專案） |
| 圖表類型 | 長條 / 堆疊長條 |
| 時間區間 | 整月 / 自訂（execute 時再選亦可） |

流程：選維度 → `POST /reports/preview` 即時預覽 → 儲存為範本 → 之後從範本列表執行。不受 A/B/C 固定軸向限制。

### 儀表板

- 數字卡片：進行中 Ticket 數、今日新增、今日解決、SLA 違反數
- 優先級分佈：Donut Chart（P1/P2/P3/P4）
- 每 30 秒自動刷新（`refetchInterval: 30_000`）

---

## 個人資料頁設計（`/settings/profile`）

### 頁面結構

```text
個人資料                          ← 頁面標題 Typography h5

┌─────────────────────────────────────────────────────┐
│  👤 基本資料  🔒 修改密碼  🛡 MFA 設定               │  ← Tabs
│ ─────────────                                       │  ← 底線 indicator
│                                                     │
│  ╭──────╮  admin                                    │
│  │  AD  │  系統管理員                               │  ← Avatar 80px + username + 角色
│  ╰──────╯                                           │
│                                                     │
│  ┌── 暱稱 ──────────────────────────────────────┐   │
│  │  系統管理員                                   │   │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌── 電子郵箱 ────────────────────────────────────┐  │
│  │  admin@example.com                            │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌── 聯絡電話 ────────────────────────────────────┐  │
│  │  13800138999                                  │  │
│  └───────────────────────────────────────────────┘  │
│  138-0013-8999                ← helperText 格式提示  │
│                                                     │
│                              [✓ 儲存]               │  ← 右下角
└─────────────────────────────────────────────────────┘
```

### Tab 定義

| Tab | Icon | 路由 hash |
| ----- | ------ | ---------- |
| 基本資料 | `PersonIcon` | `#profile` |
| 修改密碼 | `LockIcon` | `#password` |
| MFA 設定 | `SecurityIcon` | `#mfa` |

Tab 切換不跳頁，使用 hash 或 `useState` 控制，URL 保持 `/settings/profile`。

### Avatar 區塊

```typescript
// 大 Avatar：80px，青藍色漸層背景，顯示 full_name 前兩字或首字母
const avatarSx = {
  width: 80,
  height: 80,
  fontSize: '1.8rem',
  fontWeight: 700,
  background: 'linear-gradient(135deg, #7ecef4 0%, #5b8dee 100%)',
};

function getAvatarText(fullName: string): string {
  // 中文：取前兩字；英文：取首字母大寫
  const isChinese = /[\u4e00-\u9fa5]/.test(fullName);
  return isChinese ? fullName.slice(0, 2) : fullName.slice(0, 2).toUpperCase();
}
```

Avatar 下方顯示 `username`（`Typography variant="h6"`）和角色名稱（`Typography variant="body2" color="text.secondary"`）。

### 基本資料表單

欄位：

| 欄位 | 對應 DB | 備註 |
| ------ | --------- | ------ |
| 暱稱 | `full_name` | 必填 |
| 電子郵箱 | `email` | 必填，email 格式驗證 |
| 聯絡電話 | `users.phone` | 選填，helperText 顯示格式範例 |

`username` 和 `global_role` 唯讀，不顯示在表單中（僅顯示於 Avatar 下方）。

```typescript
// Zod schema
const profileSchema = z.object({
  full_name: z.string().min(1, t('error.required')),
  email: z.string().email(t('error.invalid_email')),
  phone: z.string().optional(),
});
```

### 儲存按鈕

```typescript
const saveButtonSx = {
  background: 'linear-gradient(90deg, #5b8dee 0%, #42a5f5 100%)',
  color: '#fff',
  px: 3,
  borderRadius: 2,
  '&:hover': {
    background: 'linear-gradient(90deg, #4a7de0 0%, #2196f3 100%)',
  },
};
// <Button startIcon={<CheckIcon />}>{t('action.save')}</Button>
```

儲存成功後顯示 Toast（`severity="success"`）；儲存失敗時顯示錯誤 Toast，訊息優先使用後端回傳內容。

基本資料儲存時必須送出 `full_name`、`email`、`phone`；`phone` 對應後端既有 `users.phone` 欄位，不得在前端隱藏或從 request payload 省略。

### 修改密碼 Tab

```text
┌── 目前密碼 ──────────────────────────────────────┐
│                                            👁   │
└─────────────────────────────────────────────────┘
┌── 新密碼 ────────────────────────────────────────┐
│                                            👁   │
└─────────────────────────────────────────────────┘
┌── 確認新密碼 ────────────────────────────────────┐
│                                            👁   │
└─────────────────────────────────────────────────┘
                                    [✓ 更新密碼]
```

Zod 驗證：新密碼長度需大於等於 `GET /api/v1/auth/password-policy` 回傳的 `min_length`，確認密碼需與新密碼一致（`.refine`）。若前端尚未取得 policy、API 失敗或回傳值無效，表單提示與驗證先以預設值 4 處理；後端 `PUT /api/v1/auth/password` 仍是最終驗證來源。

前端不得直接呼叫 `/api/v1/admin/global-settings/:key` 取得密碼長度，也不得要求後端提供可由瀏覽器端解密的混淆或加密設定內容。密碼政策 API 僅提供 `min_length`，不顯示設定 key、分類、描述、來源或其他安全設定。

送出修改密碼時呼叫 self-service API：`PUT /api/v1/auth/password`，payload 為 `{ current_password, new_password }`。後端必須驗證 `current_password` 與目前登入使用者的 `password_hash` 相符後才更新新密碼；前端不得沿用 `/api/v1/admin/users/:id` 或其他 Admin 使用者更新 API 直接修改自己的密碼。

成功後清空三個密碼欄位並顯示成功 Toast；目前密碼錯誤、新密碼格式錯誤或系統錯誤時顯示錯誤 Toast，訊息優先使用後端回傳內容。

### MFA 設定 Tab

對應 `/settings/profile#mfa`，顯示 2FA 狀態與已綁定裝置列表。

#### 視覺規格

```text
┌─────────────────────────────────────────────────────┐
│  🛡  雙因素驗證 (2FA)                      [已啟用] │  ← 狀態卡片
│      您的帳號已啟用雙因素驗證                        │
└─────────────────────────────────────────────────────┘

已綁定的驗證器(2)                ← Typography variant="subtitle1" fontWeight=700

┌─────────────────────────────────────────────────────┐
│  📱  TOTP Authenticator  [主要]              [🗑]   │
│      類型：TOTP                                     │
│      綁定時間：2025/12/3 下午 4:12:41               │
│      最後使用：2026/6/1 下午 4:32:29                │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  📱  TOTP Authenticator                    [🗑]    │
│      類型：TOTP                                     │
│      綁定時間：2025/12/4 上午 9:55:12               │
│      最後使用：2025/12/4 上午 9:55:42               │
└─────────────────────────────────────────────────────┘

[+ 新增裝置]                     ← 漸層藍色按鈕

╭─────────────────────────────────────────────────────╮
│ ℹ  雙因素驗證 (2FA) 會在您登入時要求輸入驗證碼...    │  ← info 提示區塊
╰─────────────────────────────────────────────────────╯
```

#### 2FA 狀態卡片

```typescript
// 已啟用：綠色盾牌 + 綠色 Chip
// 未啟用：灰色盾牌 + 灰色 Chip
const statusCardSx = {
  display: 'flex',
  alignItems: 'center',
  gap: 2,
  p: 2.5,
  borderRadius: 2,
  backgroundColor: 'rgba(255,255,255,0.04)',
  border: '1px solid rgba(255,255,255,0.08)',
};

// 盾牌 icon 顏色
const shieldColor = hasMfa ? '#4caf50' : 'text.disabled';  // 綠 / 灰

// 狀態 Chip
<Chip
  label={hasMfa ? t('mfa.enabled') : t('mfa.disabled')}
  color={hasMfa ? 'success' : 'default'}
  variant="outlined"
  size="small"
/>
```

#### 裝置列表卡片

```typescript
interface MfaDevice {
  id: string;
  name: string;          // 'TOTP Authenticator'
  is_verified: boolean;
  created_at: string;    // 綁定時間
  last_used_at?: string; // 最後使用
}

// 第一個已驗證裝置標記「主要」
const isPrimary = (index: number, device: MfaDevice) =>
  index === 0 && device.is_verified;
```

```typescript
const deviceCardSx = {
  display: 'flex',
  alignItems: 'flex-start',
  gap: 2,
  p: 2,
  borderRadius: 2,
  backgroundColor: 'rgba(255,255,255,0.04)',
  border: '1px solid rgba(255,255,255,0.08)',
  mb: 1.5,
};

// 手機 icon：綠色
// <PhoneAndroidIcon sx={{ color: '#4caf50', mt: 0.5 }} />

// 「主要」Chip：藍色
<Chip label={t('mfa.primary')} color="primary" size="small" sx={{ ml: 1 }} />

// 刪除按鈕：紅色垃圾桶，點擊彈出確認 Dialog
<IconButton
  onClick={() => handleDeleteDevice(device.id)}
  sx={{ color: 'error.main', ml: 'auto' }}
>
  <DeleteIcon />
</IconButton>
```

時間格式：`yyyy/M/d 上午/下午 h:mm:ss`（使用 `date-fns` + `zh-TW` locale）。

#### 新增裝置按鈕

```typescript
const addButtonSx = {
  background: 'linear-gradient(90deg, #5b8dee 0%, #42a5f5 100%)',
  color: '#fff',
  borderRadius: 2,
  px: 2.5,
  '&:hover': {
    background: 'linear-gradient(90deg, #4a7de0 0%, #2196f3 100%)',
  },
};
// <Button startIcon={<AddIcon />}>{t('mfa.add_device')}</Button>
```

點擊後開啟「新增 MFA 裝置」Dialog（QR Code 掃描流程，見下方）。

#### 新增裝置 Dialog（QR Code 綁定流程）

```text
步驟 1：顯示 QR Code
┌──────────────────────────────────┐
│  新增驗證器                  [✕] │
│                                  │
│  1. 開啟 Google Authenticator    │
│     或 Microsoft Authenticator   │
│  2. 掃描下方 QR Code             │
│                                  │
│         ┌──────────┐             │
│         │  QR Code │             │
│         └──────────┘             │
│                                  │
│  無法掃描？手動輸入金鑰：         │
│  JBSWY3DPEHPK3PXP  [複製]        │
│                                  │
│              [下一步]            │
└──────────────────────────────────┘

步驟 2：驗證 TOTP
┌──────────────────────────────────┐
│  驗證設定                    [✕] │
│                                  │
│  請輸入驗證器顯示的 6 位數驗證碼  │
│  ┌──────────────────────────┐    │
│  │  6 位數字（大字體）       │    │
│  └──────────────────────────┘    │
│                                  │
│         [返回]  [確認綁定]        │
└──────────────────────────────────┘
```

流程：

1. `POST /api/v1/auth/mfa/setup` → 取得 `secret` + `qr_code_url`
2. 前端用 `qrcode` 套件渲染 QR Code（`npm i qrcode`）
3. 使用者掃描後輸入 TOTP → `POST /api/v1/auth/mfa/verify`（帶 `is_setup: true`）
4. 驗證成功 → 關閉 Dialog，刷新裝置列表，顯示 `Snackbar` 成功提示

前端行為補充：

- 新增 MFA 裝置時需允許使用者自行輸入裝置名稱；名稱用於後端 `POST /api/v1/auth/mfa/setup` 的 `name`
- 尚未完成 TOTP 驗證的 setup 裝置不得顯示在「已綁定裝置」列表；列表只呈現後端回傳的已驗證裝置，或在前端過濾 `is_verified = true`
- 使用者關閉 Dialog、刷新頁面或離開流程時，不需把未完成驗證的裝置視為已綁定；後端需透過 `clean_unverified_mfa_devices` 自動清理逾時待驗證裝置
- 若 setup 初始化已建立待驗證裝置但驗證未完成，下一次新增流程仍需允許重新產生 QR Code，不得因舊的未驗證裝置名稱衝突阻塞使用者

#### 刪除確認 Dialog

```text
┌──────────────────────────────────┐
│  移除驗證器                  [✕] │
│                                  │
│  確定要移除「TOTP Authenticator」│
│  嗎？移除後需重新綁定才能使用    │
│  雙因素驗證。                    │
│                                  │
│              [取消]  [移除]      │  ← 移除按鈕 color="error"
└──────────────────────────────────┘
```

#### Info 提示區塊

```typescript
const infoBoxSx = {
  display: 'flex',
  gap: 1.5,
  p: 2,
  borderRadius: 2,
  backgroundColor: 'rgba(33, 150, 243, 0.08)',
  border: '1px solid rgba(33, 150, 243, 0.2)',
  mt: 2,
};
// <InfoOutlinedIcon sx={{ color: 'info.main', flexShrink: 0 }} />
// 文字：t('mfa.info_text')
// 推薦 Google Authenticator 或 Microsoft Authenticator
```

---

`AttachmentUpload` 元件支援：

- 點擊選擇檔案
- 拖曳上傳
- 直接貼上截圖（`paste` 事件監聽 `ClipboardEvent`）

建立 Ticket 頁尚未取得 `ticket_id` 前，附件需先暫存在前端 pending list；送出建立成功後，再使用新 Ticket ID 呼叫附件上傳 API 逐一上傳。若附件上傳失敗，Ticket 建立結果仍保留，介面需提示「Ticket 已建立，但附件上傳失敗」並導向 Ticket 詳情。

```typescript
// 貼上截圖
document.addEventListener('paste', (e) => {
  const items = e.clipboardData?.items;
  for (const item of items ?? []) {
    if (item.type.startsWith('image/')) {
      const file = item.getAsFile();
      if (file) uploadAttachment(ticketId, file);
    }
  }
});
```

驗證：content-type 白名單、單檔 ≤ 10MB、數量 ≤ 20。

**附件顯示（需求 3.12）：**

- 列表 API 僅回傳 metadata（`id`、`filename`、`content_type`、`size_bytes`），不含 `storage_key`
- 圖片預覽 / 下載統一請求 `GET /api/v1/attachments/:id/content`（帶 JWT）
- React 使用 authenticated fetch 或 axios `responseType: 'blob'` 建立 object URL，或直接以帶 Token 的 URL 供 `<img>` 使用（需確保請求帶 `Authorization` header）

```typescript
// 範例：以 blob URL 顯示附件
const res = await api.get(`/attachments/${id}/content`, { responseType: 'blob' });
const objectUrl = URL.createObjectURL(res.data);
// <img src={objectUrl} alt={filename} />
```

S3 模式下 Browser **不**直接連線 S3；所有內容經 Go 串流代理。

---

---

## 排程管理頁設計（`/admin/schedulers`）

### 頁面結構

兩個 Tab：**任務列表** / **執行記錄**

---

### Tab 1：任務列表

```text
系統管理 > 排程管理

┌─ 任務列表 ──┬─ 執行記錄 ─┐

[🔍 搜尋任務名稱]  [+ 新增]

┌──────────────────────────────────────────────────────────────────┐
│ ID  任務名稱        任務標識                  Cron  狀態  操作    │
├──────────────────────────────────────────────────────────────────┤
│ 1   清理過期安全稽核日誌 clean_expired_security_audit_logs 0 2 * * * 啟用 👁 ✏ ▶ 🗑 │
│ 2   分區維護        partition_maintenance      0 0 1 * * 啟用 👁 ✏ ▶ 🗑 │
│ ...                                                              │
└──────────────────────────────────────────────────────────────────┘
```

#### 任務列表欄位

| 欄位 | 說明 |
| ------ | ------ |
| ID | 序號 |
| 任務名稱 | `task_name` |
| 任務標識 | `task_key`（唯一識別碼，英文底線） |
| Cron | `cron_expr`，顯示原始表達式 |
| 狀態 | 啟用（綠色 Chip）/ 停用（灰色 Chip） |
| 操作 | 👁 詳情 / ✏ 編輯 / ▶ 立即執行 / 🗑 刪除 |

#### 新增 / 編輯 Dialog

```
┌── 編輯任務 ──────────────────────────────────┐
│                                          [✕] │
│  ┌── 任務名稱 ────────────────────────────┐  │
│  │  清理過期安全稽核日誌                   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── Cron 表達式 ─────────────────────────┐  │
│  │  0 2 * * *                             │  │
│  └────────────────────────────────────────┘  │
│  例如：0 0 * * *（每小時執行）               │  ← helperText
│                                              │
│  ┌── 描述 ────────────────────────────────┐  │
│  │  根據系統設定的保留天數，自動清理...    │  │  ← multiline rows=4
│  └────────────────────────────────────────┘  │
│                                              │
│                      [取消]  [儲存]          │  ← 儲存藍色漸層
└──────────────────────────────────────────────┘
```

```typescript
const schedulerSchema = z.object({
  task_name:  z.string().min(1).max(128),
  cron_expr:  z.string().regex(/^(\S+\s){4}\S+$/, t('error.invalid_cron')),
  description: z.string().optional(),
});
```

#### 任務詳情 Dialog

點擊眼睛 icon 展開，key-value 列表顯示所有欄位：

```text
┌── 任務詳情 ──────────────────────────────────┐
│                                              │
│  id:           1                             │
│  task_name:    清理過期安全稽核日誌           │
│  task_key:     clean_expired_security_audit_logs │
│  task_type:    system                        │
│  cron_expr:    0 2 * * *                     │
│  task_config:  （空）                        │
│  description:  根據系統設定的保留天數...      │
│  status:       1（啟用）                     │
│  last_run_at:  2025-12-05T16:29:23+08:00     │
│  next_run_at:  2025-12-06T02:00:00+08:00     │
│  created_at:   2025-12-04T17:35:27+08:00     │
│  updated_at:   2025-12-05T16:29:23+08:00     │
│                                              │
│                                    [關閉]    │
└──────────────────────────────────────────────┘
```

```typescript
// key-value 列表樣式
const kvRowSx = {
  display: 'grid',
  gridTemplateColumns: '160px 1fr',
  gap: 1,
  py: 0.75,
  borderBottom: '1px solid rgba(255,255,255,0.06)',
};
// key：Typography variant="body2" fontWeight=700 color="text.secondary"
// value：Typography variant="body2"
```

---

### Tab 2：執行記錄

```text
┌─ 任務列表 ──┬─ 執行記錄 ─┐

[搜尋任務名稱] [搜尋任務標識]  [🔍 搜尋]  [🔄 刷新]  [🗑 批量刪除]  [🧹 清空]

☐ ID▼  任務名稱▼  任務標識▼  狀態▼  訊息▼  耗時▼  執行時間▼  操作▼
☐ 14   清理過期操...  clean_e...  [成功]  操作日誌清理完成...  12ms  2026/6/1...  👁 🗑
☐ 13   清理過期登...  clean_e...  [成功]  登入日誌清理完成...  25ms  2025/12/5... 👁 🗑
...

Page Size: [20▼]    1 to 14 of 14    |< < Page 1 of 1 > >|
```

#### 執行記錄欄位

| 欄位 | 說明 |
| ------ | ------ |
| checkbox | 多選，用於批量刪除 |
| ID | 序號 |
| 任務名稱 | 截斷 + Tooltip |
| 任務標識 | `task_key`，截斷 + Tooltip |
| 狀態 | 成功（綠色 Chip）/ 失敗（紅色 Chip）/ 執行中（藍色 Chip） |
| 訊息 | 執行結果訊息，截斷 + Tooltip |
| 耗時 | `12ms`、`25ms`，`-` 表示執行中 |
| 執行時間 | `yyyy/M/d 上午/下午 h:mm:ss` |
| 操作 | 👁 詳情（`VisibilityIcon`）+ 🗑 刪除（`DeleteIcon` 紅色） |

#### 批量操作按鈕

```typescript
// 批量刪除：選中項 > 0 才啟用
<Button
  variant="outlined"
  color="error"
  startIcon={<DeleteIcon />}
  disabled={selectedIds.length === 0}
  onClick={handleBatchDelete}
>
  {t('action.batch_delete')}（{selectedIds.length}）
</Button>

// 清空：清除所有執行記錄，需二次確認 Dialog
<Button
  variant="outlined"
  color="warning"
  startIcon={<ClearAllIcon />}
  onClick={() => setConfirmClearOpen(true)}
>
  {t('action.clear_all')}
</Button>
```

#### 執行記錄詳情 Dialog

```text
┌── 執行記錄詳情 ──────────────────────────────┐
│                                          [✕] │
│  任務名稱：清理過期安全稽核日誌               │
│  任務標識：clean_expired_security_audit_logs  │
│  狀態：    成功                              │
│  耗時：    12ms                              │
│  執行時間：2026/6/1 下午 4:55:xx             │
│                                              │
│  執行訊息：                                  │
│  ┌──────────────────────────────────────┐   │
│  │  操作日誌清理完成，共刪除 91 筆記錄   │   │  ← pre 格式化
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

### 後端 DB 擴充

`scheduler_logs` 表新增欄位以支援前端顯示：

```sql
-- 現有 scheduler_logs 已有：id, name, started_at, finished_at, status, detail
-- 新增欄位
ALTER TABLE scheduler_logs
  ADD COLUMN task_key   VARCHAR(64),   -- 對應 task_key
  ADD COLUMN duration_ms INT;          -- 耗時毫秒（finished_at - started_at）
```

`schedulers` 任務定義表（新增）：

```sql
CREATE TABLE schedulers (
  id          SERIAL PRIMARY KEY,
  task_name   VARCHAR(128) NOT NULL,
  task_key    VARCHAR(64) UNIQUE NOT NULL,
  task_type   VARCHAR(32) NOT NULL DEFAULT 'system',
  cron_expr   VARCHAR(64) NOT NULL,
  task_config JSONB,
  description TEXT,
  status      SMALLINT NOT NULL DEFAULT 1,  -- 1=啟用, 0=停用
  last_run_at TIMESTAMPTZ,
  next_run_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO schedulers (task_name, task_key, task_type, cron_expr, description) VALUES
  ('分區維護',         'partition_maintenance',          'system', '0 0 1 * *',   '每月 1 日建立下兩個月分區，並依 system.log.ticket_activity_keep_days 清理 ticket_activities 分區（0 表示不清理）'),
  ('SLA 檢查',         'sla_checker',                    'system', '*/5 * * * *', '每 5 分鐘檢查 SLA 超時，寫入 ticket_activities'),
  ('清理過期安全稽核日誌', 'clean_expired_security_audit_logs', 'system', '0 2 * * *', '依 system.log.security_audit_keep_days 清理'),
  ('清理過期系統稽核日誌', 'clean_expired_system_audit_logs',   'system', '0 2 * * *', '依 system.log.system_audit_keep_days 清理'),
  ('清理過期登入日誌', 'clean_expired_login_logs',       'system', '0 2 * * *',   '根據系統設定的保留天數，自動清理過期的登入日誌'),
  ('清理過期排程日誌', 'clean_expired_scheduler_logs',   'system', '0 2 * * *',   '根據系統設定的保留天數，自動清理過期的排程日誌');
```

---

#### 頁面結構

```text
系統管理 > 角色管理          ← 麵包屑

[🔍 搜尋...]  [+ 新增]

┌──────────────────────────────────────────────────────────────┐
│ ID  角色清單    角色代碼    角色狀態  角色備註              操作 │
├──────────────────────────────────────────────────────────────┤
│ 1   系統管理員  admin      啟用      系統管理員，擁有所有權限  ✏ 🗑 │
│ 2   管理員      manager    啟用      系統管理員，擁有大部分...  ✏ 🗑 │
│ 3   操作員      operator   啟用      系統操作員，可執行常規操作 ✏ 🗑 │
│ 4   只讀用戶    viewer     啟用      只讀用戶，僅有查看權限    ✏ 🗑 │
└──────────────────────────────────────────────────────────────┘

Page Size: [10▼]    1 to 4 of 4    |< < Page 1 of 1 > >|
```

### 欄位定義

| 欄位 | 寬度 | 說明 |
| ------ | ------ | ------ |
| ID | 60 | 自動遞增序號 |
| 角色清單 | 140 | 角色顯示名稱（`name`） |
| 角色代碼 | 120 | 角色唯一識別碼（`code`），英文小寫 |
| 角色狀態 | 80 | `is_active`：啟用（綠色 Chip）/ 停用（灰色 Chip） |
| 角色備註 | 240 | `description`，超長截斷 + `Tooltip` |
| 操作 | 80 | 編輯 `EditIcon`（藍色）+ 刪除 `DeleteIcon`（紅色） |

> 內建角色（`admin`、`viewer`）刪除按鈕灰色禁用，hover `Tooltip` 說明「內建角色不可刪除」。

### 預設角色種子資料

| ID | 名稱 | 代碼 | 備註 |
| ---- | ------ | ------ | ------ |
| 1 | 系統管理員 | `admin` | 系統管理員，擁有所有權限 |
| 2 | 管理員 | `manager` | 系統管理員，擁有大部分管理權限 |
| 3 | 操作員 | `operator` | 系統操作員，可執行常規操作 |
| 4 | 只讀用戶 | `viewer` | 只讀用戶，僅有查看權限 |

### 新增 / 編輯 Dialog

```text
┌── 新增角色 ──────────────────────────────────┐
│                                          [✕] │
│  ┌── 角色名稱 * ──────────────────────────┐  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── 角色代碼 * ──────────────────────────┐  │  ← 英文小寫 + 底線，建立後唯讀
│  └────────────────────────────────────────┘  │
│  角色代碼建立後不能修改                       │  ← helperText
│                                              │
│  ┌── 備註 ────────────────────────────────┐  │  ← multiline rows=3
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  狀態  ● 啟用  ○ 停用                        │
│                                              │
│                      [取消]  [✓ 儲存]        │
└──────────────────────────────────────────────┘
```

```typescript
const roleSchema = z.object({
  name:        z.string().min(1).max(64),
  code:        z.string().regex(/^[a-z][a-z0-9_]*$/, t('error.invalid_role_code')),
  description: z.string().optional(),
  is_active:   z.boolean(),
});
```

### 後端 DB

```sql
CREATE TABLE roles (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(64) UNIQUE NOT NULL,
  code        VARCHAR(32) UNIQUE NOT NULL,
  description TEXT,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  is_builtin  BOOLEAN NOT NULL DEFAULT FALSE,  -- 內建角色不可刪除
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO roles (name, code, description, is_builtin) VALUES
  ('系統管理員', 'admin',    '系統管理員，擁有所有權限',         TRUE),
  ('管理員',     'manager',  '系統管理員，擁有大部分管理權限',   FALSE),
  ('操作員',     'operator', '系統操作員，可執行常規操作',       FALSE),
  ('只讀用戶',   'viewer',   '只讀用戶，僅有查看權限',           TRUE);
```

### 後端 API

```text
GET    /api/v1/admin/roles          — 列表（分頁 + 關鍵字）
POST   /api/v1/admin/roles          — 新增
GET    /api/v1/admin/roles/:id      — 單筆
PUT    /api/v1/admin/roles/:id      — 更新（is_builtin=true 不可刪除）
DELETE /api/v1/admin/roles/:id      — 刪除（is_builtin=true 回傳 403）
```

---

## 群組管理與選單權限設定（`/admin/groups`）

> 此流程從選單管理搬出，作為系統管理下的獨立「群組管理」。角色管理頁只負責角色 CRUD；選單 / 表單權限需串接後端 `groups`、`group_members`、`group_form_permissions` API。前端不得自行假造 `/api/v1/admin/roles/:id/menus`，也不得用 `role.code` 硬編碼選單權限。
> `casbin_rule` 是後端授權 runtime policy table，由 Go 後端依 `group_form_permissions` 同步控制；前端不提供 `casbin_rule` 直接管理介面，也不得直接呼叫任何 Casbin policy CRUD。

### 頁面結構

```text
系統管理 > 群組管理                    ← 麵包屑

[🔍 搜尋群組名稱 / 代碼]  [狀態 ▼]  [+ 新增權限群組]

┌──────────────────────────────┬──────────────────────────────────────────────┐
│ 權限群組                       │ 群組詳情                                      │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ ● 值班工程師  engineers        │ 群組名稱         值班工程師                   │
│   成員 8  權限 3               │ 群組代碼         engineers                    │
│ ○ 只讀人員    viewers          │ 狀態             啟用                         │
│                                │ 成員數           8                            │
│                                │ 權限數           3                            │
│                                │                                              │
│                                │ [編輯群組] [管理成員] [停用] [刪除]            │
└──────────────────────────────┴──────────────────────────────────────────────┘

下方：選單 / 表單權限矩陣

┌────────────────────────────────────────────────────────────────────────────┐
│ 節點名稱        路徑             來源      讀取  新增  修改  刪除  子節點  操作 │
├────────────────────────────────────────────────────────────────────────────┤
│ Ticket          ticket          直接      ✓     -     -     -     套用    ✏ │
│ 建立 Ticket     ticket/create   繼承      ✓     ✓     ✓     -     -       ✏ │
└────────────────────────────────────────────────────────────────────────────┘
```

### 資訊架構

- 第一層是權限群組清單，對應 `groups`
- 第二層是目前群組詳情與操作，對應 `groups/:id`
- 第三層是目前群組對選單樹節點的權限矩陣，對應 `groups/:id/permissions`
- 選單節點來源固定使用管理端 `GET /api/v1/admin/forms`，只用於群組管理的權限矩陣；一般使用者側邊欄來源固定為 `GET /api/v1/forms/tree`
- 成員管理使用 `group_members`，不可改寫 `users.global_role`
- Casbin policy 不作為前端資訊架構層級；前端只顯示後端回傳的有效來源狀態，不直接管理 `casbin_rule`

### 權限矩陣欄位

| 欄位 | 來源 | 說明 |
| ---- | ---- | ---- |
| 節點名稱 | `form_name` / `form_nodes.name` | 顯示選單樹節點名稱 |
| 路徑 | `form_path` / `form_nodes.full_path` | Casbin 授權物件 |
| 來源 | `source` | `direct` 直接授權、`inherited` 繼承、`override` 覆寫 |
| 讀取 | `can_read` | 控制選單可見與查看 |
| 新增 | `can_create` | 控制建立資料 |
| 修改 | `can_update` | 控制更新資料 |
| 刪除 | `can_delete` | 控制刪除資料 |
| 子節點 | `inherit_children` | 是否套用至子節點 |
| 覆寫 | `override_parent` | 是否覆寫父節點繼承 |
| 操作 | - | 編輯權限、刪除直接授權 |

> `inherited` 權限列只可顯示來源，不可直接刪除；若要改變繼承結果，需在該節點新增 `override_parent = true` 的直接授權。

### 新增 / 編輯權限群組 Dialog

```text
┌── 新增權限群組 ──────────────────────────────┐
│                                          [✕] │
│  ┌── 群組名稱 * ──────────────────────────┐  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── 群組代碼 * ──────────────────────────┐  │  ← 英文小寫、數字、底線
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── 說明 ────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  狀態  ● 啟用  ○ 停用                        │
│                                              │
│                      [取消]  [儲存]          │
└──────────────────────────────────────────────┘
```

### 管理成員 Dialog

```text
┌── 管理群組成員：值班工程師 ───────────────────┐
│                                          [✕] │
│  [🔍 搜尋使用者]  [新增成員 ▼]                │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ 使用者      姓名       Email        操作 │  │
│  │ admin       系統管理員 admin@...    🗑  │  │
│  └────────────────────────────────────────┘  │
│                                              │
│                              [關閉]          │
└──────────────────────────────────────────────┘
```

- 新增成員成功後 invalidate `['admin', 'groups']` 與 `['admin', 'groups', id, 'members']`
- 移除成員需二次確認；若後端拒絕最後管理員或其他保護規則，直接顯示後端錯誤訊息
- 使用者搜尋需串真實使用者 API，不得使用本地假清單

### 編輯節點權限 Dialog

```text
┌── 編輯節點權限 ──────────────────────────────┐
│                                          [✕] │
│  群組       值班工程師                        │
│  節點       建立 Ticket                       │
│  路徑       ticket/create                     │
│  目前來源   繼承自 Ticket                     │
│                                              │
│  權限       ☑ 讀取   ☑ 新增   ☑ 修改   ☐ 刪除 │
│  ☑ 套用至子節點                              │
│  ☐ 覆寫父節點繼承                            │
│                                              │
│  異動摘要                                    │
│  讀取：繼承 → 直接授權                        │
│  新增：未授權 → 直接授權                      │
│                                              │
│                      [取消]  [儲存權限]      │
└──────────────────────────────────────────────┘
```

- 至少需勾選一個操作層級；完全取消權限時使用刪除直接授權流程
- 儲存前顯示異動摘要，內容來自目前直接授權與表單狀態差異
- 儲存成功後 invalidate `['admin', 'groups', id, 'permissions']`，並重新讀取有效來源
- 儲存失敗不得先行更新 UI 成功狀態；錯誤訊息優先使用後端回傳

### API 契約

```text
GET    /api/v1/admin/groups
POST   /api/v1/admin/groups
GET    /api/v1/admin/groups/:id
PUT    /api/v1/admin/groups/:id
DELETE /api/v1/admin/groups/:id

GET    /api/v1/admin/groups/:id/members
POST   /api/v1/admin/groups/:id/members
DELETE /api/v1/admin/groups/:id/members/:uid

GET    /api/v1/admin/groups/:id/permissions
POST   /api/v1/admin/groups/:id/permissions
DELETE /api/v1/admin/groups/:id/permissions/:pid

GET    /api/v1/admin/forms
GET    /api/v1/forms/tree
GET    /api/v1/forms/permissions?path=ticket/create
```

新增權限群組 body：

```json
{
  "code": "ops_admin",
  "name": "系統管理權限群組",
  "description": "OIDC ops_admin 對應群組"
}
```

編輯權限群組 body：

```json
{
  "name": "系統管理權限群組",
  "description": "OIDC ops_admin 對應群組"
}
```

啟用 / 停用權限群組 body：

```json
{
  "name": "系統管理權限群組",
  "description": "OIDC ops_admin 對應群組",
  "is_active": true
}
```

> 後端 request decoder 會拒絕未知欄位。前端新增時不得送 `is_active`，編輯時不得送 `code`；只有啟用 / 停用流程可送 `is_active`。

新增 / 更新群組權限 body：

```json
{
  "form_node_id": "01K...",
  "can_read": true,
  "can_create": true,
  "can_update": true,
  "can_delete": false,
  "inherit_children": true,
  "override_parent": false
}
```

### 權限與空狀態

- 僅 `global_role = admin` 可進入此分頁；非 Admin 由路由守衛導向無權限頁或回到 Dashboard。
- 無權限群組時顯示空狀態：「尚未建立權限群組」，並提供「新增權限群組」按鈕。
- 尚未選取群組時，權限矩陣顯示「請先選取權限群組」。
- API 回傳 403 時顯示「沒有權限群組管理權限」，不得顯示假資料或本地預設權限。
- 停用群組後，前端需顯示停用 Chip；後端不得再將該群組同步為有效 Casbin policy。

### i18n 規範

- namespace 使用 `admin`，key 前綴使用 `permission_groups.*`。
- 群組欄位、成員 Dialog、權限矩陣欄位、來源狀態、Toast、錯誤訊息、刪除確認與空資料提示全部放入 i18n。
- 權限矩陣若使用 Data Grid，必須使用共用 `getDataGridLocaleText`，欄位選單、排序、篩選、分頁、空資料與載入文字都需跟隨語系。

---

## 選單管理頁設計（`/admin/menus`）

> 選單管理畫面顯示為「選單樹」，實際管理需求 2 的表單節點資料（`form_nodes`）。前端入口為 `/admin/menus`，但實際 API 依後端設計串接管理端 `/api/v1/admin/forms`，不得自行假造 `/api/v1/admin/menus`。此頁只管理選單節點結構、顯示名稱、排序與啟用狀態；權限群組、群組成員與群組表單權限搬移至 `/admin/groups` 群組管理。一般使用者側邊欄不可使用 `/api/v1/admin/forms`，需使用 `/api/v1/forms/tree` 取得有效選單樹。

### 頁面結構

```text
系統管理 > 選單管理          ← 麵包屑

[🔍 搜尋節點名稱 / 路徑]  [節點類型 ▼]  [狀態 ▼]       [+ 新增根節點]

┌──────────────────────────────┬──────────────────────────────────────────────┐
│ 選單樹                         │ 節點詳情                                      │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ ▾ Ticket                       │ 名稱             Ticket                       │
│   └ 建立 Ticket                │ 路徑             ticket                       │
│ ▸ 排程管理                     │ 類型             root                         │
│ ▸ 報表                         │ 父節點           -                            │
│                                │ 排序             1                            │
│                                │ 狀態             啟用                         │
│                                │ 說明             Ticket 模組根節點             │
│                                │                                              │
│                                │ [新增子節點] [編輯] [停用] [刪除]              │
└──────────────────────────────┴──────────────────────────────────────────────┘

下方：節點子項 Data Grid

┌────────────────────────────────────────────────────────────────────────────┐
│ 名稱          路徑              類型       排序    狀態      更新時間    操作 │
├────────────────────────────────────────────────────────────────────────────┤
│ 建立 Ticket   ticket/create    form       1       啟用      2026/...   ✏ 🗑 │
└────────────────────────────────────────────────────────────────────────────┘
```

### 版面規格

- 頁面採用與用戶管理、角色管理一致的深色毛玻璃管理後台風格：上方篩選列、主要內容區、Data Grid。
- 左側選單樹寬度桌機 `360px`，右側節點詳情自適應；平板以下改為上下堆疊。
- 選單樹使用現有 MUI `List` + `Collapse` 組成樹狀列表，不新增 `@mui/x-tree-view`；節點列固定高度 `44px`，避免展開 / 收合造成文字擠壓。
- 節點選中狀態使用系統管理子選單相同的青藍色半透明背景，保持 Admin 區視覺一致。
- 節點詳情以欄位列呈現，不使用巢狀卡片；只在選單樹、詳情區、Data Grid 外層使用單層容器。
- 所有 Dialog、Popover、Select 選單不得被毛玻璃背景影響；彈層需使用實色深色底與足夠對比。

### 選單樹節點欄位

| 欄位 | 來源 | 說明 |
| ------ | ------ | ------ |
| ID | `id` | ULID，詳情中顯示，列表預設不顯示 |
| 父節點 | `parent_id` | 根節點為空；新增子節點時自動帶入目前選取節點 |
| 節點類型 | `node_type` | `root` / `category` / `form` |
| 節點 key | `form_key` | 父節點下唯一，例如 `create` |
| 完整路徑 | `full_path` | 後端計算，例如 `ticket/create` |
| 顯示名稱 | `name` | UI 顯示名稱，必填 |
| 說明 | `description` | 選填，詳情中顯示 |
| 排序 | `sort_order` | 同層排序，小到大 |
| 狀態 | `is_active` | 啟用 / 停用；停用節點不應出現在一般使用者可見選單樹 |
| 建立時間 | `created_at` | 詳情中顯示 |
| 更新時間 | `updated_at` | Data Grid 與詳情中顯示 |

### 篩選列

| 控制項 | 型態 | 行為 |
| ------ | ------ | ------ |
| 搜尋 | `TextField` | 對 `name`、`form_key`、`full_path` 前端過濾；Enter 或 debounce 300ms 套用 |
| 節點類型 | `Select` | 全部 / 根節點 / 分類 / 表單 |
| 狀態 | `Select` | 全部 / 啟用 / 停用 |
| 新增根節點 | `Button` | 開啟新增 Dialog，`parent_id = null` |

> `GET /api/v1/admin/forms` 若後端尚未提供搜尋參數，前端可在已載入的樹資料上做本地過濾；不得因此改用假資料。

### 節點操作

| 操作 | 條件 | UI |
| ------ | ------ | ------ |
| 新增子節點 | 已選取節點且節點啟用 | `AddIcon` + 文字按鈕 |
| 編輯 | 已選取節點 | `EditIcon` |
| 啟用 / 停用 | 已選取節點 | `ToggleOnIcon` / `ToggleOffIcon` |
| 刪除 | 已選取節點且無子節點 | `DeleteIcon`，刪除前確認 |
| 重新整理 | 任意狀態 | `RefreshIcon` |

- 刪除 API 回傳 409 時顯示後端訊息，例如「存在子節點」或「已有業務資料引用」。
- 編輯時若節點已有子節點或引用，`full_path` 不可由前端手動修改；前端只送允許修改的欄位。
- 新增、編輯、刪除、啟用 / 停用成功後需 invalidate `['admin', 'forms']` 並顯示 Toast。

### 新增 / 編輯 Dialog

```text
┌── 新增節點 ──────────────────────────────────┐
│                                          [✕] │
│  父節點  Ticket                              │  ← 新增子節點時顯示；根節點顯示「根目錄」
│                                              │
│  ┌── 顯示名稱 * ──────────────────────────┐  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── 節點 key * ──────────────────────────┐  │  ← 英文小寫、數字、底線、連字號
│  └────────────────────────────────────────┘  │
│  完整路徑將由後端依父節點計算                  │
│                                              │
│  類型  [分類 ▼]        排序  [ 10 ]           │
│                                              │
│  ┌── 說明 ────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  狀態  ● 啟用  ○ 停用                        │
│                                              │
│                      [取消]  [✓ 儲存]        │
└──────────────────────────────────────────────┘
```

```typescript
const formNodeSchema = z.object({
  parent_id:   z.string().optional().nullable(),
  node_type:   z.enum(['root', 'category', 'form']),
  form_key:    z.string().regex(/^[a-z][a-z0-9_-]*$/, t('admin.menus.error.invalid_key')),
  name:        z.string().min(1).max(128),
  description: z.string().max(1000).optional(),
  sort_order:  z.number().int().min(0),
  is_active:   z.boolean(),
});
```

### 刪除確認 Dialog

```text
┌── 刪除節點 ──────────────────────────────────┐
│                                          [✕] │
│  確定要刪除節點「建立 Ticket」嗎？            │
│  刪除後將同步移除相關選單樹設定；若節點仍有    │
│  子節點或業務資料引用，後端會拒絕刪除。        │
│                                              │
│                      [取消]  [刪除]          │
└──────────────────────────────────────────────┘
```

### API 契約

```text
GET    /api/v1/admin/forms
GET    /api/v1/admin/forms/:id
POST   /api/v1/admin/forms
PUT    /api/v1/admin/forms/:id
DELETE /api/v1/admin/forms/:id
```

新增 body：

```json
{
  "parent_id": "01K...",
  "node_type": "form",
  "form_key": "create",
  "name": "建立 Ticket",
  "description": "Ticket 建立表單",
  "sort_order": 1,
  "is_active": true
}
```

更新 body：

```json
{
  "node_type": "form",
  "name": "建立 Ticket",
  "description": "Ticket 建立表單",
  "sort_order": 1,
  "is_active": true
}
```

### i18n 規範

- namespace 使用 `admin`，key 前綴使用 `menus.*`。
- 欄位標題、節點類型、狀態、Dialog 文案、Toast、錯誤訊息、刪除確認與空資料提示全部放入 i18n。
- 子項 Data Grid 必須使用共用 `getDataGridLocaleText`，欄位選單、排序、篩選、分頁、空資料與載入文字都需跟隨語系。

### 權限與空狀態

- 僅 `global_role = admin` 可進入 `/admin/menus`；非 Admin 由路由守衛導向無權限頁或回到 Dashboard。
- API 回傳 403 時顯示「沒有選單管理權限」Toast，頁面保留空狀態，不顯示假資料。
- 無節點時顯示空狀態：「尚未建立選單樹節點」，並提供「新增根節點」按鈕。

---

## 專案管理頁設計（`/admin/projects`）

> 前端管理入口為 `/admin/projects`，但目前 server 已實作的專案 API 掛在 `/api/v1/projects`，不是 `/api/v1/admin/projects`。前端需依 server 契約串接，不得自行假造 `owner`、`is_active`、`sub_project_count` 等後端未回傳欄位。

### 頁面結構

```text
系統管理 > 專案管理          ← 麵包屑

[🔍 搜尋專案名稱 / 代碼]  [+ 新增專案]

┌────────────────────────────────────────────────────────────────────────────┐
│ ID  專案名稱        專案代碼      可見性      狀態       建立者    備註  操作 │
├────────────────────────────────────────────────────────────────────────────┤
│ 01K... OPS 平台維運 OPS       private   active     01K...  ...  ✏ 🗑 │
│ 01K... 客服系統     SUPPORT   public    inactive   01K...  ...  ✏ 🗑 │
└────────────────────────────────────────────────────────────────────────────┘

Page Size: [10▼]    1 to 10 of 20    |< < Page 1 of 2 > >|
```

### 欄位定義

| 欄位 | 寬度 | 說明 |
| ------ | ------ | ------ |
| ID | 100 | `id`，ULID，超長截斷 + `Tooltip` |
| 專案名稱 | 180 | `name`，超長截斷 + `Tooltip` |
| 專案代碼 | 120 | `key`，全系統唯一；server 會正規化為大寫 |
| 可見性 | 100 | `visibility`：`public` / `private` Chip |
| 狀態 | 100 | `status`：`active`（綠色）/ `inactive`（灰色）/ `archived`（橘色）Chip |
| 建立者 | 120 | `created_by`，目前 server 回傳使用者 ID |
| 備註 | 220 | `description`，超長截斷 + `Tooltip` |
| 操作 | 96 | 編輯 `EditIcon`（藍色）+ 刪除 `DeleteIcon`（紅色） |

### 搜尋與篩選

```typescript
interface ProjectSearchParams {
  keyword?: string;      // 對 key / name 模糊搜尋
}
```

- 搜尋框使用 `TextField size="small"`，左側 `SearchIcon`。
- Enter 鍵或搜尋按鈕觸發查詢；查詢時重設頁碼為第 1 頁。
- 目前 server `GET /api/v1/projects` 僅支援 `keyword`、`limit`、`offset`；前端不得新增未支援的 `status` 篩選並假裝生效。
- Data Grid 需使用共用 `getDataGridLocaleText`，欄位選單、排序、篩選、分頁與空資料文字皆跟隨 i18n。

### 新增按鈕

```typescript
<Button
  variant="contained"
  startIcon={<AddIcon />}
  sx={{
    background: 'linear-gradient(90deg, #5b8dee 0%, #42a5f5 100%)',
    borderRadius: 2,
  }}
>
  {t('projects.add_project')}
</Button>
```

### 新增 / 編輯 Dialog

```text
┌── 新增專案 ──────────────────────────────────┐
│                                          [✕] │
│  ┌── 專案名稱 * ──────────────────────────┐  │
│  └────────────────────────────────────────┘  │
│  ┌── 專案代碼 * ──────────────────────────┐  │  ← 新增可編輯，編輯唯讀
│  └────────────────────────────────────────┘  │
│  專案代碼會由後端正規化為大寫                  │
│  可見性 [private ▼]    狀態 [active ▼]        │
│  ┌── 備註 ────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│                      [取消]  [✓ 儲存]        │
└──────────────────────────────────────────────┘
```

```typescript
const projectSchema = z.object({
  key:         z.string().trim().regex(/^[A-Za-z0-9_-]{2,32}$/),
  name:        z.string().trim().min(1).max(128),
  description: z.string().optional(),
  visibility:  z.enum(['public', 'private']),
  status:      z.enum(['active', 'inactive', 'archived']),
});
```

### 子專案顯示邊界

子專案管理頁面放在 Ticket / 專案工作區脈絡中，不放在 Admin 專案管理頁。`/admin/projects` 只管理主專案基本資料與啟用狀態，不提供子專案 CRUD 入口。

Ticket / 專案工作區後續實作子專案管理時，需使用現有 server API：

```text
GET    /api/v1/projects/:id/sub-projects
POST   /api/v1/projects/:id/sub-projects
GET    /api/v1/sub-projects/:sid
PUT    /api/v1/sub-projects/:sid
DELETE /api/v1/sub-projects/:sid
```

`/admin/projects` 列表不顯示 `sub_project_count`，因為子專案管理不屬於此頁，且目前 `GET /api/v1/projects` item contract 未回傳該欄位。

子專案欄位契約：

```typescript
interface SubProjectItem {
  id: string;
  project_id: string;
  key: string;
  name: string;
  description?: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

### 刪除確認 Dialog

```text
┌── 刪除專案 ──────────────────────────────────┐
│                                          [✕] │
│  確定要刪除專案「OPS 平台維運」嗎？           │
│  若後端拒絕刪除，前端需顯示錯誤 Toast。       │
│                                              │
│                      [取消]  [刪除]          │
└──────────────────────────────────────────────┘
```

- 刪除按鈕使用 `color="error"`。
- 刪除成功後需重新整理 `GET /api/v1/projects` 列表。
- 若後端回傳錯誤，前端以錯誤 Toast 顯示原因，不得先行從列表移除資料。

### 後端 API

```text
GET    /api/v1/projects          — 列表（分頁 + 關鍵字）
POST   /api/v1/projects          — 新增
GET    /api/v1/projects/:id      — 單筆
PUT    /api/v1/projects/:id      — 更新
DELETE /api/v1/projects/:id      — 刪除
```

### API item contract

```typescript
interface AdminProjectItem {
  id: string;
  key: string;
  name: string;
  description?: string;
  visibility: 'public' | 'private';
  status: 'active' | 'inactive' | 'archived';
  created_by: string;
  created_at: string;
  updated_at: string;
}
```

### 專案成員管理邊界

專案成員管理放在 Ticket / 專案工作區脈絡中，不放在 Admin 專案管理頁。`/admin/projects` 不顯示成員數，也不提供成員新增、編輯或移除入口。

Ticket / 專案工作區後續實作專案成員管理時，需使用現有 server API：

```text
GET    /api/v1/projects/:id/members
POST   /api/v1/projects/:id/members
PUT    /api/v1/projects/:id/members/:uid
DELETE /api/v1/projects/:id/members/:uid
```

成員 role 使用 server 既有值：`project_manager`、`engineer`、`viewer`。

---

#### 頁面結構

```text
系統管理 > 用戶管理          ← 麵包屑

[🔍 搜尋...]  [+ 新增]       ← 搜尋框 + 新增按鈕

┌──────────────────────────────────────────────────────────────────┐
│ ID  用戶名  真實姓名  帳號  郵箱  手機  狀態  MFA狀態  備註  操作 │
├──────────────────────────────────────────────────────────────────┤
│ 6   55566666  116666...  55566666  aaa@...  116...  啟用    ✏ 🗑 │
│ 1   admin     系統管理員  admin    admin@...  138...  啟用   ✏ 🗑(灰) │
└──────────────────────────────────────────────────────────────────┘

Page Size: [10▼]    1 to 2 of 2    |< < Page 1 of 1 > >|
```

#### 欄位定義

| 欄位 | 寬度 | 說明 |
| ------ | ------ | ------ |
| ID | 60 | 自動遞增序號 |
| 用戶名 | 120 | `username` |
| 真實姓名 | 140 | `full_name`，超長截斷 + `Tooltip` |
| 帳號 | 120 | 同 `username`（顯示登入帳號） |
| 郵箱 | 180 | `email` |
| 手機 | 130 | `phone` |
| 狀態 | 80 | `is_active`：啟用（綠色 Chip）/ 停用（灰色 Chip） |
| MFA 狀態 | 110 | `has_mfa`：已啟用（綠色 Chip）/ 未啟用（灰色 Chip）；後端僅以已驗證且啟用中的 MFA 裝置判斷 |
| 備註 | 150 | `remark`（選填，超長截斷） |
| 操作 | 80 | 編輯 `EditIcon`（藍色）+ 刪除 `DeleteIcon`（紅色） |

> 自身帳號與 `global_role = 'admin'` 的最後一個 admin 帳號，刪除按鈕灰色禁用（`disabled`），hover 顯示 `Tooltip` 說明原因。

### 搜尋列

```typescript
// 單一搜尋框，對 username / full_name / email 模糊搜尋
<TextField
  size="small"
  placeholder={t('action.search')}
  InputProps={{ startAdornment: <SearchIcon /> }}
  onChange={(e) => setKeyword(e.target.value)}
/>
```

Enter 鍵或 debounce 300ms 觸發搜尋。

### 新增按鈕

```typescript
<Button
  variant="contained"
  startIcon={<AddIcon />}
  sx={{
    background: 'linear-gradient(90deg, #5b8dee 0%, #42a5f5 100%)',
    borderRadius: 2,
  }}
  onClick={() => setDialogOpen(true)}
>
  {t('action.add')}
</Button>
```

### 新增 / 編輯 Dialog

```text
┌── 新增用戶 ──────────────────────────────────┐
│                                          [✕] │
│  ┌── 用戶名 ──────────────────────────────┐  │  ← 必填
│  └────────────────────────────────────────┘  │
│  ┌── 真實姓名 ────────────────────────────┐  │  ← 必填
│  └────────────────────────────────────────┘  │
│  ┌── 電子郵箱 ────────────────────────────┐  │  ← 必填
│  └────────────────────────────────────────┘  │
│  ┌── 手機 ────────────────────────────────┐  │  ← 選填
│  └────────────────────────────────────────┘  │
│  ┌── 密碼 ────────────────────────────────┐  │  ← 新增必填；編輯選填（空白=不修改）
│  └────────────────────────────────────────┘  │
│  班別  [早班 ▼]    全域角色  [member ▼]       │
│  狀態  ● 啟用  ○ 停用                        │
│  ┌── 備註 ────────────────────────────────┐  │  ← 選填，multiline
│  └────────────────────────────────────────┘  │
│                                              │
│                      [取消]  [✓ 儲存]        │
└──────────────────────────────────────────────┘
```

```typescript
const userSchema = z.object({
  username:    z.string().min(1).max(64),
  full_name:   z.string().min(1).max(128),
  email:       z.string().email(),
  phone:       z.string().optional(),
  password:    z.string().min(8).optional(),  // 編輯時選填
  shift:       z.enum(['morning', 'afternoon', 'night']),
  global_role: z.enum(['admin', 'op_admin', 'member']),
  is_active:   z.boolean(),
  remark:      z.string().optional(),
});
```

班別 `Select` 選項：早班（morning）/ 中班（afternoon）/ 晚班（night）。
全域角色 `Select` 選項：管理員（admin）/ 成員（member）。

### 刪除確認 Dialog

```text
┌── 刪除用戶 ──────────────────────────────────┐
│                                          [✕] │
│  確定要刪除用戶「55566666」嗎？              │
│  此操作無法復原。                            │
│                                              │
│                      [取消]  [刪除]          │  ← 刪除 color="error"
└──────────────────────────────────────────────┘
```

### 狀態 Chip

```typescript
<Chip
  label={user.is_active ? t('status.active') : t('status.inactive')}
  color={user.is_active ? 'success' : 'default'}
  size="small"
  variant="outlined"
/>
```

---

### 頁面結構

```
系統管理 > 日誌查詢          ← 麵包屑 Breadcrumbs                [⬇ 匯出]

┌─ 操作日誌 ──┬─ 登入日誌 ─┐   ← Tabs
│             │            │
└─────────────┴────────────┘

[🔍 搜尋用戶名] [搜尋模組] [搜尋 IP]  [🔍 搜尋]  [🔄 刷新]

┌────────────────────────────────────────────────────────────────────┐
│ ID▼ 用戶名▼ 模組▼ 操作▼ 方法▼ 路徑▼ IP▼ HTTP▼ 狀態▼ 耗時▼ 時間▼ 操作▼ │
├────────────────────────────────────────────────────────────────────┤
│ 23  admin  scheduler 修改 PUT /api/v1/... ::1 [200] [成功] 2ms ... 👁 │
│ 22  admin  scheduler 修改 PUT /api/v1/... ::1 [200] [成功] 2ms ... 👁 │
│ ...                                                                │
└────────────────────────────────────────────────────────────────────┘

Page Size: [20▼]    1 to 20 of 26    |< < Page 1 of 2 > >|
```

### Tab 定義

| Tab | 資料來源 | 說明 |
|-----|---------|------|
| Ticket 活動 | `ticket_activities` | Ticket 操作、處理明細、留言與欄位異動 |
| 登入日誌 | 獨立 `login_logs` 表（擴充） | 登入成功 / 失敗、IP、裝置 |

> 後端契約：操作日誌列表欄位必須由 `GET /api/v1/admin/logs/activity` 直接回傳，不得要求前端從 `detail` JSON 推測。若後端尚未完成 schema / API 補強，前端只能顯示 `-`，不得偽造 HTTP 狀態、結果或耗時。

### 搜尋列

```typescript
interface LogSearchParams {
  username?: string;
  module?: string;
  ip?: string;
  // 操作日誌額外篩選
  method?: string;
  status_code?: number;
  // 登入日誌額外篩選
  result?: 'success' | 'failed';
}
```

三個 `TextField`（`size="small"`）+ 搜尋 `Button`（漸層藍）+ 刷新 `IconButton`（`RefreshIcon`）。Enter 鍵觸發搜尋。

### 操作日誌欄位定義

| 欄位 | 寬度 | 說明 |
|------|------|------|
| ID | 60 | 自動遞增序號 |
| 用戶名 | 100 | `username` |
| 模組 | 100 | 對應 BC（scheduler / ticket / auth...） |
| 操作 | 80 | 新增 / 修改 / 刪除 / 查詢 |
| 方法 | 70 | HTTP method（GET/POST/PUT/DELETE） |
| 路徑 | 200 | API path，超長截斷 + `Tooltip` 顯示完整 |
| IP | 100 | 來源 IP |
| HTTP | 70 | 狀態碼，`200` 綠色 / `4xx` 橘色 / `5xx` 紅色 Chip |
| 狀態 | 80 | 成功（綠色）/ 失敗（紅色）Chip |
| 耗時 | 70 | `2ms`、`28ms` |
| 時間 | 150 | `yyyy/M/d 上午/下午 h:mm:ss` |
| 操作 | 60 | `👁 IconButton`，點擊開啟詳情 Dialog |

欄位來源：`username`、`method`、`path`、`status_code`、`result`、`duration_ms` 必須使用後端 response 的一等欄位；`detail` 僅用於詳情 Dialog 與補充資訊，不作為列表欄位的唯一來源。

每欄標題右側有 `FilterListIcon`（`size="small"`），點擊展開欄位篩選 Popover。

### HTTP 狀態碼 Chip 顏色

```typescript
function getStatusChipColor(code: number): 'success' | 'warning' | 'error' | 'default' {
  if (code >= 200 && code < 300) return 'success';   // 綠
  if (code >= 400 && code < 500) return 'warning';   // 橘
  if (code >= 500)               return 'error';     // 紅
  return 'default';
}
```

### 詳情 Dialog

點擊眼睛 icon 展開：

```
┌── 操作日誌詳情 ──────────────────────────────┐
│                                          [✕] │
│  用戶名：admin                               │
│  模組：scheduler                             │
│  操作：修改                                  │
│  方法：PUT                                   │
│  路徑：/api/v1/scheduler/task/5              │
│  IP：::1                                     │
│  HTTP 狀態：200                              │
│  狀態：成功                                  │
│  耗時：2ms                                   │
│  時間：2025/12/4 下午 5:28:00                │
│                                              │
│  Request Body：                              │
│  ┌──────────────────────────────────────┐   │
│  │ { "status": "active" }               │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  Response Body：                             │
│  ┌──────────────────────────────────────┐   │
│  │ { "code": 0, "message": "ok" }       │   │
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

Request / Response Body 使用 `<pre>` + `JSON.stringify(data, null, 2)` 格式化顯示，背景 `rgba(0,0,0,0.3)`，圓角，可捲動。

### 分頁元件

底部分頁列：左側 Page Size `Select`（10 / 20 / 50 / 100）、中間「1 to 20 of 26」文字、右側 `|< < Page N of M > >|` 按鈕組。使用 MUI `TablePagination`，`rowsPerPageOptions={[10, 20, 50, 100]}`。

### 匯出按鈕

```typescript
// 右上角，匯出當前篩選結果為 CSV
<Button variant="outlined" startIcon={<DownloadIcon />} onClick={handleExport}>
  {t('action.export')}
</Button>
// GET /api/v1/admin/logs/export?type=activity&...filters
```

### 登入日誌欄位定義

| 欄位 | 說明 |
|------|------|
| ID | 序號 |
| 用戶名 | 登入帳號 |
| IP | 來源 IP |
| 裝置 | User-Agent 解析（瀏覽器 / OS） |
| 結果 | 成功（綠）/ 失敗（紅）Chip |
| 失敗原因 | 密碼錯誤 / 帳號停用 / MFA 失敗 等 |
| 時間 | 登入時間 |

### 後端 API 擴充

```
GET /api/v1/admin/logs/activity   — 操作日誌列表（分頁 + 篩選）
GET /api/v1/admin/logs/login      — 登入日誌列表（分頁 + 篩選）
GET /api/v1/admin/logs/export     — 匯出 CSV（?type=activity|login）
```

`GET /api/v1/admin/logs/activity` item contract：

```typescript
interface ActivityLogItem {
  id: string;
  actor_id?: string;
  username: string;
  module: string;
  action: string;
  target_type: string;
  target_id: string;
  method: string;
  path: string;
  ip_address: string;
  status_code?: number;
  result: 'success' | 'failed';
  duration_ms?: number;
  detail: Record<string, unknown>;
  created_at: string;
}
```

### 值班統計矩陣顯示規則

`DailyShiftExecutionMatrix` 在顯示 `alert_notification`、`domain_change` 或 `payment_domain_change` 群組時，需在第一個日期欄位前插入「總計」欄。總計由前端依後端回傳的 `matrix.columns` 順序，將每列 `values[column.date]` 加總；缺少的日期值視為 `0`。指標與人員列使用相同規則。

上述三種群組不渲染後端回傳的 `row_type = shift` 班別彙總列，因此畫面不顯示獨立的早班、中班、晚班三列；`row_type = person` 的「班別-人員」列仍需保留，讓使用者辨識每位人員所屬班別。

此欄與班別列隱藏皆為既有矩陣資料的前端呈現調整，不新增或變更後端欄位；`jira_notification` 維持原有日期矩陣與班別彙總列，不顯示總計欄。

報表模式、空資料提示及三種指定群組的標題統一使用「值班統計」，不顯示「每日值班統計」或「每日值班執行統計」。內部 `daily_shift_execution` 識別值與 API 路徑維持不變。

報表控制列的「報表模式」只使用單一 `E` 選項顯示「值班統計」，不再把 `E` 與群組組合成多個選項。當 `reportMode === 'E'` 時啟用「數值」欄位，選項為 `alert_notification`、`domain_change`、`payment_domain_change`；選取結果寫入既有 `dailyShiftMetricGroup` 狀態，預覽仍以 `metric_groups` 呼叫值班統計 API，匯出與範本仍使用既有 `indicators` 格式。

---

## 全域設定頁設計（`/admin/system`）

### 列表頁結構

```
系統管理 > 全域設定          ← 麵包屑

[分類 ▼]                     ← 分類篩選 Select（通用/系統/郵件/安全/儲存/介面）

[🔍 搜尋...]  [+ 新增]

┌──────────────────────────────────────────────────────────────────┐
│ 設定鍵名           設定值        分類  值類型    描述        操作  │
├──────────────────────────────────────────────────────────────────┤
│ general.site_name  Admin         通用  [string]  網站名稱    ✏ 🗑 │
│ general.site_logo               通用  [string]  網站Logo URL ✏ 🗑 │
│ security.mfa.force_enable  🟢   安全  [boolean] 是否強制MFA  ✏ 🗑 │
│ security.mfa.allowed_types [...] 安全 [json]    允許的MFA類型 ✏ 🗑 │
│ system.log.ticket_activity_keep_days 0 系統 [number] Ticket Activity Log 保留天數 ✏ 🗑 │
│ ...                                                              │
└──────────────────────────────────────────────────────────────────┘

Page Size: [10▼]    1 to 10 of 10    |< < Page 1 of 1 > >|
```

### 分類定義

| 分類 key | 顯示名稱 | 說明 |
|---------|---------|------|
| `general` | 通用 | 網站名稱、Logo 等 |
| `system` | 系統 | 日誌保留天數、排程設定等 |
| `mail` | 郵件 | SMTP 設定 |
| `security` | 安全 | MFA、密碼、Session 等 |
| `storage` | 儲存 | 附件儲存後端設定 |
| `ui` | 介面 | Sidebar 預設狀態等 UI 偏好 |

### 值類型 Chip 顏色

```typescript
const valueTypeColor: Record<string, string> = {
  string:  '#42a5f5',  // 藍
  number:  '#ab47bc',  // 紫
  boolean: '#26a69a',  // 青綠
  json:    '#ff7043',  // 橘紅
};

// <Chip label={type} size="small" sx={{ backgroundColor: valueTypeColor[type], color: '#fff' }} />
```

### 設定值顯示規則

```typescript
function renderSettingValue(value: string, type: string): React.ReactNode {
  if (type === 'boolean') {
    const isTrue = value === 'true';
    // 綠色小圓點（true）或灰色（false）
    return <Box sx={{ width: 12, height: 12, borderRadius: '50%', backgroundColor: isTrue ? '#4caf50' : '#757575' }} />;
  }
  if (type === 'json') {
    // 截斷顯示，hover Tooltip 顯示完整 JSON
    return <Typography variant="body2" noWrap sx={{ maxWidth: 200 }}>{value}</Typography>;
  }
  return value || '—';
}
```

### 預設全域設定項目

```typescript
// system_settings 初始化種子資料（擴充）
const defaultSettings = [
  // 通用
  { key: 'general.site_name',               value: 'OnCall Ticket System', category: 'general',  type: 'string',  sort: 1 },
  { key: 'general.site_logo',               value: '',                     category: 'general',  type: 'string',  sort: 2 },
  // 安全
  { key: 'security.mfa.force_enable',       value: 'false',                category: 'security', type: 'boolean', sort: 1 },
  { key: 'security.mfa.allowed_types',      value: '["totp","email","sms"]', category: 'security', type: 'json', sort: 2 },
  { key: 'security.login.password_length',  value: '4',                    category: 'security', type: 'number',  sort: 3 },
  { key: 'security.session.timeout',        value: '3200',                 category: 'security', type: 'number',  sort: 4 },
  // 系統
  { key: 'system.log.ticket_activity_keep_days', value: '0',               category: 'system',   type: 'number',  sort: 1 },
  { key: 'system.log.security_audit_keep_days',  value: '180',             category: 'system',   type: 'number',  sort: 2 },
  { key: 'system.log.system_audit_keep_days',    value: '180',             category: 'system',   type: 'number',  sort: 3 },
  { key: 'system.log.login_keep_days',           value: '180',             category: 'system',   type: 'number',  sort: 4 },
  { key: 'system.log.scheduler_keep_days',       value: '180',             category: 'system',   type: 'number',  sort: 5 },
  // 介面
  { key: 'ui.sidebar.collapsed',            value: 'false',                category: 'ui',       type: 'boolean', sort: 1 },
];
```

### 編輯頁（`/admin/system/:key/edit`）

獨立頁面（非 Dialog），標題「編輯設定」：

```
┌── 設定鍵名 * ──────────────────────────────────────────────────┐
│  general.site_logo                                            │  ← 編輯時唯讀（disabled）
└────────────────────────────────────────────────────────────────┘
鍵名建立後不能修改                                               ← helperText

┌── 設定值 ──────────────────────────────────────────────────────┐
│                                                               │
└────────────────────────────────────────────────────────────────┘

[分類 ▼ 通用]              [值類型 ▼ string]   ← 並排兩個 Select

┌── 描述 ────────────────────────────────────────────────────────┐
│  網站 Logo URL                                                │  ← multiline, rows=3
└────────────────────────────────────────────────────────────────┘

[排序  2  ]    🟢 狀態                          ← 排序數字輸入 + Switch

                                  [💾 儲存]  [✕ 取消]
```

```typescript
// 分類 Select 選項
const categoryOptions = [
  { value: 'general',  label: t('setting.category.general') },
  { value: 'system',   label: t('setting.category.system') },
  { value: 'mail',     label: t('setting.category.mail') },
  { value: 'security', label: t('setting.category.security') },
  { value: 'storage',  label: t('setting.category.storage') },
  { value: 'ui',       label: t('setting.category.ui') },
];

// 值類型 Select 選項
const valueTypeOptions = [
  { value: 'string',  label: 'string' },
  { value: 'number',  label: 'number' },
  { value: 'boolean', label: 'boolean' },
  { value: 'json',    label: 'json' },
];
```

**設定值輸入框依值類型動態切換**：
- `string`：普通 `TextField`
- `number`：`TextField` type="number"
- `boolean`：`Switch`（不顯示文字輸入框）
- `json`：`TextField` multiline，失焦時驗證 JSON 格式（`JSON.parse`）

**儲存 / 取消按鈕**：

```typescript
// 儲存：藍色漸層 + SaveIcon
<Button variant="contained" startIcon={<SaveIcon />} sx={saveButtonSx}>
  {t('action.save')}
</Button>

// 取消：灰色 outlined + CloseIcon
<Button variant="outlined" startIcon={<CloseIcon />} color="inherit" onClick={handleCancel}>
  {t('action.cancel')}
</Button>
```

### 新增頁（`/admin/system/new`）

與編輯頁相同佈局，差異：
- 標題「新增設定」
- 設定鍵名可編輯
- 設定值、描述、排序為空

### 後端 DB 擴充

`system_settings` 表新增欄位：

```sql
ALTER TABLE system_settings
  ADD COLUMN category    VARCHAR(32) NOT NULL DEFAULT 'general',
  ADD COLUMN value_type  VARCHAR(16) NOT NULL DEFAULT 'string'
                         CHECK (value_type IN ('string','number','boolean','json')),
  ADD COLUMN description TEXT,
  ADD COLUMN sort_order  INT NOT NULL DEFAULT 0,
  ADD COLUMN is_active   BOOLEAN NOT NULL DEFAULT TRUE;

CREATE INDEX ON system_settings (category);
```

### 後端 API 擴充

```
GET    /api/v1/admin/global-settings          — 列表（分頁 + 分類篩選 + 關鍵字）
POST   /api/v1/admin/global-settings          — 新增
GET    /api/v1/admin/global-settings/:key     — 單筆
PUT    /api/v1/admin/global-settings/:key     — 更新
DELETE /api/v1/admin/global-settings/:key     — 刪除
```

---

---

## 無障礙（Accessibility）

- 互動元素使用語意化 HTML（`button`、`input`、`select`）
- 表單元素關聯 `label`
- 狀態 Badge 加入 `aria-label`
- 鍵盤可操作（Tab 順序、Enter 觸發）
- 色彩對比符合 WCAG 2.1 AA
