# UPSTREAM

| 欄位 | 值 |
|---|---|
| 上游 | [rahilp/second-brain-cloudflare](https://github.com/rahilp/second-brain-cloudflare) |
| 鎖定 commit | `4e5424efae99e9666d4aa901f1ad2715bac0cb13`（Worker v2.2.0） |
| License | MIT |
| 鎖定日期 | 2026-08-03（前次 371a8f9 / 2026-08-01，相隔 96 個 commit） |

## patches 對照

| patch | 內容 | 為什麼 |
|---|---|---|
| 0001-remove-vectorize-binding.patch | wrangler.jsonc 移除 vectorize binding 段 | token 無 Vectorize 權限的帳號（含驗證沙盒，error 10000——B run E43）無法部署帶此 binding 的 worker。移除後上游讀路徑原生降級為 keyword-only（recall/search.ts、lifecycle.ts 既有的 catch） |
| 0002-vectorize-optional-binding.patch | `src/env.ts` 把 `VECTORIZE` 宣告為選配；寫路徑（store.ts 的 upsert／deleteStale／append insert／reembedOrThrow、duplicate.ts、traverse.ts）改為「binding 不存在才跳過」；已有 catch 的讀路徑加非空斷言；`/migration/reembed` 在索引不可用時回 503 | 上游 graceful degradation 只覆蓋讀路徑與 lifecycle——capture 寫路徑在 binding 缺失時 500（**上游真缺口**，B run E44 首例「需改碼才能部署」）。**patch 0001 同時移除了生成型別裡的 `VECTORIZE`，故降級樹的 typecheck 會在 13 處報 TS2339——型別宣告是本 patch 的必要組成，不是附帶整理** |

### patch 0002 的三條設計約束（2026-08-03 改版時確立）

1. **只在 binding 不存在時跳過，不吞例外**。前一版用 try/catch 一律吞掉，破壞了上游用測試明文釘住的錯誤傳播語義（`append.test.ts` 的「Vectorize 失敗回 500」與 `vectorize-pending.test.ts` 的失敗計數兩案，套用前一版會紅）。現版對上游 **981 個測試零改動全綠**——降級是部署形狀，索引錯誤是真故障，兩者不可混為一談。
2. **不記錄不存在的向量**。降級模式下 `vector_ids` 寫入空陣列，使該筆落入上游既有的 `vector_ids = '[]'` 集合（admin 路由的 `unvectorized` 統計與回補對象）——日後綁上索引即可被回補。前一版記錄幻影 id，會讓這些筆記永久不被回補。
3. **不製造新的沉默失敗**。v2.2 新增的 `POST /migration/reembed` 直接呼叫 `storeEntry`，跳過向量寫入後會逐筆回報成功卻什麼都沒寫；因此用上游自己的 `checkVectorizeHealth` 在該路由前 fail-closed 回 503。**這個沉默失敗是我方 patch 造成的**（未套 patch 時該路由會誠實 500），所以由我方封住。

**還原完整模式**：帳號 token 具 Vectorize 權限時，撤 patch 0001 即回復語意搜尋；patch 0002 保留無害（守衛只在 binding 缺失時生效，選配型別在兩種模式皆通過 tsc）。注意 E45：wrangler auto-provision 中途失敗不寫回資源 ID，重跑撞 already exists（10014）——需手動查 API 補 ID。

## 同步程序

1. `gh api repos/rahilp/second-brain-cloudflare/commits/main --jq .sha` 取新 commit，與鎖定值 compare 審閱（重點：wrangler.jsonc 資源增減、`env.VECTORIZE` 呼叫點增減）
2. 對新 commit 依序 `git apply --check patches/0001-*.patch` 與 `patches/0002-*.patch`
3. **兩種模式都要過 `profile.yaml` 的 `build_requirements.gates`**：完整模式（只套 0002）須 tsc 零錯誤且 `npm test` 全綠；降級模式（0001+0002）須 tsc 零錯誤
4. 更新本檔與 profile.yaml 的 `upstream.commit`（兩處一致，CON-6 驗證）
5. 重跑 conformance，並重驗一輪產生新 Evidence Pack（跨版本存活＝D5b）

**新呼叫點靠型別偵測，不靠巡檢**：`VECTORIZE` 宣告為選配後，上游任何新增的未處理 `env.VECTORIZE.x` 在第 3 步就是型別錯誤。v2.2 一次新增了 `src/vectorize/health.ts` 與 `src/migration/embedding.ts` 兩個相關檔案——靠 grep 巡檢遲早會漏。

## 上游問題的處理方式

本 adapter 發現的上游缺口（如 E44 寫路徑）**只記錄在本檔與 Evidence Pack**，供部署者自行判斷，不主動對上游發 issue／PR。
