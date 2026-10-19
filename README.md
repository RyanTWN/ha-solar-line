# HA × AstrBot (LINE) 太陽能監控與日報 — 開發紀錄 v2.1

泉海閑居太陽能系統：Home Assistant 監控 SunGery ESP32 逆變器資料，透過 LINE 推播日報 / 警報，並由 AstrBot 在 LINE 中回答查詢。

## 0. 系統架構（方案 C）

為避免 HA 與 AstrBot 共用同一個 LINE Official Account 造成 Webhook 衝突：

- **Webhook 入口獨佔（AstrBot）**：LINE Messaging API 的 Webhook URL 只指向 AstrBot，負責所有使用者訊息與自然語言對話；群聊隔離，僅 `@標記` 喚醒（喚醒詞留空）。
- **系統主動發報（Home Assistant）**：HA 不監聽 Webhook，用 `rest_command` 呼叫 `/v2/bot/message/push` 主動推播給管理員。
- **查詢（方案 B 互補）**：AstrBot 透過 HA 內建 MCP Server 即時查詢感測器（見第 4 節）。
- **額度**：使用者對話走 `replyToken`，免費且不限則數；HA 推播為 Push Message，免費方案每月 200 則（用完停止發送、不收費、次月 1 日歸零）。

## 版本紀錄

| 版本 | 內容 |
| :--- | :--- |
| v1 | `rest_command.line_push` + 18:30 文字日報 + 逆變器離線警報（設定寫在 `configuration.yaml` / `automations.yaml`） |
| v2 | 改為單一 package；以下修正與 A–D 功能 |
| v2.1 | E. 電費估算（農業灌溉電錶、低壓電力非時間電價、每月抄表） |
| v2.2 | 逆變器狀態改用 `binary_sensor.quan_hai_xian_ju_sungeryesp32_ni_bian_qi_gao_jing_inverter_alarm`：區分「告警（on）」與「離線（unavailable）」，分別發送不同警報與恢復通知 |
| v2.3 | 備援手機通知設定為 `notify.mobile_app_sm_s9180`，YAML 已無待填的 TODO |

### v2 / v2.1 變更明細

| 項目 | 內容 |
| :--- | :--- |
| 修正 | Token / User ID 改用 `secrets.yaml`；日報狀態文字依實際情況變化；自給率改由全日能量計算；ESP32 夜間離線時保留最後有效值；夜間不誤報；新增恢復通知；30 分鐘告警冷卻 |
| A. 額度監控 / 備援 | 讀取 LINE 額度 API；`script.solar_notify` 統一推播：剩 ≤20 則時一般通知改走 Companion App，緊急警報 LINE + App 雙發（App 用 critical 通知） |
| B. AstrBot 查詢 HA | HA 內建 MCP Server → AstrBot 掛 MCP，LINE 對話即時查詢（走 replyToken，不吃額度） |
| C. 日落觸發 | 日落後 30 分鐘發日報 |
| D. 週報 / 月報 | `utility_meter` 週、月累計，週一 / 每月 1 日 08:00 推播 Flex 卡片 |
| E. 電費估算 | 泉海閑居農業灌溉電錶（每月抄表）；支援單一費率 / 累進兩種方案，計算本期流動電費與「省下電費」，日 / 週 / 月報皆顯示 |

## 檔案

```
packages/solar_line.yaml   # 所有 HA 設定（rest_command / rest / template / utility_meter / script / automation）
custom_templates/tw_power.jinja  # 電價方案、費率與計算 macro（電價調整只改這裡）
secrets.example.yaml       # secrets.yaml 需要新增的兩個 key
```

## 1. 安裝到 Home Assistant

1. `configuration.yaml` 啟用 packages：
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. 把 `packages/solar_line.yaml` 放到 `/config/packages/`，`custom_templates/tw_power.jinja` 放到 `/config/custom_templates/`。
3. **刪除舊設定**（否則重複）：
   - `configuration.yaml` 的 `rest_command: line_push:` 區塊
   - `automations.yaml`（或 UI）中舊的「每日日報」與「逆變器離線警報」
