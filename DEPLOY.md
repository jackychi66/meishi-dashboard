# DEPLOY — 梅侍工廠梅酒庫存與出貨管理後台

## 系統簡介

梅侍（Plummate）工廠端庫存與出貨的內部管理後台，給奧提瑪同仁查詢長盛酒廠的梅酒庫存水位、每月出貨與叫貨紀錄、以及叫貨周轉分析。需登入且經管理員核准才能使用。

## 線上位置

- 網址：https://meishi-dashboard.vercel.app
- Repo：https://github.com/jackychi66/meishi-dashboard （main 分支）
- 平台：Vercel（靜態站，自動部署自 GitHub main）＋ Supabase（會員登入）
- GitHub Pages 舊入口（同一份檔案，可停用）：https://jackychi66.github.io/meishi-dashboard

### 頁面結構

| 檔案 | 用途 |
|---|---|
| `index.html` | 登入頁（梅侍官方 logo，副標「工廠梅酒庫存與出貨管理後台」） |
| `apply.html` | 新會員申請（Supabase signUp，需管理員核准） |
| `admin.html` | 會員管理（僅 admin：核准／停用／移除） |
| `dashboard.html` | 主儀表板，5 tabs：庫存總覽／每月出貨／出貨明細／叫貨紀錄／周轉分析 |
| `config.js` | Supabase URL＋publishable key |
| `style.css` | 共用樣式 |
| `.github/workflows/keepalive.yml` | Supabase 保活（每日 ping REST＋Auth） |
| `CHANGELOG.md` | 版本更新記錄（**任何異動都要補一筆**） |

## 部署流程（照做就能上線）

### A. 只改頁面文字或功能

1. 修改本機 `dashboard.html`（或其他頁面檔）。
2. 驗證 JS 語法：有 shell 用 `node --check`（先抽出最後一個 `<script>` 區塊）；沒 shell 就先推到新分支，在 meishi-dashboard.vercel.app（無 CSP）的分頁用 `new Function()` 驗證。⚠️ 不要在 github.com 上做，CSP 會擋。
3. 上傳到 https://github.com/jackychi66/meishi-dashboard/upload/main → 選「Commit directly to the main branch」→ Commit changes。
   - **先確認瀏覽器登入的是 `jackychi66`**（`document.querySelector('meta[name="user-login"]').content`）。登入別的帳號會顯示「Uploads are disabled」。
   - 若 `file_upload` 工具被路徑權限擋住，改用瀏覽器就地改寫：fetch raw main → 正則換掉四個常數 → `new File()` + `DataTransfer` 注入 `input[type=file]`。細節見 skill `meishi-inventory-backend`。
4. **同一次更新 `CHANGELOG.md`**。
5. 等 30–60 秒，Vercel 自動部署。

### B. 更新庫存／出貨數據

1. 取雲端資料（見下方「資料來源」）：庫存流水帳存成 `rows.json`、生產量存成 `prod.json`。
2. **模板一律從線上版反推**（不要用舊模板，會蓋掉新功能）：
   ```bash
   curl -s https://raw.githubusercontent.com/jackychi66/meishi-dashboard/main/dashboard.html -o live.html
   # 用 python 把 PROD_SNAP / LSTAT_SNAP / LEV_SNAP / FMAP 四個常數換成 __PROD__ 等佔位符，
   # 並把 BUILD_TS 換成當下時間，輸出 template.html
   ```
3. `python3 build_dashboard.py --ledger-rows rows.json --prod prod.json --template template.html --out dashboard.html`
   → 必須看到「✓ 全部 N 項零誤差」才可部署（驗證：期初＋2026叫貨−2026出貨 ＝ 該品項最新庫存）。
   → 若不符，先判斷是解析錯誤還是**雲端結存列自身漏更新**。後者以計算值覆寫 `cur`，並通知長盛。
4. 依 A 的步驟 3–5 上傳部署。

### 驗證部署完成

⚠️ 不要只看 raw.githubusercontent（CDN 快取會騙人）。用這兩個：

```js
// A. commit 是否進 main
await fetch('https://api.github.com/repos/jackychi66/meishi-dashboard/commits?path=dashboard.html&per_page=1').then(r=>r.json());
// B. 在 meishi-dashboard.vercel.app 分頁內同源 fetch（跨網域會 CORS 失敗）
const live = await fetch('/dashboard.html?t='+Date.now()).then(r=>r.text());
const st = JSON.parse(live.match(/const LSTAT_SNAP=(\{[\s\S]*?\});\n/)[1]);
[(live.match(/const BUILD_TS="([^"]*)"/)||[])[1], st.date,
 st.rows.reduce((a,r)=>a+r.cur,0),
 st.rows.filter(r=>r.open+r.o26-r.x26!==r.cur).length];   // 最後一項應為 0
```

## 環境變數與金鑰

- **Supabase URL 與 publishable key**：明碼寫在 repo 的 `config.js`（publishable key 設計上可公開，安全性靠 RLS）。若在 Supabase 後台輪替金鑰，需同步更新 config.js 並重新部署。
- **service_role / secret key**：本系統前端完全不使用，切勿寫入 repo。
- **管理員帳號密碼**：由使用者本人保管，不記錄於任何檔案。

## 資料來源與更新方式

| 來源 | Spreadsheet ID | 分頁 | 用途 |
|---|---|---|---|
| 長盛｜在庫與生產統計表 2026 | `1c2Ibj5KdqYdoO4IWcfCi4W8bmvIN-4rq` | 梅酒庫存（gid 1579110842） | **庫存與出貨的唯一來源** |
| 2026長盛梅酒叫貨量及對帳表 | `1pl-uAwzbakxqtHHGcyFDPHhnoaIT_iFu4reDeggSx0I` | 梅酒生產量（gid 1059917080） | 叫貨（生產）紀錄 |
| 2025長盛梅酒叫貨量及對帳表 | `106fNleVmjBo3X5mNDHGp4PbDPWNrk_M1Lem6Aid5wSs` | 8–12月長盛請款 | 2025 請款出貨（參考） |

