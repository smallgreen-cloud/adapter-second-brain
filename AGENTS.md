# AGENTS.md — Second Brain 部署引導（跨 CLI，keyword-only 降級版）

你是導遊不是裁判：每一關綠燈由機械驗證判定。同一步驟失敗 3 次即停，記錄卡點。使用者確認不超過 3 次。

## 前置

- node 20+、npm（上游有 package-lock.json，安裝可重現）；`CLOUDFLARE_API_TOKEN`＋`CLOUDFLARE_ACCOUNT_ID` 環境變數（Workers＋D1＋KV Edit 權限；**不需 Vectorize 權限——本 adapter 即為無該權限帳號的降級版**）
- 帳號需已註冊 workers.dev 子網域；Workers AI 免費額度可用（embedding＋LLM 用）
- 降級語義：語意搜尋→關鍵字搜尋（上游原生 fallback）；語意去重與向量鄰居邊停用（patch 0002 non-fatal）

## 步驟

### 1. 說明與同意（確認 1）

讀 `.smallgreen/profile.yaml` 向使用者說明：這是個人知識庫＋MCP server；筆記存自己帳號 D1（珍貴資料，維護契約含備份義務）；本版為 keyword-only 降級（要語意搜尋需帳號 token 有 Vectorize 權限，見 UPSTREAM.md 還原程序）；嵌入與 LLM 走 Workers AI 免費額度。確認 AUTH_TOKEN 要設什麼。

### 2. 組裝

```bash
git clone https://github.com/rahilp/second-brain-cloudflare.git second-brain && cd second-brain
git checkout 371a8f987b2e4e8d95fc60d20580fb4757877641   # 與 UPSTREAM.md 一致
git apply <本repo>/patches/0001-remove-vectorize-binding.patch
git apply <本repo>/patches/0002-vectorize-writepath-nonfatal.patch
npm ci
```

### 3. 部署（確認 2）

```bash
npx wrangler deploy   # auto-provision D1（second-brain-db）與 OAUTH_KV
printf "%s" "<32+ 亂數>" | npx wrangler secret put AUTH_TOKEN
```

- **E45 陷阱**：wrangler auto-provision 若中途失敗，已建資源的 ID 不會寫回 wrangler.jsonc——重跑會撞 already exists（10014）。此時 `npx wrangler d1 list` / `kv namespace list` 查 ID 手動填入 wrangler.jsonc 再部署
- D1 migrations：依上游 db/ 與 README 指示執行（如有 migration 檔）

### 4. 驗收（確認 3）

照 `.smallgreen/acceptance.yaml`：

1. `GET /` → 200
2. 帶 AUTH_TOKEN 擷取一則筆記 → success 且 **不得 500**（寫路徑防呆驗證點；端點路徑讀上游 src/routes）
3. 關鍵字搜尋命中＋降級提示（「keyword matches only」語義）
4. 無 token 打寫入端點 → 401/403
5. 刪除筆記 → deleted、再搜不命中

全過才算完成（Build 成功不算）。

## 維護與移除

照 `.smallgreen/maintenance.yaml`。**移除前必先 d1 export（筆記是珍貴資料）**；移除＝worker＋D1＋OAUTH_KV＋repo 副本。
