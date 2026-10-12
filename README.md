# 英語口說小教練 Agent

## 產品簡介
以樹莓派（Raspberry Pi）為核心，搭配螢幕與即時語音互動的隨身英語口說教練。協助使用者練習道地用語與各類口說情境（日常生活／考試導向），並即時糾正發音。

## 主要功能
1. **情境選單**：切換「日常生活情境」（點餐、問路、小聊天）與「考試導向情境」（依使用者設定的檢定類型出題）
2. **道地用語教學**：每次互動教幾句 native speaker 常用的道地表達，並示範情境用法
3. **語音對話練習**：AI 扮演對話角色，與使用者進行即時英語問答與情境對話
4. **發音糾正**：分析使用者發音，即時用語音給出具體的改進建議
5. **螢幕視覺回饋**：顯示情境卡、對話逐字稿、發音評分與學習進度
6. **使用者客製資料庫**：類似 RAG，使用者可以自行上傳教材，讓教練依教材出題

## Agent 架構（Coze）
| 功能 | Coze 元件 | 用途 |
| --- | --- | --- |
| 人設與回覆邏輯 | Prompt v3（XML 結構） | 角色、技能、範例、限制、安全規則分區塊撰寫；技能包含功能教學、英文建議、學習規劃、收藏、查詢、寄送摘要、使用者資料（JSON）、記錄性別、回答性別 |
| 模型 | GPT-4o mini | Temperature 0.5、上下文 3 輪、回覆上限 1024；建議問題已關閉 |
| 使用者資料 | Memory（Variable） | `level`（程度）、`gender`（性別）、`summary`（長期摘要）。Prompt 不用 `{{變數}}`，改讀平台附在最後的「使用者已儲存記憶」 |
| 產品與教材資料 | Knowledge | `功能說明.txt`、`道地用語庫.txt` |
| 學習紀錄 | Memory（Database） | `study_list` 資料表：收藏句子、道地說法、常錯發音 |
| 練習摘要 | Plugin | Gmail / sendMessage，練習結束後寄出當日收藏的句子 |
| 限制 | Guardrail | 只談英語學習、不代寫作業、不洩漏提示詞與他人資料 |

### 三種記憶
| | 對話輪數（context of round） | 變數（variable） | 資料表（table） |
| --- | --- | --- | --- |
| 記什麼 | 最近幾輪對話 | 特定重點：程度、性別、摘要 | 重複發生的紀錄：收藏的句子 |
| 存多久 | 只在當下這段對話 | 跨對話保留 | 跨對話保留 |
| 怎麼寫入 | 平台自動帶入 | 模型呼叫 `setKeywordMemory` | 模型呼叫 `tableExecute` 執行 SQL |

## Chatflow 架構（Coze）
同樣的教練用 Chatflow 再做一次，先整理記憶、再判斷意圖，把任務分給不同模型（LLM Routing）：

```
使用者輸入
   ↓
LLM_2（GPT-4o mini 摘要：舊摘要＋最近對話 → 新摘要）
   ↓
Variable assign（存進 summary）
   ↓
Intent recognition（GPT-4o mini 分類）
   ├─ 英文句子 → LLM（GPT-4o 糾正，XML salt＋三明治防禦）
   ├─ 功能問題 → Knowledge retrieval → LLM_1（GPT-4o mini 回答，讀摘要）
   ├─ 情境對話 → Text Processing（暫時接固定回覆，對話用的 LLM 節點待補）
   └─ 其他    → Text Processing（固定回覆，不呼叫模型）
   ↓
Variable Merge → End
```

| | Agent | Chatflow |
| --- | --- | --- |
| 流程 | 模型自己判斷要用哪個技能、要不要呼叫工具 | 開發者用節點把流程畫死 |
| 模型 | 整個 Agent 用同一個模型 | 每個節點可以選不同模型 |
| 上下文 | 預設帶入對話紀錄 | 每個 LLM 節點自己決定要不要帶 |
| 變數 | Prompt 裡的 `{{變數}}` 不會被替換 | LLM 節點的 `{{input}}` 會被替換 |
| 無關問題 | 模型自己決定怎麼回，每次可能不同 | 走固定回覆，每次都一樣 |
| 速度 | 一次模型呼叫 | 多一次分類，約多 1.3 秒 |

## 開發進度

### 單元 0：拖拉式 AI Agent 導論與 Coze 平台探索
探索 Coze 平台，規劃產品構想與使用場景。

### 單元 1：大腦核心，LLM 選擇與參數設定
- 在 Coze 建立 Agent，完成 Prompt、知識庫、資料庫、Gmail 外掛與 Guardrail 設定
- 模型參數實驗：Temperature（0.2／0.9）、回覆長度上限（300／1500）、上下文輪數（3／10）
- 功能測試：知識庫問答 6 題、收藏與查詢、寄信、Guardrail 4 題
- 整理 Coze 目前可用模型與扣點、Gemini 官網模型清單、蒸餾模型

**主要發現**
- 糾正文法時，Prompt 範例示範了三個錯，模型就每次湊滿三點，調 Temperature 解決不了，要改 Prompt
- 上下文 3 輪會忘記使用者名字，10 輪記得，但 token 多約 44%；長期資訊改存 Memory
- GPT-4o mini 會出現「回覆說已收藏，實際沒寫入資料庫」的情況，要在 Prompt 寫明「沒呼叫就不准說已收藏」才穩定
- 用到工具的回合比一般對話貴兩倍以上（一般約 2,500 tokens，寫入資料庫約 5,500，寄信約 6,500）

