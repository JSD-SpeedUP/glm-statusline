# 智譜 GLM Coding Plan 額度接口實測筆記

非官方文檔化端點，社區流傳＋雙向實測確認（2026-10-19）。

## 端點與鑑權

- `GET https://open.bigmodel.cn/api/monitor/usage/quota/limit`（Z.AI 對應 `api.z.ai` 同路徑）
- Header：`Authorization` 放 API key 裸值，**不加 Bearer**（裸值 HTTP 200／code 200；缺頭返回 code 1001）

## 返回字段

`data.level`＝套餐檔位（lite／pro／max）。`data.limits` 三條：

| 數據行 | 含義 | 關鍵字段 |
|---|---|---|
| `TOKENS_LIMIT, unit=3, number=5` | 5 小時窗口積分池 | `percentage`＝已用% |
| `TOKENS_LIMIT, unit=6, number=1` | 周積分池 | `percentage`＝已用% |
| `TIME_LIMIT, unit=5, number=1` | MCP 月額度（次數制） | `usage`／`currentValue`／`remaining`＝額度總次／已用次／剩餘次 |

`nextResetTime`＝毫秒時間戳。池容量（絕對積分）接口不返回，按檔位換算（如 Max：周 140,000、5 小時 28,000）。

## unit 語義的判定依據（三條獨立證據）

1. **重置時間矛盾排除**：unit=6 條重置時刻若按 5 小時窗倒推，開窗時間晚於當次讀數——未開的窗不可能已用 55%；unit=3 條倒推自洽。
2. **會話增量對賬**：短時間消耗的積分只推動 unit=3 條上漲，unit=6 條紋絲不動（該增量對周池容量低於取整精度）。
3. **MCP 條自洽**：`currentValue/usage`＝8.2% 與該條 `percentage`＝8 一致。

## 緩存經濟學（與 Cache% 顯示位相關）

- 計分公式（官方，每萬 token 折 1 積分）：輸入 6.9／緩存命中 1.7／輸出 24。命中價約為新輸入的 1/4。
- 緩存為隱式自動前綴匹配，無需配置。TTL 官方未公開（原文僅「有合理時間限制，過期後重新計算」），命中是否續期未知。
- 長會話 agent 消耗 90% 以上在輸入側重讀，緩存命中率基本決定成本；觀測手段即本倉 statusline 的 Cache% 顯示位（口徑見 README）。
