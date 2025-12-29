<blockquote>
我今天其中一個系統的資料架構如下

```
foo_file	客戶主檔	foo01	客戶編號	
foo_file	客戶主檔	foo02	客戶簡稱	
foo_file	客戶主檔	foo03	客戶分類	
```

要交給 AI 辨識，轉換資料欄位名稱比較好，還是使用 dictionary table ？

</blockquote>

結論先講：**兩個都要，但主力放在 dictionary table（資料字典/對照表）**；欄位名稱要不要改，取決於你「只做一次」還是「要長期維運、可追溯、可擴充」。

---

## 為什麼主力建議用 dictionary table

你這份 `foo_file` 的欄位有幾個特性：

* 欄位代碼很多、而且有兩套：`foo*` + `fooa*`（看起來像申請/簽核版本或變更檔）
* 有不少 **No Use**、重複（例如 fax no-2 出現兩次）、同義（收款客戶編號出現多次）、以及「一段式地址 foo231~foo235 / foo241~foo245」
* 未來如果另外一套系統 B 也有不同命名，你會遇到更多 mapping

這種情境如果直接「改欄位名稱」：

* 你會把原系統語意硬寫死在 schema 裡，一旦原系統升版、欄位新增/廢止，你要改 DB + 程式 + ETL
* 可追溯性差：很難回答「這個欄位最早從哪個 source field 來？哪天改過 mapping？」

用 dictionary table 可以：

* 把「欄位代碼 → 標準欄位」做成資料驅動（config-driven）
* 支援版本控管、欄位廢止、別名、轉換規則、資料品質規則
* 之後 AI 要辨識也更穩：AI 不是直接看你 DB 欄位名猜意思，而是看「你整理過的欄位語意描述 + 規則」。

---

## 建議架構：欄位名稱標準化 + dictionary table 驅動（最佳解）

### 1) 資料庫層面不要動 raw，標準化從 stg/dw 做

* RAW：保留 `foo01/foo02...` 原樣，永遠可回放
* STG/DW：用「標準欄位命名」(例如 `customer_code`, `tax_id`, `invoice_address_line1`…)

AI 主要吃：**你的 dictionary + 標準欄位語意**，而不是 raw 欄位代碼。

---

## 什麼時候「直接改欄位名稱」才比較好？

只有在這幾種情況我才會偏向直接改：

1. **你只會用一次匯入**（短期專案、匯完就不再同步）
2. 原系統欄位很少（十幾個以內）、且穩定不會變
3. 你不需要追溯、也不會做跨系統整併/去重

你這個 `foo_file` 顯然不是。

---

## 你可以做的 dictionary table 長相（實務上很夠用）

建議至少兩張：

### A) `field_dictionary`（欄位字典）

| 欄位               | 例                                                  |
| ---------------- | -------------------------------------------------- |
| `source_system`  | A                                                  |
| `source_table`   | foo_file                                           |
| `source_field`   | foo01                                              |
| `source_desc_zh` | 客戶編號                                               |
| `standard_field` | customer_code                                      |
| `data_type`      | string                                             |
| `pii_level`      | low/medium/high                                    |
| `transform_rule` | trim / to_date / phone_normalize / concat_address… |
| `is_active`      | Y                                                  |
| `version`        | v1                                                 |

### B) `code_dictionary`（代碼表/枚舉 mapping）

像 `foo1004 狀態碼 0-開立,1-確認,2-留置,3-停止交易`
你要有一張把它轉成標準狀態：

| domain          | source_code | standard_code | standard_name |
| --------------- | ----------- | ------------- | ------------- |
| customer_status | 0           | OPEN          | 開立            |
| customer_status | 1           | CONFIRMED     | 確認            |
| customer_status | 2           | HOLD          | 留置            |
| customer_status | 3           | STOP          | 停止交易          |

> 這兩張表讓 ETL/AI 都有「可依循的語意層」。

---

## AI 辨識角度：dictionary table 反而是讓 AI 更準的關鍵

AI 真正需要的是：

* 欄位的中文描述（你已經有）
* 同義詞/別名（收款客戶編號、發票客戶編號、送貨客戶編號…）
* 結構資訊（foo231~foo235 是發票地址多段、foo241~foo245 是送貨地址多段）
* 轉換規則（電話正規化、地址合併、日期格式）

這些都**不適合只靠「改欄位名稱」承載**，因為欄位名最多放 1 個意思，放不了規則、版本、備註、例外。

---

## 我會給你的落地建議（你現在就能做）

1. **RAW 不改欄位**：原樣進來（foo01…）
2. 做一份 `field_dictionary`：把你貼的這份直接變成表格資料（含 `standard_field`）
3. STG 依 dictionary 產生標準欄位（可用程式自動產 SQL / 自動生成 mapping）
4. `foo*` vs `fooa*`：在 dictionary 裡標記 `record_type = master / apply`，避免混用
5. 把「No Use」欄位在 dictionary 設 `is_active = N`，但 raw 仍保留

---

如果你想，我可以直接用你這段資料，幫你先做出一版「標準欄位命名建議」：

* `foo01 -> customer_code`
* `foo11 -> tax_id`
* `foo18/foo19 -> legal_name_line1/line2`
* `foo231~foo235 -> invoice_address_line1~line5`
* `foo241~foo245 -> shipping_address_line1~line5`
* `foo261~foo263 -> phone1~phone3`
* `foo1004 -> status_code`

  …並把 `foo* / fooa*` 分類好，讓你 dictionary table 可以直接匯入。