4. `secrets.yaml` 加入 `line_bearer`、`line_admin_user_id`（見 `secrets.example.yaml`）。
5. 確認實體名稱與你的 HA 一致（已預填泉海閑居的實際名稱）：
   - 逆變器告警：`binary_sensor.quan_hai_xian_ju_sungeryesp32_ni_bian_qi_gao_jing_inverter_alarm`
   - 備援手機通知：`notify.mobile_app_sm_s9180`
   - 計費期已設為每月 11 日歸零（每月 10–12 日抄表），一般不需再改
6. 開發者工具 → YAML → 「檢查設定」→ 重新啟動。
7. 到 **設定 → 輔助** 把 `Solar Weekly Report Enabled` 打開（若想收週報）。

### 驗證

在 開發者工具 → 動作 執行：

```yaml
action: script.solar_send_report
data:
  title: "📊 測試卡片"
  period: "📅 2026-10-19"
  pv: 12.3
  grid: 4.5
  load: 10.2
  saved: 38.5
  note: "✅ 測試"
```

應在 LINE 收到 Flex 卡片；同時檢查 `sensor.line_push_quota_used` 是否 +1。
要測備援：暫時把 `solar_notify` 的 `line_reserve` 改成 `999`，應改收到 App 通知。

## 2. 新建實體一覽

| 實體 | 說明 |
| :--- | :--- |
| `sensor.solar_pv_energy_today` / `_grid_` / `_load_` | 今日能量（ESP32 夜間離線時保留最後有效值） |
| `sensor.solar_self_sufficiency_today` | 全日綠能自給率 |
| `sensor.solar_{pv,grid,load}_{weekly,monthly}` | 週 / 月累計，上期結算在 `last_period` 屬性 |
| `sensor.solar_{grid,load}_billing` | 本計費期（抄表日起）累計度數 |
| `sensor.electricity_cost_billing_period` | 本期電費估算；屬性 `plan`、`tier`、`marginal_rate`（方案、級距、每度單價） |
| `sensor.electricity_cost_without_solar_billing_period` | 若無太陽能的本期電費 |
| `sensor.solar_savings_billing_period` | 本期累計省下電費 |
| `sensor.solar_savings_{daily,weekly,monthly}` | 日 / 週 / 月省下電費 |
| `sensor.line_push_quota_used` / `_limit` / `_remaining` | LINE 推播額度 |
| `counter.solar_offline_alerts_today` | 今日逆變器告警 / 斷線次數（日報引用） |
| `input_boolean.solar_inverter_alert_active` | 告警中旗標（恢復通知用） |
| `input_boolean.solar_weekly_report_enabled` | 週報開關 |

## 3. 推播額度估算

| 類型 | 每月則數 |
| :--- | :--- |
| 日報 | ~30 |
| 週報（開啟時） | 4–5 |
| 月報 | 1 |
| 警報 + 恢復 | 視情況，冷卻 30 分鐘 |
| **保留給警報** | 20 |

一般情況每月約 36 則，遠低於 200。

## 4. AstrBot 透過 MCP 查詢 HA（方案 B 互補）

> ⚠️ 雲端環境無法連到官方文件，以下 HA / AstrBot 的介面名稱與欄位依記憶撰寫，若與你的版本不同請以 UI 為準並回報給我修正。

### 4.1 HA 端

1. **設定 → 裝置與服務 → 新增整合 → Model Context Protocol Server**，LLM API 選 **Assist**。
2. **只暴露唯讀感測器**：設定 → 語音助理 → 暴露（Expose），建議只勾：
   - `sensor.solar_pv_energy_today`、`sensor.solar_grid_energy_today`、`sensor.solar_load_energy_today`
   - `sensor.solar_self_sufficiency_today`、週 / 月累計 6 個、逆變器狀態、即時發電功率（如有）
   - `sensor.line_push_quota_remaining`
   - `sensor.electricity_cost_billing_period`、`sensor.solar_savings_billing_period`、`sensor.solar_savings_daily` / `_monthly`

   > 🔒 **重要**：AstrBot 在群組裡也能被 @ 呼叫，任何被暴露的開關 / 燈 / 插座，群組成員都可能透過對話操控。**不要暴露可控制的設備**，或為 AstrBot 另開權限受限的 HA 使用者。為了讓模型看得懂，可在暴露設定中為實體加上中文別名（例：「今日發電量」）。
