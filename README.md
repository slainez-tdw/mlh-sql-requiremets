# Sales-with-Geo Lambda — Technical Documentation

*Last updated: 2025-09-01*

## 1) Overview

This AWS Lambda (Python) executes a **read-only SQL** that builds a sales dataset (`ventas` CTE + client geolocation) and returns JSON. It’s designed to work in **two invocation modes**:

* **API Gateway (Lambda proxy)** → returns the proxy response envelope (`statusCode`, `headers`, `body`, `isBase64Encoded`). ([Documentación de AWS][1])
* **Amazon Bedrock Agent (Lambda action group)** → returns a plain JSON object (convenient for tool results). ([Documentación de AWS][2])

The SQL is the exact query you provided (CTEs `ventas` + `ubicacion`, final `LEFT JOIN`, and the computed `VOLUMEN_VENTA`).

---

## 2) Repository layout

```
lambda_sales_query/
├─ app/
│  ├─ handler.py        # Lambda entrypoint (detects API GW vs Bedrock, orchestrates)
│  ├─ db.py             # Connection pooling & query execution (psycopg2)
│  ├─ query_sql.py      # Your SQL (as UTF-8 string)
│  ├─ http.py           # Response helpers (api_ok / api_error / ok)
│  └─ __init__.py
└─ requirements.txt     # psycopg2-binary
```

* **Handler:** `app/handler.lambda_handler` (Python). Lambda handlers receive `event` and `context`. ([Documentación de AWS][3])

---

## 3) Environment variables

Set these in the Lambda console (Configuration → Environment variables):

| Name           | Required | Example          | Notes                           |
| -------------- | -------- | ---------------- | ------------------------------- |
| `DB_HOST`      | Yes      | `db.example.rds` | Hostname                        |
| `DB_PORT`      | No       | `5432`           | Defaults to 5432                |
| `DB_DATABASE`  | Yes      | `analytics`      | Database name                   |
| `DB_USERNAME`  | Yes      | `reporter`       | DB user (read-only)             |
| `DB_PASSWORD`  | Yes      | `********`       | Use Secrets Manager if possible |
| `DB_CONECTION` | Optional | `postgres`       | Tolerated alias; not required   |

**Tip:** For production, prefer **Amazon RDS Proxy** + IAM auth to pool connections and avoid storms from Lambda scaling. ([Documentación de AWS][4], [Amazon Web Services, Inc.][5])

---

## 4) Runtime & dependencies

* **Runtime:** Python 3.12 (or 3.13 when available). ([Documentación de AWS][6])
* **Libraries:** `psycopg2-binary==2.9.9` (in `requirements.txt`). On Lambda you can use either a **precompiled layer** or bundled wheels; AWS Prescriptive Guidance documents packaging options. ([Documentación de AWS][7], [subaud.io][8])

---

## 5) Security & networking

* Place the function in the **same VPC**/subnets (or use public DB + security group rules) so it can reach the database.
* Restrict the DB user to **read-only** and schema/table level permissions for the referenced objects.
* If you use **RDS Proxy**, enable TLS and IAM auth as recommended. ([Documentación de AWS][9])

---

## 6) Invocation modes & response shapes

### 6.1 API Gateway (Lambda proxy integration)

**Request:** API Gateway forwards HTTP request as a proxy `event`.
**Response (from Lambda):**

```json
{
  "isBase64Encoded": false,
  "statusCode": 200,
  "headers": { "Content-Type": "application/json; charset=utf-8" },
  "body": "{\"data\":[{...}],\"meta\":{\"query\":\"sales_with_geo\",\"row_count\":123}}"
}
```

This envelope is **required** by Lambda proxy integrations. ([Documentación de AWS][1])

> Note: If you later serve binary content, API Gateway may base64-encode bodies depending on “Binary media types” settings. ([Documentación de AWS][10])

### 6.2 Bedrock Agent (Lambda action group)

**Request:** Bedrock calls your Lambda with an action-group input event (JSON structure defined in Bedrock docs). ([Documentación de AWS][2])

**Response (from Lambda):**

```json
{
  "data": [ { /* row */ }, ... ],
  "meta": { "query": "sales_with_geo", "row_count": 123, "source": "bandeja_resumen_ventas + geo" }
}
```

---

## 7) API contract (data model)

### 7.1 Success payload

* **`data`**: `array<object>` — rows from the SQL (dict-style column map).
* **`meta`**: object

  * `query`: `"sales_with_geo"`
  * `row_count`: integer
  * `source`: string

