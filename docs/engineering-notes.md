# 進階附錄：把概念對照到原始碼

[返回 README](../README.md)

這份附錄整理原文的原始碼閱讀、context 管理、工具介面與工程檢查項目。先讀完主文、需要追實作時再看；這些細節不是理解 harness 的前提。

主標本固定在 [OpenLAIR/nano-claude-code @ 39fe347](https://github.com/OpenLAIR/nano-claude-code/tree/39fe3470d8379adaee2486f5d06cbf318879d85a)。以下觀察來自靜態閱讀，不代表最新版，也不代表已執行完整測試或安全稽核。本文重新整理並修正部分過度概括的說法；完整舊筆記仍可在[修改前版本](https://github.com/WilliamCHIU-ETH/what-is-agent-harness/blob/4b1d576d0079b6ef6f698059d4192adcffe0df49/README.md)查看。

## 1. API、迴圈與狀態

讀 [agent.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/agent.py)。

| 位置 | 責任 | 閱讀重點 |
|---|---|---|
| AgentState | 持有 messages、token 統計與輪數 | 這是執行期間狀態；持久化由其他程式負責 |
| run_streaming | 互動用迴圈，產生供介面顯示的事件 | 模型回應加入 messages，工具結果也加入 messages |
| run_agent_loop | 自動執行入口，可輸出 NDJSON 事件 | Anthropic 路徑用 max_turns 限制輪數，工具直接交給 dispatch_tool |
| _api_call_create / _api_call_streaming | API 呼叫與部分重試邏輯 | 分清呼叫失敗、串流中途失敗與工具執行失敗 |

兩個入口有相似骨架，但不能假定它們的權限行為相同。模型回應中的停止原因也是程式的分支條件之一，不代表模型獨自掌握整個任務的停止權。

### API 請求裡有什麼？

以下節錄互動迴圈中的請求組裝，省略其他處理：

```python
kwargs = {
    "model": model,
    "max_tokens": max_tokens,
    "system": cached_system,
    "messages": msgs_for_api,
    "tools": tools,
}
```

這幾個欄位就是主文的介入位置：程式可以改變指令、歷史、工具說明與輸出上限，再把請求交給服務。max_tokens 限制單次回應，與整個任務的輪數上限是不同限制。

### 程式也能產生訊息

這個版本在輸出碰到 token 上限時，會追加一則要求接續的訊息，最多續寫三次。壓縮歷史時，也會由程式組成摘要與一則 assistant 確認文字。

所以，messages 裡的 role 是 API 對話結構，不等於每一則 user 都是人親手輸入、每一則 assistant 都曾由模型生成。追查行為時，應保留訊息的實際來源。

## 2. 執行前檢查：看到函式，還要追呼叫路徑

對照 [permissions.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/permissions.py) 與 [agent.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/agent.py)。

在 run_streaming 的工具分支中，順序是：

1. 取得工具名稱與輸入參數。
2. 用 needs_permission 判斷是否需要確認。
3. 如需確認，產生 PermissionRequest，等介面回覆。
4. 未獲允許時加入拒絕結果，略過這次 dispatch_tool。
5. 允許或不需確認時，才調用 dispatch_tool。

| 實際路徑或設定 | 這個固定版本的行為 |
|---|---|
| 互動迴圈＋manual | needs_permission 對所有工具要求確認 |
| 互動迴圈＋auto | 依工具與參數判定是否確認 |
| 互動迴圈＋accept-all | 這層確認直接放行；run_streaming 的函式預設值也是此模式 |
| run_agent_loop 的 Anthropic 路徑 | 直接 dispatch_tool，沒有同一段 needs_permission 確認 |

這些是確認機制的觀察，不代表工具內部或作業系統完全沒有其他限制。也不能由「函式預設值」直接推斷所有使用者入口最終採用的設定。

### 為什麼字串比對不能取代環境隔離？

這個版本的 is_safe_bash 以指令前綴分類；清單包含 Python 等通用執行器。另一個 is_dangerous_bash 函式的存在，也不代表它有被 needs_permission 呼叫。

通用執行器可能產生多種副作用，僅看指令開頭無法完整判斷。若需求是「不可寫入某個位置」，應檢查所有可寫入途徑，並利用檔案權限或沙箱落實邊界。

專用讀檔工具比任意 shell 更容易限定操作，但「唯讀」仍可能讀到敏感資訊；工具名稱、參數 schema 與免確認，都不是安全保證。真正的邊界取決於實作、允許的目標與環境權限。

## 3. Context 管理：組裝、縮短、保存是不同責任

對照 [prompts.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/prompts.py)、[memory.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/memory.py)、[agent.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/agent.py) 與 [session.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/session.py)。

| 階段 | 範例中的做法 | 取捨與界線 |
|---|---|---|
| 組裝 prompt | 加入環境、Git 狀態、專案規範、agent 與 skill 目錄 | 載入順序不等於一個可量化、保證生效的模型權重 |
| 載入規範 | memory.py 沿目錄層級讀取規範檔，設字元上限 | 此處的 memory 是規範檔載入，不等於通用的長期記憶抽取系統 |
| 判斷壓縮 | 參考上一輪 input_tokens，否則估算訊息長度 | 應釐清快取 token 等欄位的計數範圍，避免把局部數字當完整 context |
| 壓縮歷史 | 保留最近六則，對部分較舊內容做另一次模型摘要 | 摘要有成本，也可能遺失資訊 |
| 持久化 | session.py 將紀錄寫成 JSON 檔案 | 檔案保存了資料，不代表後續每輪都會送入全部內容 |

### 靜態閱讀可看見的壓縮風險

- 互動路徑在加入新使用者訊息前檢查壓縮；長工具迴圈內仍可能累積過多輸出。
- 直接保留最後六則，可能切斷工具請求與結果的配對。裁切時應維持 API 所需的訊息結構。
- 摘要呼叫失敗時，只留下「已壓縮若干則訊息」的佔位文字，並不能保留舊內容的語意。
- 某段工具輸出曾經被模型看過，不代表它的所有重要資訊都留在後續回應；移除時仍要評估損失。

這些是由程式推得的風險，本文沒有提供失敗重現或事故頻率的證據。

### 壓縮可以由輕到重，但各有代價

工程上可以先減少過大的單筆輸出，再移出舊內容，最後對歷史摘要。原筆記的階梯可整理為：

| 方法 | 仍保留什麼？ | 要付出什麼？ |
|---|---|---|
| 大結果存檔，context 留預覽與路徑 | 原始資料可回查 | 模型需要再呼叫工具，且必須知道何時回查 |
| 把較舊的中段歷史歸檔 | 可追溯的紀錄 | 當前推論不再直接看見細節 |
| 移除或縮短舊工具輸出 | 近期工作內容 | 可能移除後面仍需使用的證據 |
| 模型摘要 | 較短的工作脈絡 | 摘要可能漏掉或誤述重點，並增加模型呼叫 |

壓縮可以在請求前觸發、在 API 拒絕過長輸入後補救，或讓模型主動請求；採用哪一種，要追該實作的流程。這是設計選項，不表示主標本已實作全部策略。

### Cache 與上述機制的關係

這個版本以 _build_cached_system 與 _add_cache_breakpoints 標記可快取內容，並仍然傳送 messages。快取機制與可用選項會隨 API 演進，請以 [官方文件](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) 為準。

## 4. 工具介面與錯誤回饋

讀 [tools_impl.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/tools_impl.py)。

專用工具能把常用操作的參數、輸出與限制寫得更明確。例如 Glob 可以設定排除目錄與結果數量；Edit 可以要求替換內容唯一，並在完成後回傳差異。Shell 則能處理尚未包裝成專用工具的操作。

工具拆分後，更容易按用途設定輸出上限；即使只使用 shell，也仍可依操作或任務設計不同預算。工具越多，不必然越好，還要考慮選擇成本與介面重疊。

### 模型需要知道怎麼從失敗繼續

tool_edit 在找不到要替換的內容時，會提示重新讀檔；在匹配多處時，提示補充上下文或設定全部替換。這比只回傳「失敗」更有助於下一輪判斷。

讀錯誤處理時，要同時看訊息和機器可讀的錯誤旗標。這個版本的 dispatch_tool 兜底回傳以 `Error in` 開頭的文字，但外層部分路徑用 `startswith("Error:")` 判定，格式並不一致。文字裡描述了錯誤，不保證外層正確標示 is_error。

## 5. Skills、多 agent 與長期記憶

| 延伸能力 | 放在主文 mental map 的哪裡？ | 需要注意的事 |
|---|---|---|
| Skill | 需要時把專門指引加入輸入，或交給另一條工作流程 | 目錄描述協助選擇；實際可用能力仍取決於工具與權限 |
| 子 agent | 啟動另一個有自己 context 的模型與工具迴圈 | 可以隔離探索、分工或並行；也增加費用與結果驗證工作 |
| 長期 memory | 保存資訊，並在後續任務召回 | 抽取、篩選、合併與召回都可能出錯，不保證無損 |

在主標本裡，[skills.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/skills.py) 處理 skill 發現與內容展開；[agents.py](https://github.com/OpenLAIR/nano-claude-code/blob/39fe3470d8379adaee2486f5d06cbf318879d85a/nano_claw_code/agents.py) 定義 agent 與工具篩選；工具執行入口在 tools_impl.py。

這些能力與 context 管理相關，但用途不只節省 context。分工、權限隔離、工作規範重用與跨任務延續，也各有價值。

如果保存的內容來自對話或外部資料，應保留來源並區分規範、事實與不可信內容。文字標記可以協助模型判斷，實際執行邊界仍需要程式與權限。

想進一步跟著教學實作，可看 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)。其目錄與課程會變動，這裡不把舊版章節編號當成固定介面。

## 6. 實作時再使用的檢查清單

### 呼叫與迴圈

- [ ] 模型完成、輪數上限、取消、API 失敗，各有明確出口與回報嗎？
- [ ] 單次輸出上限和整個任務預算是否分開？
- [ ] 串流中途失敗、重試與重複執行工具，分別怎麼處理？
- [ ] 互動、自動、子 agent 等入口，是否有不同的檢查路徑？

### 輸入與狀態

- [ ] 本輪輸入包含哪些資料？各自從哪裡來？
- [ ] 紀錄、長期記憶與本輪 context 是否分清楚？
- [ ] 裁切後，工具請求與結果是否仍保持有效配對？
- [ ] 摘要失敗時如何保留工作進度？關鍵原文能否回查？
- [ ] Context 計量是否涵蓋工具定義、system、快取內容及其他必要部分？

### 工具與執行權限

- [ ] 每條可執行操作的途徑，都會經過所需的檢查嗎？
- [ ] 參數格式正確之外，目標路徑與動作本身是否也允許？
- [ ] Shell、腳本和子 agent，能否繞過專用工具的限制？
- [ ] 工具程序的環境權限，是否符合使用者設定？
- [ ] 拒絕、失敗與成功是否回傳足夠資訊，讓下一輪能判斷？
- [ ] 文字訊息與錯誤旗標是否一致？

### 延伸能力

- [ ] 子 agent 收到什麼、能用什麼、回傳什麼？父層如何驗證？
- [ ] Skill 與 memory 是否只載入需要的內容，且保留來源與適用範圍？
- [ ] 跨 session 的資訊是否仍正確，過時內容如何修正？

筆記文字採 CC BY 4.0；主標本與程式碼節錄採原專案 MIT 授權。