3. 個人資料 → 安全性 → **建立長期存取權杖**（建議另建一個名為 `astrbot` 的非管理員使用者再產生）。
4. MCP 端點（依 HA 版本）：
   - 較新版本（Streamable HTTP）：`http://<HA_IP>:8123/api/mcp`
   - 舊版（SSE）：`http://<HA_IP>:8123/mcp_server/sse`

### 4.2 AstrBot 端

同一台主機的 Docker：AstrBot 容器內的 `localhost` 是容器自己，**請用主機的區網 IP**（例：`<HA_HOST_IP>`），或在 AstrBot 的 compose 加 `extra_hosts: ["host.docker.internal:host-gateway"]` 後使用 `host.docker.internal`。

先在 AstrBot 容器內測試連線：

```bash
docker exec -it astrbot curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer <HA_TOKEN>" http://<HA_HOST_IP>:8123/api/
# 預期 200
```

AstrBot WebUI → MCP → 新增伺服器（JSON）：

```json
{
  "transport": "streamable_http",
  "url": "http://<HA_HOST_IP>:8123/api/mcp",
  "headers": {
    "Authorization": "Bearer <HA_LONG_LIVED_TOKEN>"
  }
}
```

SSE 版本把 `transport` 改成 `"sse"`、`url` 改成 `/mcp_server/sse`。啟用後工具列表應出現 `GetLiveContext`、`HassTurnOn` 等 HA 工具。

> 💡 如果 AstrBot 版本的 MCP 不支援遠端 transport，可改用 `mcp-proxy` 以 stdio 方式橋接。

### 4.3 人格 / 系統提示詞（建議加入）

```
你可以使用 Home Assistant 工具查詢家中太陽能系統。
- 使用者詢問發電、用電、自給率、逆變器狀態時，先呼叫 GetLiveContext 取得最新數據再回答，不要猜測。
- 能量單位 kWh 一律稱為「度」，數值四捨五入到小數點後 1 位。
- 「今天」用 Solar ... Energy Today；「這週 / 這個月」用 weekly / monthly 感測器的狀態；「上週 / 上個月」用其 last_period 屬性。
- 電費問題：「本期電費」用 Electricity Cost Billing Period（屬性 marginal_rate 為每度單價；plan 為 tiered 時 tier 才有意義）；「省多少」用 Solar Savings 系列。金額單位為新台幣，取整數，並註明為估算。
- 晚上逆變器離線屬正常，不要當成故障。
- 只能查詢，不要嘗試控制任何設備。
- 用繁體中文、簡短回答。
```

### 4.4 測試

在 LINE 私訊（或群組 @ 機器人）：「今天發電多少？」「這個月自給率多少？」「逆變器現在正常嗎？」

## 5. 後續可再擴充

### 待辦（TODO）

- [ ] **本期總電費估算（含基本電費）**：需先從電費單確認以下資料
  - 契約種類：裝置契約 / 需量契約（經常契約、非夏月契約）
  - 契約容量（瓩）或裝置馬力數
  - 農業動力基本電費減免方式（依每月負載率的減免比例）
  - 取得後在 `tw_power.jinja` 加入基本電費計算（低壓電力：裝置契約 137.50 元/瓩/月；需量經常契約夏月 236.20、非夏月 173.20 元/瓩/月），日報 / 月報顯示「本期總電費約 NT$xx」，並把負載率減免帶來的額外節省計入「省下電費」。

### 其他構想

