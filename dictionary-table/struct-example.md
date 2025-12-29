## Dictionary Table 架構

> 一般作 ETL ，dictionary table 大概長怎樣

一般在做 ETL（或 ELT）時講的 **dictionary table**，多半是指「**資料字典 / 對照字典**」：把來源欄位、目標欄位、轉換規則、代碼對照、資料品質規則等，用**表格化、可版本化**的方式存起來，讓 ETL 可以「讀表做事」，而不是把規則硬寫死在程式裡。

下面給你幾種最常見的 dictionary table 長相（你可以依需求選 1~3 種就夠用了）。

---

### 1) 欄位對應字典（Field Mapping Dictionary）— 最常見

用途：來源欄位 → 目標欄位、型別、是否必填、預設值、基本轉換。

**table: `etl_field_mapping`**

| src_system | src_table | src_column | tgt_table    | tgt_column  | tgt_type    | transform_expr                                             | is_pk | is_required | default_value | rule_order | is_active |
| ---------- | --------- | ---------- | ------------ | ----------- | ----------- | ---------------------------------------------------------- | ----: | ----------: | ------------- | ---------: | --------: |
| ERP        | sales     | cust_id    | dim_customer | customer_id | varchar(50) | `trim(upper(:cust_id))`                                    |     1 |           1 |               |         10 |         1 |
| ERP        | sales     | gender     | dim_customer | gender      | char(1)     | `case when :gender in ('M','F') then :gender else 'U' end` |     0 |           0 | U             |         20 |         1 |

重點欄位建議：

* `transform_expr`：用你們常用語法（SQL expression、Spark SQL、dbt macro、或自訂 DSL）
* `rule_order`：同一個欄位有多條規則時排序
* `is_active`：方便灰度上線 / 停用規則

---

### 2) 代碼對照字典（Code/Value Mapping）— 例如狀態碼、分類碼

用途：把來源代碼翻成標準代碼（或中文描述），常用在維度表、狀態欄位。

**table: `etl_value_dictionary`**

| domain | src_system | src_code | tgt_code | tgt_desc | effective_from | effective_to | is_active |
| ------ | ---------- | -------- | -------- | -------- | -------------- | ------------ | --------: |
| GENDER | ERP        | M        | M        | Male     | 2020-01-01     | 9999-12-31   |         1 |
| GENDER | ERP        | 1        | M        | Male     | 2020-01-01     | 9999-12-31   |         1 |
| GENDER | ERP        | F        | F        | Female   | 2020-01-01     | 9999-12-31   |         1 |

重點欄位建議：

* `domain`：同一張表塞多個對照集（性別、站點、物料分類…）
* `effective_from/to`：支援「同一代碼不同時期意義不同」的情境

---

### 3) 資料品質規則字典（DQ Rules Dictionary）

用途：把檢核規則寫進表裡，ETL 跑完可以自動產出檢核報表 / 失敗落地。

**table: `etl_dq_rules`**

| rule_id | tgt_table     | tgt_column  | rule_type     | rule_expr                                            | severity | on_fail_action | is_active |
| ------- | ------------- | ----------- | ------------- | ---------------------------------------------------- | -------- | -------------- | --------: |
| 101     | fact_purchase | net_weight  | range         | `net_weight >= 0 and net_weight <= 50000`            | error    | reject_row     |         1 |
| 102     | dim_customer  | customer_id | not_null      | `customer_id is not null`                            | error    | reject_row     |         1 |
| 103     | fact_purchase | station_id  | ref_integrity | `station_id in (select station_id from dim_station)` | warn     | quarantine     |         1 |

重點欄位建議：

* `severity` + `on_fail_action`：錯誤要擋、要隔離、還是只警告

---

### 4) 來源系統/表級中繼資料字典（Source Metadata / Ingestion Dictionary）

用途：控制「要抽哪些表、抽取模式（全量/增量）、watermark 欄位、排程」。

**table: `etl_source_table_config`**

| src_system | src_table | load_type   | incr_column | watermark_value     | schedule    | is_active |
| ---------- | --------- | ----------- | ----------- | ------------------- | ----------- | --------: |
| ERP        | sales     | incremental | updated_at  | 2025-12-25 23:59:59 | daily_01:00 |         1 |
| ERP        | customer  | full        |             |                     | weekly_sun  |         1 |

---

### 實務上怎麼選？

* 你只想把「欄位對應＋轉換」表格化：做 **(1)** 就夠。
* 你們有大量代碼轉換、分類翻譯：一定要加 **(2)**。
* 你們常被追問「資料到底準不準」：加 **(3)**，很值。
* 你們想把 ingestion 也配置化：加 **(4)**。

## 範例

