# adapter-second-brain

> [Second Brain（Cloudflare）](https://github.com/rahilp/second-brain-cloudflare)（自架個人知識庫＋MCP server，696★）的 SmallGreen 適配 repo（Path C，**keyword-only 降級版**）。

[![conformance](https://github.com/smallgreen-cloud/adapter-second-brain/actions/workflows/conformance.yml/badge.svg)](https://github.com/smallgreen-cloud/adapter-second-brain/actions/workflows/conformance.yml)

**驗證等級：Discovered**（收錄 ≠ 驗證）

上游程式碼不在本 repo：[UPSTREAM.md](UPSTREAM.md) 鎖定 commit、[.smallgreen/](.smallgreen/) 契約三檔、[patches/](patches/)（移除 Vectorize binding＋寫路徑 non-fatal 防呆——token 無 Vectorize 權限帳號適配；防呆部分屬上游缺口，建議回饋 upstream）、[AGENTS.md](AGENTS.md) 非互動部署路徑。

## 資料流向（信任揭露）

筆記與知識圖譜存部署者自己帳號的 D1（珍貴資料——維護契約含備份義務）。嵌入與 LLM 走帳號內 Workers AI。無外連、無遙測。API/MCP 全端點以 AUTH_TOKEN 認證。

降級語義：語意搜尋→關鍵字搜尋；語意去重與向量鄰居邊推斷停用。帳號 token 具 Vectorize 權限時可還原完整模式（見 UPSTREAM.md）。

License：MIT（隨上游）。
