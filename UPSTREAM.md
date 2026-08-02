# UPSTREAM

| 欄位 | 值 |
|---|---|
| 上游 | [rahilp/second-brain-cloudflare](https://github.com/rahilp/second-brain-cloudflare) |
| 鎖定 commit | `371a8f987b2e4e8d95fc60d20580fb4757877641` |
| License | MIT |
| 鎖定日期 | 2026-08-01 |

## patches 對照

| patch | 內容 | 為什麼 |
|---|---|---|
| 0001-remove-vectorize-binding.patch | wrangler.jsonc 移除 vectorize binding 段 | token 無 Vectorize 權限的帳號（含驗證沙盒，error 10000——B run E43）無法部署帶此 binding 的 worker。移除後上游讀路徑原生降級為 keyword-only（recall/search.ts 的 non-fatal catch） |
| 0002-vectorize-writepath-nonfatal.patch | 寫路徑 5 處補 try/catch non-fatal（store.ts upsert/deleteStale/insert、duplicate.ts query、traverse.ts neighborsFromVectorQuery），沿用上游 lifecycle.ts 既有防呆風格 | 上游 graceful degradation 只覆蓋讀路徑與 lifecycle——capture 寫路徑在 binding 缺失時 500（**上游真 bug**，B run E44 首例「需改碼才能部署」）。副作用（記錄非缺陷）：語意去重與向量鄰居邊推斷在降級模式停用 |

**還原完整模式**：帳號 token 具 Vectorize 權限時，撤 patch 0001 即回復語意搜尋；patch 0002 保留無害（防呆只在 binding 缺失時觸發）。注意 E45：wrangler auto-provision 中途失敗不寫回資源 ID，重跑撞 already exists（10014）——需手動查 API 補 ID。

## 同步程序

1. `gh api repos/rahilp/second-brain-cloudflare/commits/main --jq .sha` 取新 commit，與鎖定值 diff 審閱（重點：src 內 env.VECTORIZE 呼叫點增減 vs patch 0002 覆蓋面、wrangler.jsonc 資源）
2. 對新 commit `git apply --check patches/*.patch`
3. 更新本檔與 profile.yaml 的 `upstream.commit`（兩處一致，CON-6 驗證）

## 建議回饋上游

patch 0002 的寫路徑防呆適合開 upstream issue/PR（binding 缺失時 capture 500 是通用問題，不只沙盒情境）。