<blockquote>
我現行的 table 欄位如下：

```
foo_file	客戶主檔	foo01	客戶編號	
foo_file	客戶主檔	foo02	客戶簡稱	
foo_file	客戶主檔	foo03	客戶分類	
foo_file	客戶主檔	foo04	負責業務員編號	
foo_file	客戶主檔	foo05	1.永久性 2.暫時性	
foo_file	客戶主檔	foo06	性質	
foo_file	客戶主檔	foo07	收款客戶編號	
foo_file	客戶主檔	foo08	慣用發票別	
foo_file	客戶主檔	foo09	送貨客戶編號	
foo_file	客戶主檔	foo10	No Use	
foo_file	客戶主檔	foo1001	營業職照有效開始日期	
foo_file	客戶主檔	foo1002	營業職照失效開始日期	
foo_file	客戶主檔	foo1003	所屬業態	
foo_file	客戶主檔	foo1004	狀態碼	0-開立,1-確認,2-留置,3-停止交易
```
以一般 SQL 的版本寫一套範本
</blockquote>

下面我用「**一般 SQL（偏 ANSI）**」寫一套 **資料字典 / ETL dictionary** 的範本，並把你提供的 `foo_file` 欄位資料 **示範寫入**。
（如果你用的是 PostgreSQL/MySQL/SQL Server，把 `TEXT` 的 JSON 改成 `JSON/JSONB/NVARCHAR(MAX)` 就好。）

---

### 1) 資料字典核心表（Table / Column）

#### 1-1. 表級資料字典 `md_table`

```sql
CREATE TABLE md_table (
  md_id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  src_system       VARCHAR(50)  NOT NULL,      -- 來源系統/資料源，例如 ERP
  table_name       VARCHAR(128) NOT NULL,      -- 來源表名，例如 foo_file
  table_zh_name    VARCHAR(200),              -- 中文表名，例如 客戶主檔
  table_desc       VARCHAR(500),              -- 表用途描述
  owner_team       VARCHAR(100),              -- 維護單位
  version          VARCHAR(30)  DEFAULT 'v1',  -- 版本
  effective_from   DATE         DEFAULT CURRENT_DATE,
  effective_to     DATE         DEFAULT DATE '9999-12-31',
  is_active        SMALLINT     DEFAULT 1,
  created_at       TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  updated_at       TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT uq_md_table UNIQUE (src_system, table_name, version)
);
```

#### 1-2. 欄位級資料字典 `md_column`

> `example_values` 用 `TEXT` 存 JSON 字串（跨資料庫最通用）。
> 若你是 PostgreSQL 建議改 `TEXT` → `JSONB`；MySQL 8 可用 `JSON`。

```sql
CREATE TABLE md_column (
  md_id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  src_system         VARCHAR(50)  NOT NULL,
  table_name         VARCHAR(128) NOT NULL,
  column_name        VARCHAR(128) NOT NULL,

  column_zh_name     VARCHAR(200),        -- 中文名稱：客戶編號
  business_definition VARCHAR(1000),      -- 業務定義：這欄位代表什麼
  data_type          VARCHAR(50),         -- 來源型別（若未知先留空）
  nullable_flag      SMALLINT DEFAULT 1,  -- 1=可為空,0=不可為空
  is_pk              SMALLINT DEFAULT 0,  -- 1=PK
  ordinal_position   INTEGER,             -- 欄位排序（可選）

  domain_name        VARCHAR(100),        -- 若是代碼/枚舉，填 domain（對應代碼表）
  example_values     TEXT,                -- JSON array 字串：["A001","A002"]
  notes              VARCHAR(1000),       -- 備註（例如 No Use / 0-開立...）

  version            VARCHAR(30) DEFAULT 'v1',
  effective_from     DATE        DEFAULT CURRENT_DATE,
  effective_to       DATE        DEFAULT DATE '9999-12-31',
  is_active          SMALLINT    DEFAULT 1,

  created_at         TIMESTAMP   DEFAULT CURRENT_TIMESTAMP,
  updated_at         TIMESTAMP   DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT uq_md_column UNIQUE (src_system, table_name, column_name, version),
  CONSTRAINT fk_md_column_table
    FOREIGN KEY (src_system, table_name, version)
    REFERENCES md_table (src_system, table_name, version)
);
```

---

### 2) 代碼/枚舉字典（讓 AI 很好懂的關鍵） `md_code_dictionary`