**Row fields** (partial; based on your SQL):

* `"COMPAÑIA"`, `"AÑO"`, `"PERIODO"`, `"FECHA"`, `"PREFIJO_PARTIDA_CONTABLE"`, `"NUMERO_DE_PARTIDA"`, `"FACTURA"`,
  `"CODIGO_DE_RUTA"`, `"VENDEDOR_CODIGO_BIC"`, `"NOMBRE_DEL_VENDEDOR"`, `"CLIENTE_CODIGO_BIC"`, `"NOMBRE_DEL_CLIENTE"`,
  `"TIPO_CLIENTE"`, `"CODIGO_DE_ARTICULO"`, `"NOMBRE_DEL_ARTICULO"`, `"AGRUPACION_DE_ARTICULO"`,
  `"CODIGO_DE_AGENCIA"`, `"NOMBRE_DE_AGENCIA"`,
  `"FACTURADO_QQ"`, `"DEVUELTO_QQ"`, `"QUINTALES_VENDIDOS"`,
  `"CANTIDAD_A_FACTURAR"`, `"CANTIDAD_DEVUELTA"`, `"CANTIDAD_VENDIDA"`,
  `"VALOR_LPS_FACTURADO"`, `"VALOR_LPS_DEVUEL"`, `"LEMPIRAS_VENDIDAS"`,
  `"DEPARTAMENTO"`, `"MUNICIPIO"`, `"COLONIA"`,
  `"VOLUMEN_VENTA"`

> The handler returns **all** columns produced by the query and does not rename them.

### 7.2 Error payloads

* **API Gateway mode:** Proxy error envelope with JSON body:

  ```json
  {
    "isBase64Encoded": false,
    "statusCode": 500,
    "headers": {"Content-Type": "application/json; charset=utf-8"},
    "body": "{\"error\":\"internal_error\",\"details\":{\"exception\":\"...\"}}"
  }
  ```
* **Bedrock mode:**

  ```json
  { "error": "internal_error", "details": { "exception": "..." } }
  ```

---

## 8) Handler behavior

1. **Executes the SQL** from `query_sql.SQL_SALES_WITH_GEO`.
2. Uses a **module-level connection** (psycopg2) to reuse across warm invocations.
3. Detects if the incoming `event` matches API Gateway proxy; if so, wraps with proxy response; otherwise returns plain JSON.

Handler signature and context usage follow Lambda Python conventions. ([Documentación de AWS][3])

---

## 9) Deployment steps (manual quick start)

1. **Package code**

   * Zip the `app/` folder and `requirements.txt`.
   * If you can’t build native wheels in the console, use a **Lambda layer** or Docker build for `psycopg2`. ([Documentación de AWS][7], [subaud.io][8])
2. **Create function** (Runtime Python 3.12), set **Handler** to `app/handler.lambda_handler`. ([Documentación de AWS][6])
3. **VPC config** (if DB is private): attach the Lambda to the DB’s subnets + security group.
4. **Set environment variables** (Section 3).
5. **(Optional) Add RDS Proxy** between Lambda and RDS for pooling. ([Documentación de AWS][4])
6. **(Optional) API Gateway**: create HTTP API → Lambda proxy integration → deploy stage. The function already returns the required proxy shape. ([Documentación de AWS][1])
7. **(Optional) Bedrock Agent action group**: point the action to this Lambda. Bedrock will send its action event and expect JSON output. ([Documentación de AWS][11])

---

## 10) Testing

### 10.1 Lambda console (direct)

* Create a basic test event `{}` and invoke. You should see:

  ```json
  { "data": [ ...rows... ], "meta": { "query": "sales_with_geo", "row_count": N } }
  ```

### 10.2 API Gateway

* Call your endpoint:

  ```
  GET https://<api-id>.execute-api.<region>.amazonaws.com/sales
  ```
* Expect a proxy response envelope with `body` containing the same `{ "data": ..., "meta": ... }`. ([Documentación de AWS][1])

### 10.3 Bedrock Agent

* Trigger a tool invocation that routes to this action group. The result object is your returned JSON. ([Documentación de AWS][2])

---

## 11) Operations & performance

* **Connection reuse:** Keeping the psycopg2 connection in a module variable reduces cold-start overhead; pair with **RDS Proxy** for pooled connections at scale. ([Documentación de AWS][4], [Amazon Web Services, Inc.][5])
* **Timeouts:** Size your Lambda timeout to the expected query latency; consider adding server-side limits or indices for the joined tables.
* **Payload size:** If datasets are large, add paging (e.g., `LIMIT`/`OFFSET`) or aggregation. API Gateway has payload limits; Bedrock tool results should also be kept reasonable.
* **Binary responses:** If you ever return binary, follow API Gateway base64 rules. ([Documentación de AWS][10])

