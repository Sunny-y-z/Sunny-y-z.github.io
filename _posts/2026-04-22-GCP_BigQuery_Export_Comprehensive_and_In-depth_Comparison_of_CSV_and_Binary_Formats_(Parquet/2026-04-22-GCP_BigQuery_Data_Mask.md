# 1. Industry General Data Masking Implementation Methods and Selection Comparison

## 1.1 Detailed Explanation of Mainstream Masking Technology Solutions

| Masking Technology | Implementation Principle | Core Features |
|-------------------|-------------------------|---------------|
| Masking/Replacement Masking | Fixed replacement of partial characters in sensitive fields (e.g., phone number 138****1234), preserving data format | Simple implementation, zero code threshold, strong format compatibility, most commonly used lightweight masking solution |
| Format-Preserving Encryption (FPE) | Reversible encryption of data based on encryption algorithms, preserving original data format and length | High security, reversible, no impact on business system operation, suitable for core sensitive field protection in production databases |
| Salted Hashing Masking | Irreversible transformation of data through SHA-256 and other hashing algorithms + random salt values | Irreversible, extremely high security, collision-resistant, suitable for password storage, archive data that doesn't need restoration |
| Data Generalization/Truncation | Replacement of specific data values with range values (e.g., age 28 → 20-30, address precise to city level) | Preserves data statistical characteristics, suitable for data analysis, machine learning modeling scenarios |
| Data Permutation/Shuffling | Randomly shuffling sensitive field values within the dataset, preserving data distribution and format | Preserves data business characteristics, suitable for development testing, performance testing scenarios |
| Null/Deletion | Directly nullifying or deleting sensitive fields | Highest security, zero implementation cost, completely loses data value, only suitable for scenarios where the field is not needed |

## 1.2 Comparison of Advantages, Disadvantages and Application Scenarios of Different Solutions

| Masking Technology | Core Advantages | Core Disadvantages | Security Level | Typical Application Scenarios |
|-------------------|----------------|-------------------|---------------|-------------------------------|
| Masking Replacement | Simple implementation, format compatibility, zero code threshold | Medium security, vulnerable to brute force attacks | Medium | Frontend display, report query, low sensitive data masking |
| Format-Preserving Encryption (FPE) | High security, reversible, preserves data format | Requires key management, has performance overhead | High | Production database dynamic masking, business system sensitive field protection |
| Salted Hashing | Irreversible, extremely high security, collision-resistant | Cannot be restored,破坏 data format | Extremely High | Password storage, data deduplication, archive data that doesn't need restoration |
| Data Generalization | Preserves statistical characteristics, supports data analysis | Loses precise data, does not support business transactions | Medium | Data analysis, machine learning modeling, compliant data sharing |
| Data Permutation | Preserves data distribution and format, suitable for testing scenarios | Related fields easily restored, needs to be combined with other solutions | Medium-Low | Development testing environment data supply, performance testing |
| Null Deletion | Highest security, zero implementation cost | Completely loses data value | Extremely High | Archive, obsolete data processing where the field is not needed |

***

# 2. Introduction to GCP Sensitive Data Protection (SDP/DLP) Core Capabilities

## 2.1 What is GCP SDP?

GCP Sensitive Data Protection (SDP, formerly Cloud DLP) is Google Cloud's native enterprise-level data security service, covering the full lifecycle management of sensitive data: **automatic discovery → classification and grading → masking transformation → risk audit**. Natively integrated with the entire GCP ecosystem, it enables unified data protection across GCP services without complex third-party component integration, while supporting sensitive data governance in hybrid cloud and multi-cloud environments. It is a core component of Google Cloud's data security system.

## 2.2 Core Function Modules and Masking Capabilities

- **Sensitive Data Automatic Discovery**: Built-in over 150 predefined sensitive information types (InfoTypes), covering global mainstream PII data (Chinese ID/phone number, bank card, email, passport, medical data, financial data, etc.), supporting custom sensitive types, automatically scanning structured/unstructured data in various GCP services to complete automated classification and grading.
- **Full-Scenario Masking Capabilities**: Built-in all the above mainstream masking technologies, supporting no-code configuration-based masking and API customized development, covering static/dynamic, batch/real-time full scenarios, while supporting regulatory-compliant de-identification and differential privacy capabilities.
- **Risk Assessment and Compliance Audit**: Automatically identifies data leakage risks, generates visual risk reports, natively integrates with GCP Cloud Audit Logs, all operations are traceable, meeting global compliance audit requirements such as GDPR, PIPL, HIPAA.
- **Fine-Grained Rule Management**: Supports reuse of sensitive type configurations and masking rules through templates, unifying enterprise-level data protection standards, adapting to large-scale implementation across multiple businesses and teams.

## 2.3 GCP Ecosystem Native Integration Advantages

- **Deep Integration with BigQuery**: Supports direct invocation of masking capabilities in BigQuery SQL, no data export required, zero-code implementation of query-time masking, also supports batch scanning and masking of entire tables/datasets, the most commonly used lightweight masking solution in GCP.
- **Integration with Dataflow**: Natively embeds DLP masking capabilities in streaming/batch data processing pipelines, realizing real-time masking during data lake/warehouse ingestion, adapting to TB/PB-level enterprise-scale data processing scenarios.
- **Integration with Cloud Storage**: Supports sensitive data scanning and masking of unstructured data such as CSV, JSON, images, documents in object storage, covering all types of data assets.
- **Integration with Cloud IAM**: Based on GCP's unified identity permission system,实现 fine-grained access control, following the principle of least privilege, ensuring the security of masking operations.

***

# 3. GCP SDP Data Masking Full-Process Practical Operation

## 3.1 Pre-environment and Permission Preparation

### 3.1.1 GCP Account and Project Creation

`Note: All operations must use the same location.`

Prerequisites:
- GCP account (with project creation, IAM permission configuration, billing permissions)
- Optional: gcloud CLI (for quick verification of GCP configuration)

(1) Create GCP Project
- Log in to the [GCP Console](https://console.cloud.google.com/), click "Select Project" at the top → "New Project".
- Enter project name (e.g., gcp-dlp-demo), automatically generate project ID (write it down, use it throughout), click "Create".
- Wait for project creation to complete (about 1 minute), switch to the project.

### 3.1.2 Enable Core API Services

Core APIs to enable: Sensitive Data Protection API, BigQuery API, Cloud Dataflow API (optional for enterprise solutions).

- In the GCP Console left navigation bar, enter "API and Services" → "Library";
- Search for the above APIs respectively, click into the details page, then click "Enable";
- After enabling, verify in "API and Services" → "Enabled APIs and Services".

Enablement success example:

![](/assets/images/20260422/Sensitive_Data_Protection_API.png)

### 3.1.3 Service Account Creation and Key Configuration

Service accounts are used for code calls to DLP API, BigQuery and other services, and need to be configured following the principle of least privilege.

Service account creation:

- Home page → Quick access → IAM & Admin → Service Accounts → Create service account
- Configuration:
  - Service account name: bq-dlp-service-account
  - Service account ID: automatically generated (no modification needed)
  - Description: Service account for DLP masking of BigQuery data
- Click "Create and Continue", add the following roles (principle of least privilege):
  - DLP Administrator (DLP operations)
  - BigQuery Data Editor (BigQuery data read/write)
  - BigQuery Job User (execute BigQuery jobs)
  - Dataflow Admin (optional): Dataflow task management permissions
- Click "Continue" → "Done".

Key configuration:

- Find the newly created service account, click the right "⋮" → "Manage Keys".
- Switch to the "Keys" tab → "Add key" → "Create new Key".
- Select JSON as the key type, click "Create", the key file will be automatically downloaded (e.g., bq-dlp-demo-xxxx.json).
- Save the key file: It is recommended to place it in the secrets/ folder at the project root (configure .gitignore later to avoid submission).

### 3.1.4 Test Environment Preparation: BigQuery Dataset and Sensitive Test Table Construction

We will create a BigQuery test table containing common sensitive data for subsequent scanning and masking operations.

(1) Create Dataset
- Navigate to BigQuery → "Studio" → click "Create Dataset" next to the project ID.
- Configuration parameters:
  - Dataset ID: test_dataset (only letters/numbers/underscores)
  - Location: europe-west2 (London, must be consistent with subsequent DLP operation regions)
  - Keep others default, click "Create".

(2) Create Test Table and Insert Sensitive Data
- Click "Create Table" next to test_dataset, select "Empty Table":
  - Table ID: sensitive_user_data
  - Schema configuration (add the following fields):
    | Name | Type | Mode |
    |------|------|------|
    | id | INTEGER | NULLABLE |
    | name | STRING | NULLABLE |
    | phone | STRING | NULLABLE |
    | id_card | STRING | NULLABLE |
    | email | STRING | NULLABLE |
    | description | STRING | NULLABLE |
- Click "Create Table", then execute the following SQL to insert test data:

```sql
INSERT INTO `[Your Project ID].test_dataset.sensitive_user_data`
(id, name, phone, id_card, email, description)
VALUES
(1, 'Zhang San', '13800138000', '110101199001011234', 'zhangsan@example.com', 'Useless backup information'),
(2, 'Li Si', '13900139000', '310101198505156789', 'lisi@example.com', 'Related description information');

-- Delete data
DELETE `[Your Project ID].test_dataset.sensitive_user_data` WHERE 1=1;
```

- Verification:
  - Method 1: View data through Preview
  - Method 2: Execute SELECT * FROM [Your Project ID].test_dataset.sensitive_user_data to confirm data insertion success.

## 3.2 Step 1: Sensitive Data Automatic Discovery and Classification

The core premise of masking is **finding sensitive data first, then performing targeted masking**. In enterprise-level scenarios, data assets are large in scale and have many fields, making it impossible to manually enumerate sensitive fields. SDP is needed to realize automated sensitive data discovery.

### 3.2.1 Create Data Discovery Scanning Task Based on SDP

- Enter GCP Console → Sensitive Data Protection, open the SDP console;
- Sensitive Data Protection → Learn about your data → Deep inspection → Create job and job triggers, start configuring the scanning task;
- Step 1: Name, location, storage system (scanned storage), scope, enter scanning name dlp-bigquery-demo-scan, select corresponding region, select corresponding storage system, select data scope (default 1000 rows), click Continue;
- Step 2: Configure detection settings, can select "All predefined InfoTypes", or customize rules (through Manage infoTypes or Inspection rulesets -> Add a ruleset), confidence threshold defaults to POSSIBLE, click Continue;
- Step 3: Configure result storage, click Continue;
- Step 4: Scheduling settings, select run once immediately for test environment, configure periodic scheduling for production environment, click Continue;
- Step 5 (optional): Preview all configurations;
- After all configurations are completed, click Create, the scanning task starts immediately.

Step 1 Example
![Step 1 Example](/assets/images/20260422/step1_example.png)

Step 2 Example
![Step 2 Example](/assets/images/20260422/step2_example.png)

Step 3 Example
![Step 3 Example](/assets/images/20260422/step3_example.png)

### 3.2.2 Sensitive Data Identification Result Verification and Label Management

- Wait for the scanning task to complete (small data volumes take tens of seconds), click the task name in the scanning list to view scanning results;
- The results page shows sensitive fields automatically identified by SDP, corresponding sensitive types, matching counts, and confidence, such as phone_number identified as Chinese phone number, id_card identified as Chinese ID number;
- Based on identification results, sensitive fields can be classified and graded, and corresponding masking rules can be formulated. High-sensitive fields use high-strength masking solutions, while medium-low sensitive fields use lightweight masking solutions.

Note: If default rules are used for scanning, due to overlapping matching of multiple rules, the same field may be repeatedly marked in the scanning results.

## 3.3 Step 2: Implementation of 3 Core Masking Solutions in GCP Ecosystem

### 3.3.1 Solution 1: BigQuery Native SQL Zero-Code Masking (Lightweight Scenarios)

This solution requires no additional development, no data export, directly completes masking in BigQuery SQL, does not modify original data, only masks query results, adapting to temporary queries, report display, lightweight analysis and other scenarios. It is the most commonly used lightweight masking solution in GCP.

#### 3.3.1.1 Character Replacement/Masking Implementation

Through BigQuery string functions, partial characters of sensitive fields are replaced, preserving data format, adapting to most display scenarios.

Direct character replacement through BigQuery SQL, rule explanation:
- Phone number: Keep first 3 digits and last 4 digits, replace middle 4 digits with * (e.g., 13812345678 → 138****5678)
- ID card: For 18 digits, keep first 6 digits and last 4 digits, replace middle 8 digits with *; for 15 digits, keep first 6 digits and last 3 digits, replace middle 6 digits with *
- Email: Keep first character before @, replace remaining characters with *, keep @ and domain name complete (e.g., abc123@xxx.com → a*****@xxx.com)

```sql
SELECT
  id, -- No processing
  name, -- No processing
  -- Phone number masking: keep first 3 and last 4, middle 4 *
  CASE 
    WHEN LENGTH(TRIM(phone)) = 11 THEN CONCAT(SUBSTR(phone, 1, 3), '****', SUBSTR(phone, 8, 4))
    ELSE phone  -- Non-11-digit phone numbers not processed (can be adjusted according to needs)
  END AS masked_phone,
  
  -- ID card masking: distinguish 18-digit/15-digit
  CASE
    WHEN LENGTH(TRIM(id_card)) = 18 THEN CONCAT(SUBSTR(id_card, 1, 6), '********', SUBSTR(id_card, 15, 4))
    WHEN LENGTH(TRIM(id_card)) = 15 THEN CONCAT(SUBSTR(id_card, 1, 6), '******', SUBSTR(id_card, 13, 3))
    ELSE id_card  -- Non-standard length not processed
  END AS masked_id_card,
  
  -- Email masking: keep first character before @, replace rest with *
  CASE
    WHEN REGEXP_CONTAINS(email, '@') THEN 
      CONCAT(
        SUBSTR(SPLIT(email, '@')[OFFSET(0)], 1, 1),  -- Take first character before @
        REPEAT('*', LENGTH(SPLIT(email, '@')[OFFSET(0)]) - 1),  -- Replace remaining characters before @ with *
        '@',
        SPLIT(email, '@')[OFFSET(1)]  -- Keep domain name after @
      )
    ELSE email  -- Non-standard email format not processed
  END AS masked_email,
  description -- No processing
FROM
  `[Your Project ID].test_dataset.sensitive_user_data`;  -- Replace with your project, dataset, table name
```

Key function explanations:
- SUBSTR(str, start_pos, length): Extracts string, start_pos starts from 1 (BigQuery string index is not 0-based)
- CONCAT(str1, str2, ...): Concatenates multiple strings
- LENGTH(TRIM(str)): First removes leading and trailing spaces from string, then calculates length (avoiding length judgment errors caused by spaces)
- SPLIT(str, delimiter): Splits string by delimiter, [OFFSET(0)] takes first split result, [OFFSET(1)] takes second
- REPEAT('*', n): Generates n consecutive *, more flexible than manually writing multiple *
- REGEXP_CONTAINS(str, pattern): Determines whether string contains specified regular expression (used here to check if email has @)

Notes:
- If the field is NULL or empty string, the above logic will directly return the original value, you can add IS NULL judgment according to needs (e.g., WHEN phone IS NULL THEN '')
- If phone number / ID card contains non-numeric characters (such as spaces, letters), need to clean first through REGEXP_REPLACE (e.g., REGEXP_REPLACE(phone, '[^0-9]', ''))
- Masking rules can be flexibly adjusted: for example, if you want to keep first 2 digits of email, just change SUBSTR(SPLIT(email, '@')[OFFSET(0)], 1, 1) to SUBSTR(..., 1, 2), and reduce the REPEAT length by 2.
- It is recommended to clean fields before masking (remove spaces, filter non-target characters) to avoid masking failure due to length judgment errors.

`Advantages:`
- **Lightweight and easy to implement**: Based on BigQuery native string functions (SUBSTR/CONCAT/REPEAT, etc.), no dependency on third-party libraries or custom functions, beginners can quickly get started, SQL writing and maintenance costs are low.
- **Flexible and controllable rules**: Can precisely adjust masking digits (e.g., change phone number to keep first 2 and last 3, email keep first 2 digits), adapt to different business masking requirements, and support compatibility processing of non-standard data (such as non-11-digit phone numbers, emails without @).
- **High performance**: Only involves string extraction/concatenation/length calculation, no complex logic, under BigQuery distributed architecture, can efficiently process tens of millions or even hundreds of millions of scale datasets.
- **Good compatibility**: Naturally adapts to BigQuery's SQL syntax, no additional environment configuration required, can be directly embedded in queries, views, ETL tasks.

`Disadvantages:`
- **Weak security**: Masking rules are fixed (e.g., phone number must keep first 3 and last 4), no randomness, does not meet high compliance requirements.
- **Limited coverage of abnormal scenarios**: Does not handle extreme abnormal data (such as ID cards with special characters, emails with multiple @, phone numbers with letters), additional cleaning logic (such as regular filtering of non-numeric characters) is needed.
- **No unified management capability**: Masking rules are hard-coded in SQL, if enterprises need to uniformly modify rules (such as adjusting ID card masking digits company-wide), need to modify related SQL one by one, maintenance cost increases with the increase of scenarios.
- **Irreversible but non-encryption level**: Only character replacement, not encryption masking (such as hashing/masking), cannot meet compliance requirements of highly regulated industries such as finance and medical care.

`Applicable scenarios:`
- **Internal non-core data processing**: Such as internal operational reports, business analysis, data visualization and other scenarios, only need basic masking to protect privacy, no strict compliance certification required.
- **One-time/temporary data masking**: Such as exporting data to third-party partners, temporarily querying masked data, quick implementation without building complex masking systems.
- **Medium-small scale/fixed rule scenarios**: Dataset scale is moderate, masking rules remain unchanged for a long time (such as fixed phone number keeping first 3 and last 4), no need for dynamic adjustment of masking level.

`Inapplicable scenarios:`
- **High compliance requirement scenarios**: Finance, medical, government and other fields that need to comply with strong regulatory requirements such as "Personal Information Protection Law", need to adopt random masking, encryption masking, dynamic masking and other solutions.
- **Enterprise-level unified masking management**: Need cross-team, cross-project unified control of masking rules, or need to dynamically adjust masking level (such as administrators viewing full amount, ordinary employees viewing masked data).
- **Scenarios with high proportion of extreme abnormal data**: Such as phone numbers/ID cards with a lot of special characters, chaotic formats, need complex cleaning + dynamic rules.

`Summary:`
- This method is a **lightweight, efficient, low-cost** basic masking solution in BigQuery, core adapting to internal analysis, temporary processing and other low compliance requirement scenarios;
- Its core shortcoming is **insufficient security due to static rules**, and no unified management capability, not adapting to high compliance, enterprise-level unified control scenarios;

#### 3.3.1.2 Built-in Encryption Function Strong Masking Implementation

[Reference documentation: https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/dlp_functions](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/dlp_functions)

For highly sensitive data, BigQuery's built-in DLP_DETERMINISTIC_ENCRYPT encryption function can be used to implement reversible strong masking, while supporting fine-grained key permission control, only users with key permissions can restore original data. This method belongs to deterministic encryption, with higher security and support for reversible decryption, while retaining data关联性 (same plaintext encrypts to same ciphertext).

Core premise and background:
DLP_DETERMINISTIC_ENCRYPT is an encryption function provided by BigQuery in conjunction with Google Cloud DLP (Data Loss Prevention) and KMS (Key Management Service), with core features:
- **Deterministic encryption**: Same plaintext + same key + same context → same ciphertext (can be used for grouping, association analysis);
- **Reversibility**: Can decrypt and restore original value through corresponding key (requires strict key permission control);
- **High security**: Depends on Google Cloud KMS symmetric encryption keys, meets compliance requirements (such as GDPR, Personal Information Protection Law);
- **Permission control**: Need to configure KMS key access permissions, only authorized accounts can encrypt/decrypt.

(1) Prerequisites (must be completed)
Before using this function, you need to create Cloud KMS keys, steps as follows:
- **Create KMS Key Ring**:
  - Enter Google Cloud Console → search for "Key Management Service"
    ![KMS Service Search](/assets/images/20260422/kms_example1.png)
  - Enable the service (note: cost issue!), after enabling, it will show API Enabled status
  - Enter the management page (first time开通 service, need to wait a few minutes), select region (must be consistent with BigQuery dataset region) → create key ring (Create key ring, such as dlp-key-ring).
    ![Create Key Ring](/assets/images/20260422/kms_example2.png)
- **Create symmetric encryption key**:
  - Create **Symmetric encrypt/decrypt** under the key ring (algorithm default cannot be changed Google symmetric key), name such as dlp-deterministic-key.
  - `Note: Expiration time.` Same plaintext, using different keys, encrypted ciphertext will be different.
- **Authorize BigQuery to access KMS key**:
  - Grant Cloud KMS CryptoKey Encrypter/Decrypter role (allow encryption/decryption) to BigQuery service account (format: service-PROJECT_NUMBER@gcp-sa-bigquery.iam.gserviceaccount.com).

(2) DLP_DETERMINISTIC_ENCRYPT Function Syntax

According to BigQuery official documentation, DLP_DETERMINISTIC_ENCRYPT has the following two forms (core difference is whether optional context is included):

```sql
-- Form 1: Only required parameters (key + plaintext + surrogate)
DLP_DETERMINISTIC_ENCRYPT(
  key,         -- Required: KMS key complete path
  plaintext,   -- Required: sensitive string to be encrypted
  surrogate    -- Required: replacement value generation rule
)

-- Form 2: Required parameters + optional context
DLP_DETERMINISTIC_ENCRYPT(
  key,         -- Required: KMS key complete path
  plaintext,   -- Required: sensitive string to be encrypted
  surrogate,   -- Required: replacement value generation rule
  context      -- Optional: context string, enhance encryption uniqueness
)
```

Parameter detailed explanation (aligned with official definition)

| Parameter Name | Type | Required | Core Meaning & Constraints (official requirements) |
|---------------|------|----------|--------------------------------------------------|
| key           | STRING | Yes | KMS symmetric encryption key complete path, fixed format: projects/PROJECT_ID/locations/REGION/keyRings/KEY_RING/cryptoKeys/KEY_NAME; only supports keys with GOOGLE_SYMMETRIC_ENCRYPTION algorithm |
| plaintext     | STRING | Yes | Sensitive field to be encrypted (phone number/ID card/email), **only supports string type**; if the field is numeric (such as INT64 type phone number), need to convert to string through CAST first |
| surrogate     | STRING | Yes | 【Core required】Replacement value generation rule, controls encrypted ciphertext format: <br>- Set to NULL: output original encrypted bytes (poor readability, not recommended); <br>- Set to custom prefix (such as 'PHONE'): generate unique replacement value identified by this prefix (recommended); <br>- Only allows letters, numbers, underscores ([a-zA-Z0-9_]+) |
| context       | STRING | No | Context string, used to enhance encryption uniqueness: same plaintext + different context → different ciphertext; recommended to set as field name (such as 'phone'), avoid cross-field ciphertext conflict |

(3) Correct Practical Example (Encryption)

Generate DEK and encrypt to get wrapped_key (using gcloud command line)
Open Google Cloud Shell (or install gcloud CLI locally), execute the following commands (replace your project / key information):

```shell
# I. Generate random DEK (32 bytes, conforming to AES-256 standard)
DEK=$(openssl rand 32 | base64)

# II. Encrypt DEK with KMS key, get wrapped_key (Base64 format)
WRAPPED_KEY=$(echo -n $DEK | base64 -d | gcloud kms encrypt \
  --project=your-project-id \
  --location=europe-west2 \
  --keyring=dlp-key-ring \
  --key=dlp-deterministic-key \
  --plaintext-file=- \
  --ciphertext-file=- | base64)

# III. Output WRAPPED_KEY (copy this value, will be used in subsequent SQL)
echo $WRAPPED_KEY
```

Encryption (masking) example

```sql
-- Replace with your KMS key path, project/dataset/table name
DECLARE kms_key_path STRING DEFAULT 'gcp-kms://projects/your-project-id/locations/your_datase_locations/keyRings/dlp-key-ring/cryptoKeys/dlp-deterministic-key';

DECLARE DLP_KEY_VALUE BYTES;

SET DLP_KEY_VALUE =
  FROM_BASE64('wrapped_key');

SELECT
  id,
  name,

  -- Phone number encryption (complete parameters: key + plaintext + surrogate + context)
  DLP_DETERMINISTIC_ENCRYPT(
    -- Step 1: Use DLP_KEY_CHAIN to generate compliant key structure
    DLP_KEY_CHAIN(
      kms_key_path,  -- kms_resource_name
      DLP_KEY_VALUE  -- wrapped_key (set to NULL for ordinary scenarios)
    ),
    TRIM(CAST(phone AS STRING)),  -- Convert to string and remove spaces
    'PHONE_SURROGATE',            -- Replacement value prefix (identifies phone number ciphertext)
    'phone'                       -- Context: field name
  ) AS encrypted_phone,

  -- ID card encryption (complete parameters)
  DLP_DETERMINISTIC_ENCRYPT(
    -- Step 1: Use DLP_KEY_CHAIN to generate compliant key structure
    DLP_KEY_CHAIN(
      kms_key_path,  -- kms_resource_name
      DLP_KEY_VALUE  -- wrapped_key (set to NULL for ordinary scenarios)
    ),
    TRIM(id_card),
    'ID_CARD_SURROGATE',
    'id_card'
  ) AS encrypted_id_card,

  -- Email encryption (only required parameters, omit context)
  DLP_DETERMINISTIC_ENCRYPT(
    -- Step 1: Use DLP_KEY_CHAIN to generate compliant key structure
    DLP_KEY_CHAIN(
      kms_key_path,  -- kms_resource_name
      DLP_KEY_VALUE  -- wrapped_key (set to NULL for ordinary scenarios)
    ),
    TRIM(email),
    'EMAIL_SURROGATE'  -- Only required parameters: key + plaintext + surrogate
  ) AS encrypted_email,

  description
FROM
  `your-project.your-dataset.user_info`;
```

(4) Correct Practical Example (Decryption - Restore Original Value)

The parameters of the decryption function DLP_DETERMINISTIC_DECRYPT must **exactly match** those during encryption (key, surrogate, context are all required):

```sql
-- Replace with your KMS key path, project/dataset/table name
DECLARE kms_key_path STRING DEFAULT 'gcp-kms://projects/your-project-id/locations/your_datase_locations/keyRings/dlp-key-ring/cryptoKeys/dlp-deterministic-key';

DECLARE DLP_KEY_VALUE BYTES;

SET DLP_KEY_VALUE =
  FROM_BASE64('wrapped_key');

SELECT
  phone,
  -- Decrypt phone number (match surrogate and context during encryption)
  DLP_DETERMINISTIC_DECRYPT(
    DLP_KEY_CHAIN(
      kms_key_path,  -- kms_resource_name
      DLP_KEY_VALUE  -- wrapped_key (set to NULL for ordinary scenarios)
    ),
    'PHONE_ciphertext',
    'PHONE_SURROGATE',
    'phone'
  ) AS decrypted_phone,
FROM
  `your-project.your-dataset.encrypted_user_info`
where id = 1;
```

(5) Key Notes

- **Strict permission control**: The account performing encryption needs to have cloudkms.cryptoKeyVersions.useToEncrypt permission for KMS keys, and decryption needs cloudkms.cryptoKeyVersions.useToDecrypt permission, which needs to be configured in Google Cloud IAM.

(6) Advantages and Disadvantages Analysis and Applicable Scenarios

`Advantages:`
- **Preserves equality query/association capability (core advantage)**: The core feature of deterministic encryption is "same plaintext generates same ciphertext", which means encrypted sensitive data can still support WHERE equality query, GROUP BY grouping, JOIN table association and other operations. For example, after encrypting phone numbers, you can still query "whether a certain phone number exists", "group by encrypted user ID to count order volume", which is impossible with hashing (irreversible) and random encryption (same plaintext different ciphertext).
  
- **Seamless integration with BigQuery, low use cost**: Directly called in BigQuery SQL, no additional ETL process or external tools needed, such as directly encrypting table columns, friendly to beginners, low development and maintenance costs.

- **Balances compliance and reversibility**
  - Complies with GDPR, CCPA, PCI DSS and other compliance requirements, effectively masks sensitive data such as phone numbers, ID cards, bank cards;
  - Can be decrypted through DLP_DETERMINISTIC_DECRYPT function (requires key permission), meeting scenarios where "original data needs to be restored when necessary" (such as customer service verifying user information).

- **Complete key management**: Integrates with Google Cloud KMS (Key Management Service), supports key rotation, fine-grained permission control (such as only allowing specific roles to decrypt), conforms to enterprise-level security best practices, avoids the risk of key leakage/loss (compared to manual key management).

`Disadvantages:`
- **Lower security than random encryption (core risk)**: Due to "same plaintext → same ciphertext", attackers can crack through "frequency analysis": for example, high-frequency ciphertext likely corresponds to high-frequency plaintext, there is dictionary attack risk, security is much lower than random encryption (AEAD).

- **Significant performance overhead**: Encryption/decryption operations increase BigQuery query latency, especially when operating on large-scale datasets (100 million rows), query time may increase by 20%-50%, affecting performance of real-time analysis scenarios.

- **Data type and usage limitations**: Only supports STRING, BYTES types, complex types such as STRUCT, ARRAY need to be disassembled first; and encrypted ciphertext length is fixed (about 64 bytes), which will increase storage costs.

- **Strong key dependency**: Encryption and decryption must use the same key: key leakage will lead to all encrypted data being cracked, permanent key loss will make data irrecoverable, requiring high key management processes.


`Applicable scenarios:`

- **Sensitive data encryption that needs to preserve equality queries**
  - Typical scenarios: E-commerce order table user phone number/ID card encryption (need to query "whether a user has orders"), financial customer table bank card number encryption (need to group by card number to count transactions), medical data patient ID encryption (need to associate patient medical records and examination reports).

- **Balancing compliance masking and business analysis**: Need to meet compliance requirements to mask sensitive data, but cannot completely lose the analysis value of data (such as counting user activity by encrypted email, screening high-value users by encrypted phone number).

- **Cross-table/cross-project data association**: Encrypted ciphertext maintains consistency across different BigQuery datasets/projects, can be used for equality association across tables/projects (such as user table and transaction table association through encrypted user ID).

- **Reversible masking requirements**: No need for permanent irreversible masking (such as salted hashing), need to decrypt original data after authorization (such as bank customer service verifying customer phone numbers, e-commerce after-sales querying user addresses).

`Inapplicable scenarios:`

- **Highly sensitive data without equality query needs**: Such as passwords, payment verification codes, login tokens, should use hashing (with salt) or random encryption to avoid frequency analysis risks.
- **Real-time high-performance query scenarios**: Such as real-time risk control, real-time reports, the performance loss caused by encryption will affect result return speed.
- **Scenarios without key management capability**: If the team cannot guarantee Cloud KMS key security (such as permission confusion, keys not rotated), it is not recommended to use.

`Summary`
- Core advantage: **preserves equality query/association capability**, integrates with BigQuery with high ease of use, balances compliance masking and data analyzability;
- Core risk: **lower security than random encryption**, exists frequency analysis risk, relies on key security management;
- Applicable core scenarios: need to encrypt sensitive data, and must preserve equality query, grouping, association capability business scenarios. Highly sensitive, no equality operation or real-time high-performance requirement scenarios should avoid using.

### 3.3.2 Solution 2: DLP API + Code Implementation for Flexible Masking (Custom Scenarios)

This solution adapts to custom masking needs, can be integrated into business systems, data pipelines, unified management of masking rules, supports cross-service calls, adapts to structured/unstructured data masking, this article takes Python as an example.

(1) Prerequisites

- Install dependency libraries

```bash
pip install google-cloud-dlp google-cloud-bigquery
```

- Configure environment variables, specify service account key path

```bash
# Linux/Mac
export GOOGLE_APPLICATION_CREDENTIALS="./key.json"

# Windows PowerShell
$env:GOOGLE_APPLICATION_CREDENTIALS="./key.json"
```

(2) Complete Masking Code Example example_dlpapi_bigquery_data_masking.py

```python
from google.cloud import dlp_v2
from google.cloud import bigquery

# Initialize client
dlp_client = dlp_v2.DlpServiceClient(
    # Use http protocol when DLP API is used in environments that do not support gRPC
    # transport="rest"
)
bq_client = bigquery.Client()

# Core configuration parameters
PROJECT_ID = "Your GCP Project ID"
DATASET_ID = "test_dataset"
TABLE_ID = "sensitive_user_data"
LOCATION = "global"

def deidentify_with_mask(project_id, table_data):
    """Implement masking based on DLP API"""
    parent = f"projects/{project_id}/locations/{LOCATION}"

    # Configure sensitive information types to be identified
    info_types = [
        {"name": "EMAIL_ADDRESS"},
        {"name": "PERSON_NAME"}
    ]

    # Configure masking rules: masking replacement
    deidentify_config = {
        "info_type_transformations": {
            "transformations": [
                {
                    "info_types": info_types,
                    "primitive_transformation": {
                        "character_mask_config": {
                            "masking_character": "*",
                            "number_to_mask": 8,
                            "reverse_order": False
                        }
                    }
                }
            ]
        }
    }

    # Configure detection rules
    inspect_config = {
        "info_types": info_types,
        "min_likelihood": dlp_v2.Likelihood.POSSIBLE,
        "include_quote": True
    }

    # Construct input data format
    rows = []
    headers = [{"name": col} for col in table_data[0].keys()]
    for row in table_data:
        values = [{"string_value": str(val)} for val in row.values()]
        rows.append({"values": values})

    table = {"headers": headers, "rows": rows}
    item = {"table": table}

    # Call DLP API to perform masking
    response = dlp_client.deidentify_content(
        request={
            "parent": parent,
            "deidentify_config": deidentify_config,
            "inspect_config": inspect_config,
            "item": item
        }
    )

    # Parse masking results
    deidentified_rows = []
    for row in response.item.table.rows:
        row_data = {}
        for i, header in enumerate(headers):
            row_data[header["name"]] = row.values[i].string_value
        deidentified_rows.append(row_data)

    return deidentified_rows

if __name__ == "__main__":
    # Read original data from BigQuery
    query = f"""
    SELECT * FROM `{PROJECT_ID}.{DATASET_ID}.{TABLE_ID}` LIMIT 100
    """
    query_job = bq_client.query(query)
    table_data = [dict(row) for row in query_job.result()]

    # Execute masking and print results
    deidentified_result = deidentify_with_mask(PROJECT_ID, table_data)
    print("Masked results:")
    for row in deidentified_result:
        print(row)
```

Code explanation: By modifying deidentify_config configuration, encryption, hashing, generalization and other masking rules can be switched, and masking results can be written back to BigQuery or other storage systems to achieve batch masking processing.

`Advantages`

- Extremely high flexibility and customization capability: Can completely customize masking rules, support complex business process embedding, can customize sensitive information types (InfoType).
- Full-scenario data type support: Not only supports structured data (BigQuery tables, databases, CSV/JSON files), but also supports sensitive data identification and masking of unstructured data (text, images, documents), covering all types of enterprise data assets; can be integrated into hybrid cloud, multi-cloud environment data pipelines, not limited to GCP ecosystem, adapting to cross-cloud data governance needs.
- Unified rule management and large-scale implementation: SDP templates can be used to reuse masking rules and sensitive type configurations, multiple business lines and teams share a set of enterprise-level data protection standards, avoiding rule dispersion and inconsistency; supports batch processing of data through code, adapting to periodic, automated masking tasks, no manual intervention required.
- Fine-grained permission and audit support: Based on GCP IAM to implement fine-grained API access permission control, following the principle of least privilege; all API call operations are recorded in Cloud Audit Logs, full process traceable, meeting compliance audit requirements.

`Disadvantages`
- High development and maintenance costs: Requires teams with certain development capabilities (Python/Java/Go, etc.) for code writing, debugging and maintenance, zero technical background business teams cannot directly use; when business rules change, code needs to be modified synchronously and re-released, iteration cycle is long, maintenance cost increases with business complexity.
- Performance and latency overhead: Each masking needs to call DLP API through network, there is fixed network latency and API response time, not suitable for real-time business scenarios with extremely high latency requirements (millisecond level); during large-scale batch data processing, API call quota may become a bottleneck, need to apply for quota increase in advance, or avoid task failure through batch processing and flow control.
- Security and complexity risks: Service account keys need to be properly managed, if keys are leaked, API may be abused and sensitive data leaked; when code logic is complex, bugs may be introduced leading to incomplete masking or data damage, requiring sufficient testing and code review.
- Uncontrollable costs: DLP API charges based on call volume and data processing volume, costs may rise rapidly during large-scale data processing, need to do cost monitoring and budget management; human costs generated during development, testing and maintenance also need to be included in overall consideration.

`Applicable scenarios`

- Business system integration with high customization requirements
  - Typical scenarios: Business systems need to dynamically adjust masking rules based on user roles and business scenarios when users submit data and interfaces return data;
  - Core value: Embed masking capabilities into business logic through code, realizing "business-driven dynamic masking", not limited by fixed templates.

- Cross-service, cross-cloud data pipeline masking
  - Typical scenarios: When data is synchronized from local IDC/other cloud platforms to GCP, or flows between multiple GCP services such as Pub/Sub, Dataflow, BigQuery, need to complete masking at the flow node;
  - Core value: The lightweight call characteristics of API make it easy to embed into any data processing node, realizing "data non-landing masking".

- Unstructured data masking
  - Typical scenarios: Identify and mask sensitive text in Cloud Storage contract documents, customer service chat records, images;
  - Core value: GCP SDP's support for unstructured data, combined with code flexibility, can realize automated processing of complex unstructured data.

- Multi-team, multi-business line large-scale implementation
  - Typical scenarios: Enterprises need to unify masking rules across business lines to avoid "each doing their own thing";
  - Core value: Through template reuse + code encapsulation, package masking capabilities as internal enterprise services for multi-team calls, unified standards and audits.

`Inapplicable scenarios`

- Lightweight temporary query and report display
  - Typical scenarios: Analysts temporarily query BigQuery data, need to quickly mask results;
  - Alternative: Directly use BigQuery SQL zero-code masking, no development required, more efficient.

- Zero technical background business team independent operation
  - Typical scenarios: Business teams need to regularly mask test data, but no development resources;
  - Alternative: Dataflow built-in templates, zero-code completion of operations.

- Real-time business scenarios with extremely high latency requirements
  - Typical scenarios: Core trading systems return user data in real-time, requiring millisecond-level latency;
  - Alternative: Use format-preserving encryption (FPE) to complete masking at the database layer, or dynamic masking products, avoiding network API call latency.

- Ultra-large-scale batch data low-cost rapid processing
  - Typical scenarios: One-time masking of PB-level historical data, limited budget;
  - Alternative: Use Dataflow + DLP batch templates, fully managed automatic processing, no need to write complex code, and can reduce costs through off-peak operation.

`Summary`

- DLP API + code implementation for flexible masking is an "flexibility-first" enterprise-level masking solution, core value lies in "customization capability" and "full-scenario adaptation", but requires "development cost" and "maintenance cost" as a price.
- The best positioning of this solution is: as a "supplementary solution" for enterprise masking systems — when zero-code/low-code solutions such as BigQuery SQL and Dataflow templates cannot meet business needs, implement complex, customized masking logic through API + code; at the same time, it can be encapsulated as an internal enterprise service for multi-team reuse, balancing "flexibility" and "scale".
- For enterprises with development capabilities and customized masking rule requirements, this solution is the core choice in the GCP ecosystem; but for lightweight scenarios and teams with zero development resources, it is recommended to prioritize simpler alternative solutions.

## 3.4 Step 3: Masking Effect Verification and Compliance Audit

(1) Masking Effect Verification

- Compare original data with masked data, confirm that sensitive fields have been masked according to rules, no complete sensitive information leakage;
- Verify business compatibility of masked data, such as whether format is preserved, whether it can be normally used for statistical analysis, testing and other scenarios;
- Verify permission control of reversible masking, confirm that only authorized users can restore original data, unauthorized users cannot decrypt.

(2) Compliance Audit

- Full operation log traceability: All SDP scanning, masking, API call operations are recorded in Cloud Audit Logs, can be viewed and exported in **Operation Audit** → **Log Viewer**;
- Compliance report export: Export sensitive data scanning reports, masking rule configurations, operation logs for compliance audits, meeting regulatory requirements;
- Risk alert configuration: Configure automatic alerts for sensitive data leakage risks, unauthorized masking operations, timely response to security events.

***

# 4. Implementation Best Practices and Pitfall Avoidance Guide

## 4.1 Masking Solution Selection Recommendations for Different Business Scenarios

| Business Scenario | Recommended Solution | Core Selection Basis |
|-------------------|---------------------|----------------------|
| Temporary query, report display, lightweight analysis | BigQuery SQL zero-code masking | Zero threshold, quick implementation, no modification to original data, no additional cost |
| Business system integration, customized masking rules, cross-service calls | DLP API + code implementation | High flexibility, customizable, unified management of masking rules |
| TB-level offline data warehouse batch masking | Dataflow + DLP batch masking | Fully managed, highly scalable, supports large-scale data, no infrastructure management required |
| Real-time data warehouse, streaming data lake ingestion, business real-time interfaces | Dataflow + DLP streaming masking + dynamic masking | Low latency, high concurrency, real-time effect, no modification to original data |
| High-sensitive data storage, core business system field protection | Format-preserving encryption (FPE) + dynamic masking | High security, reversible, preserves data format, no impact on business operation |

## 4.2 Core Techniques for Balancing Performance and Availability

- **Principle of least privilege**: Assign minimum necessary permissions to service accounts and users, strictly control access permissions for keys and encryption key sets, prohibit excessive authorization;
- **Graded masking**: Match corresponding masking solutions according to data sensitivity levels, use high-strength encryption for high-sensitive data, use masking for medium-low sensitive data, balance security and performance;
- **Off-peak operation**: Batch masking tasks are run during business off-peak periods to avoid occupying production resources and affecting business stability;
- **Rule reuse**: Reuse sensitive type configurations and masking rules through SDP templates, unify enterprise-level masking standards, avoid repeated configuration;
- **Mask before storage**: Complete masking before data enters the lake/warehouse, avoid sensitive data landing, reduce leakage risks from the source;
- **Sampling scanning**: For sensitive data discovery of ultra-large-scale datasets, use sampling scanning to reduce costs and time, improve efficiency.

## 4.3 Common Pitfall Problems and Solutions

- **Problem**: DLP API call quota exceeded, masking task failed
  **Solution**: Apply for DLP API quota increase, batch tasks processed in batches, use Dataflow built-in DLP integration to automatically handle throttling.
- **Problem**: BigQuery SQL masking rules not unified, data leakage risk exists
  **Solution**: Unified configuration of masking rules through BigQuery authorized views, users can only access authorized views, direct access to original tables is prohibited.
- **Problem**: Masked data破坏 format, causing business systems/test environments to be unusable
  **Solution**: Adopt format-preserving encryption, masking replacement and other format-preserving masking solutions, test masked data compatibility in advance.
- **Problem**: Low accuracy of sensitive data identification, missed identification/misidentification
  **Solution**: Adjust scanning confidence threshold, customize InfoType to adapt to enterprise internal data formats, combine manual verification to improve accuracy.
- **Problem**: Improper key management, causing encrypted data to be unrecoverable or keys to be leaked
  **Solution**: Use Cloud KMS to manage keys, enable key rotation, strictly control key access permissions, prohibit hardcoding keys into code.

***

# 5. Summary and Expansion

Data masking is not a one-time technical operation, but a常态化 core work of enterprise data security governance. This article covers the full-scenario implementation needs from lightweight queries to enterprise-level large-scale data processing, from the core motivation, theoretical classification, and general solutions of masking to the full-process practical operation of GCP SDP, helping enterprises balance data security and value release while meeting compliance requirements.