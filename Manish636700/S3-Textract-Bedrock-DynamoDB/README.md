# S3 → Textract → Bedrock → DynamoDB (Latest‑Update Pipeline)

End‑to‑end, console‑only (UI) setup guide with diagrams and copy‑paste snippets. This builds two Lambdas:

* **Lambda A**: On new PDF/TIFF/JPG/PNG in **Source S3**, runs Amazon Textract, writes **TXT + JSON** to **Dest S3**.
* **Lambda B**: On new **TXT/JSON** in **Dest S3**, calls **Amazon Bedrock** to infer a schema and stores structured data into **DynamoDB**.


---

## ✅ Prerequisites

* **Region**: `ap-south-1` (Mumbai) (any)
* AWS account with permissions to create: S3, Lambda, IAM roles, DynamoDB, CloudWatch, and Bedrock model access.
* Bedrock **model access** (e.g., `amazon.titan-text-express-v1`) enabled in your account for this region.

---

## 1) Create S3 buckets (UI)

1. Open **S3** → **Create bucket**.
2. **Bucket 1 (Source)**: e.g., `s3-source-data-store`.
3. **Bucket 2 (Dest)**: e.g., `s3-destination-bucket-store`.
4. Keep default Block Public Access **ON**. Create.

> Optional: Enable **Versioning** on both buckets (Properties → Bucket Versioning). This helps skip stale versions in “latest‑update”.

---

## 2) Create DynamoDB table (UI)

1. Open **DynamoDB** → **Tables** → **Create table**.
2. **Table name**: `structure-data` (or your choice).
3. **Partition key**: `documentId` (String).
4. Leave sort key blank (optional). Create table.

> You can add a GSI on `docType` later if you want to query by types.

---

## 3) Enable Bedrock model access (UI)

1. Open **Amazon Bedrock** → **Model access**.
2. Request access to `amazon.titan-text-express-v1` (or your chosen text model) in **ap-south-1**.
3. Wait until the status shows **Access granted**.

---

## 4) Create IAM role for Lambdas (UI)

Create **one role** shared by both Lambdas, or two roles. For simplicity, one role:

1. Open **IAM** → **Roles** → **Create role**.
2. **Trusted entity**: AWS service → **Lambda** → Next.
3. **Permissions**: attach the following policies (you can start broad, then tighten):

   * `AWSLambdaBasicExecutionRole` (CloudWatch Logs)
   * Custom inline policy (below)
4. Name: `LambdaTextractBedrockDdbRole` → Create role.

**Inline policy (edit ARNs/buckets/table as needed):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:HeadObject"
      ],
      "Resource": [
        "arn:aws:s3:::s3-source-data-store",
        "arn:aws:s3:::s3-source-data-store/*",
        "arn:aws:s3:::s3-destination-bucket-store",
        "arn:aws:s3:::s3-destination-bucket-store/*"
      ]
    },
    {
      "Sid": "Textract",
      "Effect": "Allow",
      "Action": [
        "textract:StartDocumentTextDetection",
        "textract:GetDocumentTextDetection",
        "textract:DetectDocumentText"
      ],
      "Resource": "*"
    },
    {
      "Sid": "BedrockInvoke",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DynamoDBWrite",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:YOUR_ACCOUNT_ID:table/structure-data"
    }
  ]
}
```

Replace `YOUR_ACCOUNT_ID` and bucket names.

---

## 5) Create **Lambda A** (Textract → TXT/JSON to Dest S3)

1. **Lambda** → **Create function** → Author from scratch.

   * Name: `lambda-textract-to-s3`
   * Runtime: **Python 3.12** (or 3.11)
   * Architecture: x86_64
   * Execution role: **Use existing** → select `LambdaTextractBedrockDdbRole`
2. After creation → **Configuration** → **General configuration** → set timeout to **5–10 minutes**.
3. **Configuration → Environment variables**:

   * `DEST_BUCKET` = `s3-destination-bucket-store`
   * `MAX_WAIT_SECS` = `600`
   * `POLL_INTERVAL` = `2.0`
4. **Code**: paste your working version that supports **PDF/TIFF (async)** and **JPG/PNG (sync)** and writes:

5. **Deploy** the function.

### Add S3 trigger for Lambda A

1. Open **S3** → **Source bucket** → **Properties** → **Event notifications** → **Create event notification**.
2. Name: `on-new-docs`.
3. Event types: **PUT** (All object create events).
4. Prefix: (optional) e.g., `incoming/`. Suffix: leave blank.
5. Destination: **Lambda function** → `lambda-textract-to-s3` → Save.

> Test: upload a PDF/JPEG into the source bucket; verify `.txt` and `.textract.json` appear in the destination bucket under `textract/...`.

---

## 6) Create **Lambda B** (Bedrock → DynamoDB, latest‑update aware)

1. **Lambda** → **Create function** → Author from scratch.

   * Name: `lambda-bedrock-to-dynamodb`
   * Runtime: **Python 3.12**
   * Role: `LambdaTextractBedrockDdbRole`
2. **Environment variables**:

   * `REGION` = `ap-south-1`
   * `TABLE_NAME` = `structure-data`
   * `MODEL_ID` = `amazon.titan-text-express-v1`
   * `MAX_CHARS` = `8000`
   * *(Optional for manual/cron)* `SRC_BUCKET` = `s3-destination-bucket-store`, `SRC_PREFIX` = `textract/`
3. **Code**: paste the **latest‑update** handler you have (the one that selects the newest record from event, or lists latest object for manual runs) and writes to DynamoDB.
4. **Deploy**.

### Add S3 trigger for Lambda B (filter .txt/.json)

1. Open **S3** → **Destination bucket** → **Event notifications** → **Create event notification**.
2. Name: `on-textract-outputs`.
3. Event types: **PUT**.
4. Prefix: `textract/`. **Suffix**: add filter for `.txt` and another for `.json` (create two notifications if the UI doesn’t allow multiple suffixes).
5. Destination: **Lambda function** → `lambda-bedrock-to-dynamodb` → Save.

> Result: whenever Lambda A writes outputs, Lambda B is triggered but processes **only the newest** object in that event batch.

---

## 8) Validate end‑to‑end

1. Upload `sample.pdf` to **Source** bucket.
2. Confirm in **Dest** bucket: `textract/.../sample.txt` and `.../sample.textract.json` appear.
3. Open **DynamoDB** → **Explorer** → check **structure-data** table contains:

   ```json
   {
     "documentId": "s3://s3-destination-bucket-store/textract/.../sample.txt",
     "docType": "resume | invoice | bank_statement | others",
     "data": {"email": "...", "phone": "...", ...},
     "createdAt": "2025-12-01T13:45:00Z"
   }
   ```

---

## 9) UI Troubleshooting checklist

* **Lambda A wrote nothing** → Check CloudWatch Logs for Lambda A; confirm IAM has `textract:*` and `s3:PutObject` to dest.
* **Lambda B not triggered** → Check Dest bucket **Event notifications** suffix filters; verify `.txt`/`.json` paths.
* **Bedrock access denied** → Ensure model access granted in **Bedrock → Model access** and IAM allows `bedrock:InvokeModel`.
* **DynamoDB ValidationException** → Ensure all field values are strings (your code stringifies), and table name matches.
* **Multiple events** → Confirm logs show it selected the **latest** by `LastModified`.

---



****
