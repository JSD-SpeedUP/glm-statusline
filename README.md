# glm-statusline（修復＋增強版）

智譜 GLM Coding Plan 的 Claude Code 狀態欄。基於 [@wangjs-jacky/glm-coding-plan-statusline](https://github.com/wangjs-jacky/glm-coding-plan-statusline) v1.6.0 修改：修復額度錯標 bug，新增 7D 與 Cache 兩個顯示位。

## 修了什麼

**Bug：上游「5H」顯示的其實是周池。** `fetchQuotaLimit()` 不看 `unit` 字段，額度接口返回的兩條 `TOKENS_LIMIT`（5 小時池、周池）先後寫進同一個 `fiveHourQuota`，後一條覆蓋前一條——狀態欄 5H 格顯示的是周池百分比，真實 5 小時用量被覆蓋丟失。

修復後按 `unit` 分流，顯示位如下：

| 顯示位 | 含義 | 數據源 |
|---|---|---|
| 5H | 5 小時窗口積分池已用% | `TOKENS_LIMIT, unit=3` |
| 7D（新增） | 周積分池已用% | `TOKENS_LIMIT, unit=6` |
| MCP | MCP 月額度（次數制）已用% | `TIME_LIMIT` |
| Context | 上下文窗口已用% | Claude Code stdin |
| Cache（新增） | 當前上下文緩存命中率 | Claude Code stdin |

`unit` 字段語義的判定依據見 [docs/glm-quota-api.md](docs/glm-quota-api.md)。

## 安裝

`settings.json`（Claude Code 配置目錄下）：

```json
{
  "statusLine": {
    "type": "command",
    "command": "npx -y github:JSD-SpeedUP/glm-statusline"
  }
}
```

環境變量沿用 `ANTHROPIC_AUTH_TOKEN`／`ANTHROPIC_BASE_URL`（智譜 open.bigmodel.cn 與 Z.AI 自動識別）。

## Cache% 口徑

`cache_read ÷ (input + cache_creation + cache_read)`，即當前上下文輸入側從緩存讀回的比例。GLM Coding Plan 計分公式為 輸入 6.9／緩存命中 1.7／輸出 24（每萬 token），命中價約為新輸入的 1/4——命中越高越省積分，故配色與用量條相反（≥80 綠／50–80 黃／<50 紅）。數據取自 Claude Code 餵給 statusline 的會話 stdin，不打額外 API；新會話首輪前無數據，整段隱藏。

## 選項

- `--compact`／`-c`：單行緊湊模式
- `--local`／`-l`：只用本地上下文，不請求 API
- `--clear-cache`：清除本地緩存（額度數據有 120 秒緩存）

## 已知行為

- 重置時間 ↻ 顯示機器本地時區。
- 改動位置：`src/api.js`（unit 分流）、`src/formatter.js`（7D 與 Cache 顯示位）、`src/index.js`（緩存字段保護）。

## 緻謝與許可

上游 [@wangjs-jacky/glm-coding-plan-statusline](https://github.com/wangjs-jacky/glm-coding-plan-statusline)（MIT）。本修改同以 MIT 發佈，見 [LICENSE](LICENSE)。