- **發電異常偵測**：與過去 7 日同時段平均比較，大幅偏低（非陰雨）時提醒清洗面板。
- **Rich Menu**：LINE 選單按鈕「今日發電 / 本月統計」→ AstrBot 回覆，零額度。
- **多人推播**：改用 `/v2/bot/message/multicast`（注意：每位收件人各算 1 則）。

## 6. 電費估算說明

泉海閑居使用**農業灌溉電錶**，屬台電「農業動力用電」（電力類），計費方式通常是「基本電費 + 單一費率流動電費」，**不是**住宅的表燈累進電價；每月 10–12 日抄表。

- **方案切換**（`tw_power.jinja`）：
  - `PLAN = 'flat'`（預設）：每度單一價。已填入台電「低壓電力・非時間電價」流動電費：夏月（6/1–9/30）**2.45** 元/度、非夏月 **2.35** 元/度。
  - `PLAN = 'tiered'`：表燈非時間住宅累進（2024-04-01 起費率），供其他住宅電錶使用。
- **計算方式**：本期省下 = 流動電費(總耗電) − 流動電費(電網取電)。單一費率時即「自用太陽能度數 × 單價」；累進時自用的太陽能會先抵掉最貴的級距。日 / 週 / 月省下金額由 utility_meter 對累計值取差。
- **不計入**：基本電費（農業動力依契約容量 / 馬力計收，並依每月負載率減免）、農業用電補貼與其他優惠。太陽能自用會降低用電度數，可能讓負載率下降而**多減免一些基本電費**，這部分未計入，所以實際省下的金額可能比估算多一點。
- **誤差來源**（皆為估算，以台電帳單為準）：
  - 抄表日在 10–12 日之間浮動，以 11 日歸零近似；
  - 跨夏月 / 非夏月的計費期（5/11–6/11、9/11–10/11），台電按天數比例分攤，這裡以當下月份費率計整期；
  - ESP32 的「電網取電」若與台電電表有落差，結果也會跟著偏差。
- **費率調整**：改 `tw_power.jinja` 後，開發者工具 → 動作 執行 `homeassistant.reload_custom_templates`。

## 7. 已解決的技術問題與踩坑紀錄

| 問題現象 | 根本原因 | 解決方式 |
| :--- | :--- | :--- |
| 群組中非標記發言也會被觸發 | AstrBot 喚醒詞填了 `/`，斜線開頭皆視為指令 | 清空喚醒詞，啟用「隔離會話」與「只 @ 機器人是否觸發等待」 |
| `rest_command.line_push` HTTP 400（May not be empty） | 開發者工具手動執行未傳入 `message` | 呼叫時補上 `data: message: "..."` |
| `Message malformed: not a valid option at '['0']'` | 在單一自動化編輯器貼上帶 `-` 的清單 | 移除最外層清單，改單一自動化格式 |
| `template value should be a string at 'actions[0].data'` | `action` / `data` 縮排錯位 | 每層固定縮排 2 格 |
| 多行文字在 LINE 擠成一行 | 使用 `>-` 折疊標量 | 改用 `\|` 保留換行 |
| 數值顯示冗長小數 | 浮點數未修約 | `\| float(0) \| round(1)` |
| 多行文字導致推播無反應 | 換行未跳脫，JSON 破裂 | payload 使用 `\| to_json` |
| 傍晚日報讀到 0 / 自給率接近 0（v2 修正） | ESP32 隨逆變器斷電變 `unavailable`；`solar_ratio` 為即時值 | 「保留最後有效值」template 感測器；自給率改用全日能量計算 |
| 每晚誤報逆變器離線（v2 修正） | 逆變器夜間休眠屬正常 | 離線（`unavailable`）僅在太陽仰角 > 10° 時告警；逆變器回報告警（`on`）則全天告警 |
| 電費以住宅累進計算錯誤（v2.1 修正） | 泉海閑居為農業灌溉電錶（低壓電力），非表燈累進 | `tw_power.jinja` 改單一費率：夏月 2.45 / 非夏月 2.35 元/度 |
