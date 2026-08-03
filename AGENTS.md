# AGENTS.md — Second Brain 部署引導（跨 CLI，keyword-only 降級版）

你是導遊不是裁判：每一關綠燈由機械驗證判定。同一步驟失敗 3 次即停，記錄卡點。使用者確認不超過 3 次。
自主修復邊界照 spec v0.3.0 的 AGT-1/2/3：可補裝缺漏開發依賴、生成型別、對傳播期暫態重試，**但每次修復必須記入報告**；不得改服務邏輯、放寬驗收或跳過閘門。

## 前置

- node 22+、npm（上游 wrangler 4.x 要求 node>=22；有 package-lock.json，安裝可重現）；`CLOUDFLARE_API_TOKEN`＋`CLOUDFLARE_ACCOUNT_ID` 環境變數（Workers＋D1＋KV Edit 權限；**不需 Vectorize 權限——本 adapter 即為無該權限帳號的降級版**）
- 帳號需已註冊 workers.dev 子網域；Workers AI 免費額度可用（embedding＋LLM 用）
- 降級語義：語意搜尋→關鍵字搜尋（上游原生 fallback）；語意去重與向量鄰居邊停用；擷取的筆記 `vector_ids` 記為空（落入上游 `unvectorized` 回補集合）；`POST /migration/reembed` 回 503 而非假成功
- 閘門指令（乾淨環境須全數 exit 0，CON-8；順序不可調換——tsc 依賴 wrangler types 的生成物）：`npx wrangler types` → `npx tsc --noEmit` → `npm test`。**兩個 patch 都套上後仍須全綠**：patch 0002 含型別宣告，缺它降級樹會在 13 處報 TS2339

## 步驟

### 1. 說明與同意（確認 1）

讀 `.smallgreen/profile.yaml` 向使用者說明：這是個人知識庫＋MCP server；筆記存自己帳號 D1（珍貴資料，維護契約含備份義務）；本版為 keyword-only 降級（要語意搜尋需帳號 token 有 Vectorize 權限，見 UPSTREAM.md 還原程序）；嵌入與 LLM 走 Workers AI 免費額度。確認 AUTH_TOKEN 要設什麼。

### 2. 組裝

```bash
git clone https://github.com/rahilp/second-brain-cloudflare.git second-brain && cd second-brain
git checkout 4e5424efae99e9666d4aa901f1ad2715bac0cb13   # 與 UPSTREAM.md 一致（Worker v2.2.0）
git apply <本repo>/patches/0001-remove-vectorize-binding.patch
git apply <本repo>/patches/0002-vectorize-optional-binding.patch
npm ci
```

### 3. 部署（確認 2）

```bash
# 上游 wrangler.jsonc 有 secrets.required: ["AUTH_TOKEN"]——wrangler 4.114+ 對不存在的 worker
# 拒絕「先 deploy 再 secret put」順序（C run 1 實測必失敗）。用 secrets-file 一次帶入：
printf '{"AUTH_TOKEN":"%s"}' "<32+ 亂數>" > /tmp/sb-secrets.json
npx wrangler deploy --secrets-file /tmp/sb-secrets.json   # auto-provision D1（second-brain-db）與 OAUTH_KV
rm /tmp/sb-secrets.json
```

- **E45 陷阱**：wrangler auto-provision 若中途失敗，已建資源的 ID 不會寫回 wrangler.jsonc——重跑會撞 already exists（10014）。此時 `npx wrangler d1 list` / `kv namespace list` 查 ID 手動填入 wrangler.jsonc 再部署
- D1 migrations：依上游 db/ 與 README 指示執行（如有 migration 檔）

### 4. 驗收（確認 3）

照 `.smallgreen/acceptance.yaml`：

1. `GET /` → 200
2. 帶 AUTH_TOKEN 擷取一則筆記 → success 且 **不得 500**（寫路徑防呆驗證點；端點路徑讀上游 src/routes）
3. 關鍵字搜尋：命中時回應含 `"semantic_unavailable": true`；零命中時回 prose 降級提示（Vectorize index missing）——HTTP 路徑的降級證據是這兩個，「keyword matches only」prose 只在 MCP 路徑（C run 1 實測）
4. 無 token 打寫入端點 → 401/403
5. 刪除筆記 → deleted、再搜不命中

全過才算完成（Build 成功不算）。

## 維護與移除

照 `.smallgreen/maintenance.yaml`。**移除前必先 d1 export（筆記是珍貴資料）**；移除＝worker＋D1＋OAUTH_KV＋repo 副本。