- 更新頻率：不固定，長盛登錄新的進出或下單時。
- **抓取方式（2026-09 起改為 Google Drive 連接器，不需 Chrome）**：兩份表都已分享給 66jacky@gmail.com。
  - 在庫與生產統計表（.xlsx）：`read_file_content`
  - 叫貨量及對帳表（原生 Google Sheet）：**必須用 `download_file_content` + `exportMimeType: text/csv`**
  - ⚠️ 對帳表**不可用** `read_file_content`——文字轉換會靜默漏掉 4 月之後全部的下單列。
  - 備援：Claude in Chrome 於 docs.google.com 分頁內同源 `fetch`。
- ⚠️ 只存在於流水帳、對帳表沒有的訂單（重建時手動補回）：2026/1/16 碧螺春 250ml 360 瓶。
- ⚠️ 試算表未開放「知道連結的人可檢視」，所以儀表板開啟時的自動同步會失敗，顯示的是內嵌快照。

## Supabase 會員系統

- 專案 ref：`rnfoiterxmmryvwbfyua`（組織 jackychi66's Org，Free 方案）
- 資料表 `public.profiles`（RLS 已啟用）：id／email／name／note／role('member'|'admin')／approved／created_at
- Trigger `on_auth_user_created`：signUp 時自動建 profile；Email 驗證信已關閉，改由管理員核准把關
- 常用 SQL：
  ```sql
  select email,name,note,created_at from public.profiles where not approved;      -- 待核准
  update public.profiles set approved=true where email='xxx';                     -- 核准
  update public.profiles set role='admin', approved=true where email='xxx';       -- 設管理員
  ```

### ⚠️ 免費方案自動暫停

症狀：登入出現「**Failed to fetch**」。
解法：到 https://supabase.com/dashboard/project/rnfoiterxmmryvwbfyua 按 **Resume project**（或 **Restore**，數分鐘到數小時）。

**判斷是不是真的暫停**（2026-10-07 實證）：專案被暫停／封存後，`<ref>.supabase.co` 會連 **DNS 都解析不到**，不是回錯誤碼。
- 瀏覽器：開 `https://rnfoiterxmmryvwbfyua.supabase.co/auth/v1/health`，跳錯誤頁（連 `mode:'no-cors'` 的 fetch 都失敗）＝主機不存在。
- GitHub Actions：keepalive 以 **exit code 6**（curl「無法解析主機」）失敗。
- ⚠️ 別誤判成本機網路問題：要同時從兩個網路（本機瀏覽器＋GitHub runner）確認。

**預防**：
- `.github/workflows/keepalive.yml`——cron `0 2 * * *`（每天台灣 10:00），REST 與 Auth 兩端點都檢查且**只接受 HTTP 200**，失敗會寄信通知。
- 每週一跑一次 heartbeat commit（寫 `.github/last-keepalive.txt`），避免 GitHub 因 repo 滿 60 天無活動自動停用排程。
- Cowork 排程 `supabase-keepalive-meishi`（每週一 10:06，需 app 開啟）為輔助。

**⚠️ 教訓**：舊版 keepalive 把 401/403 也當成存活，連續 16 次報 success 卻掩蓋真實狀態；而且每 3 天 ping 一次**並沒有防止專案被暫停**（10/4 還成功，10/7 就掛了）。若再發生，考慮升級 Pro（US$25/月，不會暫停）或改用不需常駐資料庫的登入方案。

查存活：`https://api.github.com/repos/jackychi66/meishi-dashboard/actions/workflows/keepalive.yml/runs`

## 回滾方式

GitHub → 該檔案 → **History** → 選要回復的版本 → 右上 `⋯` → **Revert** 或複製舊內容重新 Commit。
Vercel 也可在 Deployments 頁選舊部署 → `⋯` → **Promote to Production**。

## 已知問題與注意事項

- GitHub 上傳頁偶爾卡住（page still loading）→ 關分頁、開新分頁重來。
- Commit 按鈕點擊有時無效 → 用 JS 找 `Commit changes` 按鈕 click()；彈窗出現後要點**彈窗內**那顆。
- 推送後若 Vercel 沒觸發部署 → 再推一個小 commit（尾端加 `<!-- vN -->`）。
- repo 內有兩個殘留分支 jackychi66-patch-1／patch-2（內容已併入 main，可刪）。
- 生產管理系統（梅侍生產管理系統.html）是**獨立本機工具**，不在此部署範圍。

---

### 變更紀錄

詳細版見 [`CHANGELOG.md`](./CHANGELOG.md)。

- 2026-08-05：新增周轉分析 tab；資料更新至 2026/8/4（庫存 6,208 瓶）；四表加容量小計＋總計；建立 Supabase 保活排程；產出本文件。
- 2026-09-02：改用 Google Drive 連接器取數；資料更新至 2026/8/28（庫存 **10,714 瓶**）＋叫貨紀錄至 2026/9/1；流水事件 535 筆、叫貨 65 筆。修正雲端 8/28 結存列漏扣 30 瓶，確立「雲端結存與計算值衝突時以計算值為準」原則。
- 2026-10-07：Supabase 專案被暫停（DNS 不解析），由使用者 Restore 復原，後台恢復正常。keepalive.yml 改版（每日、雙端點、只接受 200、週一 heartbeat commit）。新增 CHANGELOG.md 並確立所有異動都要留版本記錄。