---

## 12) Troubleshooting

* **`ModuleNotFoundError: psycopg2`** – Package via layer or build for the Lambda runtime/arch; see AWS prescriptive guidance. ([Documentación de AWS][7])
* **DB connection spikes / “too many connections”** – Use **RDS Proxy**. ([Documentación de AWS][4])
* **Proxy 502 from API Gateway** – Ensure response matches the exact proxy format (keys and types). ([Documentación de AWS][1])
* **Bedrock tool not parsing** – Ensure you return **valid JSON** (object/array) from the Lambda; Bedrock’s action group forwards tool output as JSON to the agent. ([Documentación de AWS][2])

---

## 13) Extensibility

* **Filters**: Parse `event.queryStringParameters` (API Gateway) or the Bedrock action input (parameters object) and parameterize the SQL with psycopg2 placeholders.
* **Schema stability**: If downstreams need stable keys without accents, add a projection/rename layer before returning.

---

## 14) Compliance notes

* The function is **read-only** by design. Grant the DB user `SELECT` only.
* Secure credentials with **AWS Secrets Manager** and IAM where possible; or use IAM DB auth via RDS Proxy. ([Documentación de AWS][4])

---

## 15) Appendix — Example events

**API Gateway proxy event (abridged):**

```json
{
  "version": "2.0",
  "rawPath": "/sales",
  "requestContext": { "http": { "method": "GET" } },
  "queryStringParameters": null,
  "headers": { "accept": "application/json" }
}
```

*Your Lambda must answer with the proxy envelope.* ([Documentación de AWS][1])

**Bedrock action group event (shape reference):**

```json
{
  "agent": { "alias": "...", "id": "..." },
  "actionGroup": "sales_query",
  "parameters": { /* tool params if you add them */ },
  "sessionId": "..."
}
```

*Bedrock defines the general input format for Lambda action groups.* ([Documentación de AWS][2])

---

### References

* Lambda Python handler & context. ([Documentación de AWS][3])
* API Gateway Lambda **proxy** output format. ([Documentación de AWS][1])
* Bedrock Agents ↔ Lambda action groups. ([Documentación de AWS][2])
* Using `psycopg2` on Lambda (packaging strategies). ([Documentación de AWS][7], [subaud.io][8])
* RDS Proxy guidance & best practices (pooling, TLS). ([Documentación de AWS][4], [Amazon Web Services, Inc.][5])
* Binary media handling with API Gateway. ([Documentación de AWS][10])

---

If you want, I can add a **parameterized** variant of the SQL (e.g., by year/company) and document the expected query parameters for both API Gateway and Bedrock tool calls.

[1]: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html?utm_source=chatgpt.com "Lambda proxy integrations in API Gateway"
[2]: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html?utm_source=chatgpt.com "Configure Lambda functions to send information that an ..."
[3]: https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html?utm_source=chatgpt.com "Define Lambda function handler in Python - AWS Documentation"
[4]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-planning.html?utm_source=chatgpt.com "Planning where to use RDS Proxy"
[5]: https://aws.amazon.com/blogs/compute/using-amazon-rds-proxy-with-aws-lambda/?utm_source=chatgpt.com "Using Amazon RDS Proxy with AWS Lambda"
[6]: https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html?utm_source=chatgpt.com "Building Lambda functions with Python - AWS Documentation"
[7]: https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/import-psycopg2-library-lambda.html?utm_source=chatgpt.com "Import the psycopg2 library to AWS Lambda to interact with ..."
[8]: https://subaud.io/blog/using-psycopg2-with-aws-lambda-and-cdk?utm_source=chatgpt.com "Using psycopg2 with AWS Lambda and CDK: Binary vs ..."
[9]: https://docs.aws.amazon.com/prescriptive-guidance/latest/amazon-rds-proxy/best-practices.html?utm_source=chatgpt.com "Best practices - AWS Prescriptive Guidance"
[10]: https://docs.aws.amazon.com/apigateway/latest/developerguide/lambda-proxy-binary-media.html?utm_source=chatgpt.com "Return binary media from a Lambda proxy integration in ..."
[11]: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-add.html?utm_source=chatgpt.com "Add an action group to your agent in Amazon Bedrock"
