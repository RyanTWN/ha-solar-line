# CLAUDE.md

泉海閑居太陽能監控：Home Assistant（HA）讀取 SunGery ESP32 逆變器資料，經 LINE 推播日報 / 警報；AstrBot 在 LINE 中透過 HA MCP Server 回答查詢。完整背景、安裝步驟與踩坑紀錄見 `README.md`（即開發紀錄）。

所有文件、註解、commit 訊息、推播文字一律使用**繁體中文**。

## 檔案與部署位置

| repo 內 | 部署到 HA | 內容 |
| :--- | :--- | :--- |
| `packages/solar_line.yaml` | `/config/packages/` | 全部 HA 設定（rest_command / rest / counter / input_boolean / template / utility_meter / script / automation） |
| `custom_templates/tw_power.jinja` | `/config/custom_templates/` | 電價方案、費率與計算 macro |
| `secrets.example.yaml` | 內容併入 `/config/secrets.yaml` | 只放佔位字串 |

沒有 build / 測試框架；這個 repo 不是 HA 的 `/config` 本身，無法在這裡跑 HA。

## 架構規則（不要違反）

- **方案 C**：LINE Webhook 只給 AstrBot；HA 只做 Push（`/v2/bot/message/push`），不要在 HA 加任何 Webhook 接收。
- **所有推播一律走 `script.solar_notify`**（處理額度、備援、critical），不要在自動化裡直接呼叫 `rest_command.line_push` 或 `notify.*`。報表卡片走 `script.solar_send_report`。
- 免費方案每月 200 則 Push，`line_reserve: 20` 保留給 critical 警報。新增任何定期推播前先估算每月則數，並更新 README「推播額度估算」表。

## HA YAML 慣例

- 使用 HA 2024.10+ 語法：`triggers:` / `conditions:` / `actions:`、`- trigger: state`、`action:`（不用 `service:` / `platform:`）。
- 縮排固定 2 格。
- 自建實體的 `name` 一律用**英文**（entity_id 才可預測），中文顯示名稱由使用者在 UI 改。每個實體都要 `unique_id`。
- 數值：`| float(0)`，顯示前 `| round(1)`；能量對使用者一律稱「度」。
- 多行推播文字用 `|` / `|-`，不要用 `>-`（換行會被摺疊）。
- LINE payload 一律 `| to_json` 產生，不要手拼 JSON 字串。
- `!secret` 只能當整個 YAML 值，**不能用在 Jinja 模板裡**；需要在模板中使用時，先放進 script `variables`（見 `solar_notify` 的 `line_to`）。
- `custom_templates` 的 macro 回傳的是**字串**，使用時要 `| float` / `.split(',')`。

## 資料的坑

- ESP32 晚上會隨逆變器斷電 → 原始 `sensor.sungeryesp32_*` 會變 `unavailable`。報表、utility_meter、MCP 查詢一律使用「保留最後有效值」的 `sensor.solar_{pv,grid,load}_energy_today`，**不要改回讀原始感測器**。
- 這三個 held 感測器**刻意不在午夜歸零**（避免 ESP32 早上先回報昨日舊值被 utility_meter 重複計入），歸零跟隨來源。
- 逆變器狀態實體是 `binary_sensor.quan_hai_xian_ju_sungeryesp32_ni_bian_qi_gao_jing_inverter_alarm`：`on` = 逆變器告警（全天都要警報）、`off` = 正常、`unavailable` = ESP32 離線（夜間屬正常，只在 `sun.sun` 仰角 > 10° 時警報）。
- 週 / 月報讀 utility_meter 的 `last_period` 屬性（上一期結算值），不是 state。

## 電費（`tw_power.jinja`）

- 泉海閑居是**農業灌溉電錶**，適用台電「低壓電力・非時間電價」：**單一費率**，不是住宅累進。`PLAN = 'flat'`，夏月（6/1–9/30）2.45、非夏月 2.35 元/度。
- **每月**抄表（10–12 日），計費期 utility_meter 以 `cron: "0 0 11 * *"` 近似；`BIMONTHLY = false`。
- 只計算流動電費；基本電費、負載率減免、補貼都未計入。
- 電價調整只改 `tw_power.jinja`，部署後執行 `homeassistant.reload_custom_templates`。

## 安全

- **絕不提交** `secrets.yaml`、`.storage/`、`home-assistant_v2.db*`、真實 Channel Access Token、LINE User ID（`U` + 32 hex）、HA 長期權杖、真實 IP。文件與範例一律用佔位字串：`<CHANNEL_ACCESS_TOKEN>`、`Uxxxxxxxx…`、`<HA_TOKEN>`、`<HA_HOST_IP>`。
- 提交前掃描：
  ```bash
  F=$(git ls-files -co --exclude-standard)
  grep -nE '\bU[0-9a-f]{32}\b|eyJ[A-Za-z0-9_-]{10,}|\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' $F
  ```
  有任何命中先停下來問使用者。
- MCP 只暴露**唯讀感測器**給 AstrBot：AstrBot 在 LINE 群組也能被 @，暴露可控制的設備等於讓群組成員能操控。

## 修改後的驗證

1. YAML 結構（處理 `!secret` tag）：
   ```bash
   python3 - <<'EOF'
   import yaml
   class L(yaml.SafeLoader): pass
   L.add_constructor('!secret', lambda l, n: f"<secret:{l.construct_scalar(n)}>")
   d = yaml.load(open('packages/solar_line.yaml'), Loader=L)
   print('OK', list(d))
   EOF
   ```
2. 電價 macro（預期 6 月 `245.0 1,2.45`、10 月 `235.0 1,2.35`）：
   ```bash
   python3 - <<'EOF'
   import jinja2, datetime
   for m in (6, 10):
       e = jinja2.Environment(loader=jinja2.FileSystemLoader('custom_templates'))
       e.globals['now'] = lambda m=m: datetime.datetime(2026, m, 15)
       print(m, e.from_string("{% from 'tw_power.jinja' import tw_cost, tw_tier %}{{ tw_cost(100) }} {{ tw_tier(100) }}").render())
   EOF
   ```
   注意：每次測試要建新的 `Environment`，否則 Jinja 會快取 import 的模板。
3. 改到模板邏輯時，用 `jinja2` 實際 render 關鍵模板（payload、Flex bubble、日報 note）確認輸出。
4. 這些只是離線檢查；回報時要說明「尚未在真實 HA 上執行」，並請使用者用 README「驗證」一節的步驟在 HA 測試。

## 文件與 Git

- 每次功能變更都要同步更新 `README.md`：版本紀錄、新建實體一覽、必要時加入「已解決的技術問題」表（表格內的 `|` 要寫成 `\|`）。
- 待辦統一記在 README「5. 後續可再擴充 → 待辦（TODO）」。
- Commit 訊息格式：`feat: …` / `fix: …` / `docs: …`，中文描述。
- 未經使用者要求不要建立 Pull Request、不要 force push。

## 尚待使用者提供（YAML 中標 `TODO`）

- `notify.mobile_app_your_phone` → Companion App 通知服務（2 處）
- 契約種類與容量（瓩）→ 才能做「本期總電費（含基本電費）」