```sql
CREATE TABLE md_code_dictionary (
  md_id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  domain_name      VARCHAR(100) NOT NULL,     -- 例如 foo_STATUS
  src_system       VARCHAR(50)  NOT NULL,
  src_table        VARCHAR(128),
  src_column       VARCHAR(128),

  src_code         VARCHAR(50)  NOT NULL,     -- 例如 0
  tgt_code         VARCHAR(50),               -- 標準碼（可同 src_code）
  tgt_desc_zh      VARCHAR(200),              -- 中文意義：開立

  effective_from   DATE        DEFAULT CURRENT_DATE,
  effective_to     DATE        DEFAULT DATE '9999-12-31',
  is_active        SMALLINT    DEFAULT 1,

  CONSTRAINT uq_md_code UNIQUE (domain_name, src_system, src_code, effective_from)
);
```

---

### 3) 把你提供的 `foo_file` 寫進資料字典（示範）

#### 3-1. 先新增表資料

```sql
INSERT INTO md_table (src_system, table_name, table_zh_name, table_desc, owner_team, version)
VALUES ('ERP', 'foo_file', '客戶主檔', '客戶基本資料與狀態/業態資訊', 'IT/BI', 'v1');
```

#### 3-2. 新增欄位資料（以你提供的內容為主）

> 你目前沒給型別/是否可空，我先留空或採預設（nullable=1）。之後你補上再 UPDATE。

```sql
INSERT INTO md_column
(src_system, table_name, column_name, column_zh_name, business_definition, notes, domain_name, ordinal_position, version)
VALUES
('ERP','foo_file','foo01','客戶編號','客戶唯一識別碼',NULL,NULL,1,'v1'),
('ERP','foo_file','foo02','客戶簡稱','客戶簡稱/顯示名稱',NULL,NULL,2,'v1'),
('ERP','foo_file','foo03','客戶分類','客戶分類（依 ERP 定義）',NULL,NULL,3,'v1'),
('ERP','foo_file','foo04','負責業務員編號','負責該客戶之業務員識別碼',NULL,NULL,4,'v1'),

('ERP','foo_file','foo05','永久性/暫時性','客戶屬性：永久/暫時','1.永久性 2.暫時性','foo_CUSTOMER_TYPE',5,'v1'),
('ERP','foo_file','foo06','性質','客戶性質（依 ERP 定義）',NULL,NULL,6,'v1'),
('ERP','foo_file','foo07','收款客戶編號','收款對象客戶編號（可能不同於交易客戶）',NULL,NULL,7,'v1'),
('ERP','foo_file','foo08','慣用發票別','預設發票類型',NULL,NULL,8,'v1'),
('ERP','foo_file','foo09','送貨客戶編號','送貨對象客戶編號（可能不同於交易客戶）',NULL,NULL,9,'v1'),

('ERP','foo_file','foo10','No Use','已停用/保留欄位','No Use',NULL,10,'v1'),

('ERP','foo_file','foo1001','營業職照有效開始日期','營業職照開始有效日期',NULL,NULL,1001,'v1'),
('ERP','foo_file','foo1002','營業職照失效開始日期','營業職照開始失效日期',NULL,NULL,1002,'v1'),
('ERP','foo_file','foo1003','所屬業態','所屬業態（產業/業種）',NULL,NULL,1003,'v1'),

('ERP','foo_file','foo1004','狀態碼','客戶狀態','0-開立,1-確認,2-留置,3-停止交易','foo_STATUS',1004,'v1');
```

---

### 4) 把 `foo05 / foo1004` 的代碼字典補齊（讓 AI 可直接推理）

#### 4-1. `foo05` 永久/暫時

```sql
INSERT INTO md_code_dictionary
(domain_name, src_system, src_table, src_column, src_code, tgt_code, tgt_desc_zh)
VALUES
('FOO_CUSTOMER_TYPE','ERP','foo_file','foo05','1','1','永久性'),
('FOO_CUSTOMER_TYPE','ERP','foo_file','foo05','2','2','暫時性');
```

#### 4-2. `foo1004` 狀態碼

```sql
INSERT INTO md_code_dictionary
(domain_name, src_system, src_table, src_column, src_code, tgt_code, tgt_desc_zh)
VALUES
('FOO_STATUS','ERP','foo_file','foo1004','0','0','開立'),
('FOO_STATUS','ERP','foo_file','foo1004','1','1','確認'),
('FOO_STATUS','ERP','foo_file','foo1004','2','2','留置'),
('FOO_STATUS','ERP','foo_file','foo1004','3','3','停止交易');
```

---

### 5) 你接下來只要再補兩件事，AI 分析會更「準」

1. **補來源型別**（日期/字串/數字、長度）到 `md_column.data_type`
2. 每個欄位加 `example_values`（JSON array 字串），例如：

```sql
UPDATE md_column
SET example_values = '["C0001","C0002","C1024"]'
WHERE src_system='ERP' AND table_name='foo_file' AND column_name='foo01' AND version='v1';
``` 