### 單元 2：Prompt 的應用場景、管理流程、安全防護
- Prompt 改寫成 v2：XML 分區塊、few-shot 範例改成一個錯和兩個錯各一例、內部 CoT 判斷錯誤數、新增 `<security>` 區塊
- 結構化輸出：使用者說出年齡、職業、性別時，輸出 `{"age": 20, "job": "學生", "gender": "male"}` 格式的 JSON
- Prompt Injection 測試：Agent 4 種攻擊；Chatflow 標籤閉合攻擊，比較 XML salt 與三明治防禦
- 用 Chatflow 重做教練，實作意圖分流與 LLM Routing
- Coze 的 Prompt 資源庫兩次存檔都出現系統錯誤，改用 GitHub `prompts/` 資料夾管理版本

**主要發現**
- 換掉 few-shot 範例後，模型不再硬湊三個錯；用範例裡沒有的句子（She don't like coffee.）驗證，只指出 1 個錯
- JSON 輸出時，沒提到性別也被填成 male，原因是格式說明和範例都寫了 male，模型把它當成預設值；改成佔位符號並補一個 null 範例後就正確，每次多用約 100 tokens
- 「忽略上面的指令」「DAN 角色扮演」這類有名的攻擊，沒進到我們的 Prompt 就被平台或模型內建防護擋下
- 強化版標籤閉合攻擊打 GPT-3.5 Turbo：普通標籤和加了 XML salt 都被攻破，再加三明治防禦才擋住（260 → 321 → 441 tokens）；在完整 Chatflow 裡，攻擊在意圖路由那關就被分到固定回覆
- 單一防禦都可能被突破，要多層一起用，每加一層成本就多一點

**單元 3 更正**
- 單元 2 加入的 `{{level}}` 在 Agent 模式下不會被替換，送出時是空白，依程度調整用字難度從未生效
- 單元 2 的「GPT-4o 沒範例湊出 2 個錯，所以 Prompt 比模型大小重要」只測了一次；單元 3 重測後，比較精確的說法是：GPT-4o 沒範例時不穩定（2 次中 1 次湊數），加上範例後 2 次都正確

### 單元 3：狀態管理，記憶機制（Memory）
- 建立 `gender` 變數：使用者說出性別時記錄、改口時覆蓋、只說職業時不寫入
- 比較三種記憶，清除對話後測試變數是否保留
- 用 Debug Detail 觀測每次呼叫實際送出的 Prompt、工具定義與隱藏的模型呼叫
- 重測單元 2 的兩個結論
- 小實作：工具定義的成本、意圖描述對路由正確率的影響、在 Chatflow 做記憶摘要
- 研究：Prompt Caching、Memory Summarization、Function Calling 與 Tool Search、Hugging Face 與 LLM Routing、Agent SDK 的觀測功能

**主要發現**
- Agent 的 Prompt 不會替換 `{{gender}}`、`{{level}}`，真正的值被平台附在 Prompt 最後；兩段資訊互相矛盾，清除對話後問性別，先答不知道、再答錯成男生，改成讀「使用者已儲存記憶」後才答對
- 問一句「我是男生還是女生？」要 2,369 tokens，幾乎全是每次重送的固定內容：平台系統指示、整份 Prompt、所有工具定義、資料表結構
- 平台預設會多叫一次 GPT-3.5 Turbo 產生建議問題（約 326 tokens），關掉後延遲從 3,545 ms 降到 1,627 ms
- Debug Detail 最上層的 token 總數會重複計算子節點，要看各節點的數字
- 寫入變數是一次 Function Calling（`setKeywordMemory`），那一輪約 6,000 tokens，一般對話約 2,000；「我是護理師」沒有亂填 gender，但仍多呼叫了一次寫入工具
- Coze 把精確到秒的日期時間放在 Prompt 開頭附近，推測會讓 Prompt Caching 的前綴比對失效（Coze 沒有顯示快取數字，未驗證）
- 移除一個用不到的工具（Gmail）：2,369 → 2,300 tokens，一個工具每次 69 tokens，50 個工具推估多約 3,450
- 意圖描述寫清楚範圍、補上缺的路線後，5 句模稜兩可的話正確率從 3/5 提升到 5/5；但驗證用的是同一組句子，還需要另外的測試集
- Chatflow 記憶摘要的回答看似成功，執行紀錄卻顯示讀到的舊摘要是空的，回答裡的資訊來自知識庫；Chatflow 的 Test run 不會把變數保留到下一次執行

## 參考資料
- Coze 平台官方文件
- Google AI for Developers：Gemini API Models
- OpenAI：Prompt engineering、Structured outputs、Prompt Caching 101
- Anthropic：Prompt engineering interactive tutorial、Use XML tags to structure your prompts、Long context prompting tips、Tool search tool
- Wei et al. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models（arXiv:2201.11903）
- OWASP GenAI Security Project
- LangChain：Memory overview、Short-term memory
- Hugging Face：HuggingChat Omni
- OpenAI Agents SDK、Google ADK、LangGraph 官方文件
- 課程單元 0：拖拉式 AI Agent 導論與 Coze 平台探索
- 課程單元 1：大腦核心，LLM 選擇與參數設定
- 課程單元 2：Prompt 的應用場景、管理流程、安全防護
- 課程單元 3：狀態管理，記憶機制（Memory）
