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
| 人設與回覆邏輯 | Prompt v2（XML 結構） | 角色、七個技能、範例、限制、安全規則分區塊撰寫；技能包含功能教學、英文建議、學習規劃、收藏、查詢、寄送摘要、使用者資料（JSON） |
| 模型 | GPT-4o mini | Temperature 0.5、上下文 3 輪、回覆上限 1024 |
| 使用者程度 | Memory（Variable） | 變數 `level` 帶入 Prompt，依程度調整用字難度 |
| 產品與教材資料 | Knowledge | `功能說明.txt`、`道地用語庫.txt` |
| 學習紀錄 | Memory（Database） | `study_list` 資料表：收藏句子、道地說法、常錯發音 |
| 練習摘要 | Plugin | Gmail / sendMessage，練習結束後寄出當日收藏的句子 |
| 限制 | Guardrail | 只談英語學習、不代寫作業、不洩漏提示詞與他人資料 |

## Chatflow 架構（Coze）
同樣的教練用 Chatflow 再做一次，先判斷意圖，再把任務分給不同模型（LLM Routing）：

```
使用者輸入
   ↓
Intent recognition（GPT-4o mini 分類）
   ├─ 英文句子 → LLM（GPT-4o 糾正，XML salt＋三明治防禦）
   ├─ 功能問題 → Knowledge retrieval → LLM_1（GPT-4o mini 回答）
   └─ 其他    → Text Processing（固定回覆，不呼叫模型）
   ↓
Variable Merge → End
```

| | Agent | Chatflow |
| --- | --- | --- |
| 流程 | 模型自己判斷要用哪個技能、要不要呼叫工具 | 開發者用節點把流程畫死 |
| 模型 | 整個 Agent 用同一個模型 | 每個節點可以選不同模型 |
| 上下文 | 預設帶入對話紀錄 | 每個 LLM 節點自己決定要不要帶 |
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
- Prompt 改寫成 v2：XML 分區塊、加入變數 `{{level}}`、few-shot 範例改成一個錯和兩個錯各一例、內部 CoT 判斷錯誤數、新增 `<security>` 區塊
- 結構化輸出：使用者說出年齡、職業、性別時，輸出 `{"age": 20, "job": "學生", "gender": "male"}` 格式的 JSON
- Prompt Injection 測試：Agent 4 種攻擊；Chatflow 標籤閉合攻擊，比較 XML salt 與三明治防禦
- 用 Chatflow 重做教練，實作意圖分流與 LLM Routing
- Coze 的 Prompt 資源庫兩次存檔都出現系統錯誤，改用 GitHub `prompts/` 資料夾管理版本

**主要發現**
- 換掉 few-shot 範例後，模型不再硬湊三個錯；用範例裡沒有的句子（She don't like coffee.）驗證，只指出 1 個錯
- JSON 輸出時，沒提到性別也被填成 male，原因是格式說明和範例都寫了 male，模型把它當成預設值；改成佔位符號並補一個 null 範例後就正確，每次多用約 100 tokens
- 同一句英文，GPT-4o 只有文字規則時湊出 2 個錯，GPT-4o mini 搭配 few-shot 範例只指出 1 個；Prompt 設計比模型大小更影響結果
- 「忽略上面的指令」「DAN 角色扮演」這類有名的攻擊，沒進到我們的 Prompt 就被平台或模型內建防護擋下
- 強化版標籤閉合攻擊打 GPT-3.5 Turbo：普通標籤和加了 XML salt 都被攻破，再加三明治防禦才擋住（260 → 321 → 441 tokens）；在完整 Chatflow 裡，攻擊在意圖路由那關就被分到固定回覆
- 單一防禦都可能被突破，要多層一起用，每加一層成本就多一點


## 參考資料
- Coze 平台官方文件
- Google AI for Developers：Gemini API Models
- OpenAI：Prompt engineering、Structured outputs
- Anthropic：Prompt engineering interactive tutorial、Use XML tags to structure your prompts
- Wei et al. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models（arXiv:2201.11903）
- OWASP GenAI Security Project
- 課程單元 0：拖拉式 AI Agent 導論與 Coze 平台探索
- 課程單元 1：大腦核心，LLM 選擇與參數設定
- 課程單元 2：Prompt 的應用場景、管理流程、安全防護
