> 我今天有兩套系統(A、B)都有客戶資料
> 
> 我今天要做 ETL
> 
> 整個整理好的資料庫會長怎樣？


下面我用「兩套系統 A / B 都有客戶資料，要做 ETL」的典型做法，給你一個**整理後（目標端）資料庫長相**的參考藍圖。你可以把它想成：先把原始資料完整收進來（可追溯），再做清洗與整併（可查詢、可分析、可營運）。

---

## 1) 目標資料庫分三層最穩：RAW → STAGE → DW（整併後）

### A. `raw_*`（原始層 / 落地層）

目的：**完整保留 A、B 來的原始資料**，不急著改欄位、不急著合併，先確保可追溯、可重跑。

* `raw_a_customer`（A 系統客戶原始表）
* `raw_b_customer`（B 系統客戶原始表）
* `raw_a_customer_address`, `raw_b_customer_address`（若有多地址）
* `raw_a_customer_contact`, `raw_b_customer_contact`（電話/Email/聯絡人）
* `raw_etl_batch`（每次匯入批次紀錄：來源、時間、筆數、檔名/版本、hash）

常見欄位（每張 raw 表都會有）：

* `source_system`（A/B）
* `source_pk`（來源主鍵）
* `extract_time`
* `ingest_batch_id`
* `raw_payload`（可選：JSON 原文，或原始欄位完整保存）
* `row_hash`（用來偵測是否變更）

> RAW 層的重點是「不丟資料、可回放」。

---

### B. `stg_*`（清洗層 / 標準化層）

目的：把 A、B 的欄位**對齊成同一套格式**（日期格式、電話格式、地址拆欄、統一代碼），但**還不做最終合併成唯一客戶**。

* `stg_customer`（把 A/B 客戶欄位標準化後放一起）

  * `source_system`、`source_pk`
  * `customer_name_std`
  * `email_std`、`phone_std`
  * `id_no_hash`（身分證/統編建議存 hash，不直接明碼）
  * `birthday`、`gender_code`
  * `status_code_std`（把 A/B 狀態碼映射成統一值）
  * `dq_flags`（資料品質旗標：email 無效、電話不完整…）
  * `valid_from`、`valid_to`（可選）

另外會有對照表（很重要）：

* `ref_code_mapping`（A/B 的代碼 → 統一代碼，例如會員等級、城市代碼、狀態碼）
* `dq_error_log`（哪一筆、哪個欄位、什麼規則沒過）

> STG 層的重點是「欄位統一、品質可控、錯誤可追」。

---

### C. `dw_*`（整併後 / 可信任層，提供查詢、報表、系統共用）

目的：把 A/B 客戶**整併成你的「客戶主檔（Customer Master）」**，建立跨系統唯一鍵、去重、版本控管。

核心表通常會長這樣：

#### 1) `dw_customer_master`（客戶黃金主檔 / Golden Record）

每個「真實客戶」一筆（或一個版本一筆，看你要不要做歷史）。

* `customer_id`（你產生的全域唯一ID，UUID/序號都可）
* `customer_name`
* `primary_email`
* `primary_phone`
* `id_no_hash`（如果允許且必要）
* `customer_type`（個人/公司）
* `status`（有效/停用/黑名單…）
* `created_at`、`updated_at`
* （若做歷史版本）`effective_from`, `effective_to`, `is_current`

> 這張表是「整理好之後你希望大家都用的客戶資料」。

#### 2) `dw_customer_xref`（跨系統對照表 / Source to Master Mapping）

把 A / B 的客戶主鍵映射到 `customer_id`。

* `source_system`（A/B）
* `source_pk`
* `customer_id`
* `match_rule`（用哪個規則匹配：email+phone / id_hash / 人工確認）
* `confidence_score`
* `is_manual_override`（是否人工指定）
* `linked_at`

> 這張表讓你可以回答：「A 系統的客戶 123，在主檔是哪個 customer_id？」

#### 3) `dw_customer_address`（主檔地址，可能一對多）

* `customer_id`
* `address_type`（戶籍/通訊/寄送…）
* `country`、`city`、`district`、`zipcode`、`address_line`
* `is_primary`
* （可選）`effective_from`, `effective_to`

#### 4) `dw_customer_contact`（主檔聯絡方式，多筆）

* `customer_id`
* `contact_type`（email/phone/line…）
* `contact_value`
* `is_primary`
* `verified_flag`

#### 5) `dw_merge_audit`（合併紀錄/可回溯）

* `customer_id`
* `source_system`、`source_pk`
* `merge_time`
* `rule`、`operator`（如果有人工合併）

---

## 2) 你整理好的「客戶主檔」會怎麼形成（去重與優先權）

整併時通常會有一套策略（你不一定要一次做完，但要留擴充空間）：

* **匹配鍵（Match Keys）**

  * 強鍵：身分證/統編（hash）、會員卡號（若跨系統共用）、手機（較強）
  * 弱鍵：姓名+生日、姓名+地址、email（可能多人共用/變更）

* **欄位優先權（Survivorship Rule）**

  * 例如：`phone` 以 B 為準，`email` 以 A 為準
  * 或：以「最新更新時間」為準
  * 或：以「已驗證 verified」為準

* **歷史版本（SCD Type 2，可選）**

  * 若你希望知道客戶資料變更史（例如地址換了），就做 `effective_from/to`
  * 若不需要，主檔就只保留 current

---

## 3) 一個簡化的「整理後資料庫長相」示意

你最終可能會長這樣（最常見的最小集合）：

* RAW

  * `raw_a_customer`
  * `raw_b_customer`
  * `raw_etl_batch`

* STAGE

  * `stg_customer`
  * `ref_code_mapping`
  * `dq_error_log`

* DW（整併後、可用）

  * `dw_customer_master`
  * `dw_customer_xref`
  * `dw_customer_address`
  * `dw_customer_contact`
  * `dw_merge_audit`

---

## 4) 你可以直接套用的「客戶主檔最小欄位建議」

如果你只想先把骨架做起來，`dw_customer_master` 建議至少有：

* `customer_id`（全域鍵）
* `customer_name`
* `customer_type`
* `primary_phone`
* `primary_email`
* `status`
* `created_at`、`updated_at`

再搭配：

* `dw_customer_xref(source_system, source_pk, customer_id, match_rule, confidence_score)`

這兩張先有，你的 A/B 就能「同指向一個客戶」。
