# Troubleshooting LangChain + FAISS with APIM Gateway (404 Errors)

## Problem Overview
When using `AzureOpenAIEmbeddings` from LangChain with FAISS vector store through an APIM gateway, you may encounter 404 errors even though direct OpenAI client calls work fine.

## Common Symptoms
```python
# This works (OpenAI client):
response = client.embeddings.create(input="test", model="text-embedding-3-large")

# This fails with 404 (LangChain):
embeddings = AzureOpenAIEmbeddings(azure_endpoint=APIM_ENDPOINT, ...)
db = FAISS.from_documents(docs, embeddings)  # 404 Error!
```

## Root Cause Analysis

### Understanding APIM Path Routing
When APIM is configured with an API path (e.g., `path: 'openai'` in Bicep), it **strips** that prefix before forwarding to the backend:

```
Client Request:   https://apim-gateway.azure-api.net/openai/deployments/model/embeddings
                                                    ↓
APIM strips:      /openai/deployments/model/embeddings → /deployments/model/embeddings
                                                    ↓
Backend URL:      https://azure-openai-endpoint.com{stripped_path}
```

**Critical Point:** The APIM policy MUST add `/openai` back to the backend URL because Azure OpenAI expects it.

---

## Step-by-Step Troubleshooting

### Step 1: Verify APIM Policy Configuration

#### Check Current Policy
Run this command to view your current APIM policy:

```bash
az rest --method get \
  --uri "https://management.azure.com/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.ApiManagement/service/{apim-service-name}/apis/{api-id}/policies/policy?api-version=2024-05-01" \
  --query properties.value -o tsv
```

**Example:**
```bash
az rest --method get \
  --uri "https://management.azure.com/subscriptions/49aed8c0-bab4-4bc2-a8dc-8d8ef44866ba/resourceGroups/rg-apim-aoai-gateway/providers/Microsoft.ApiManagement/service/apim-aoai-gateway-dev/apis/azure-openai-api/policies/policy?api-version=2024-05-01" \
  --query properties.value -o tsv
```

#### What to Look For
The policy should include `/openai` in the backend URL:

✅ **CORRECT:**
```xml
<set-backend-service base-url="{{azure-openai-endpoint}}/openai" />
```

❌ **INCORRECT (causes 404):**
```xml
<set-backend-service base-url="{{azure-openai-endpoint}}" />
```

### Step 2: Enable APIM Tracing

APIM tracing shows you exactly what's being sent to the backend.

> **Note:** Tracing must be enabled for the subscription. If you see `Ocp-Apim-Trace-AuthorizationExpired`, enable tracing in the Azure Portal under API Management → Subscriptions → [Your Subscription] → Enable tracing (checkbox).

#### Test with Tracing Enabled
```bash
curl -v -X POST "https://your-apim-gateway.azure-api.net/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-12-01-preview" \
  -H "api-key: YOUR_SUBSCRIPTION_KEY" \
  -H "Ocp-Apim-Trace: true" \
  -H "Content-Type: application/json" \
  -d '{"input": ["test"]}'
```

#### Examine Trace Output
Look for the `Ocp-Apim-Trace-Location` header in the response, then fetch the trace:

```bash
curl -s "TRACE_URL_FROM_HEADER"
```

**Alternative:** If tracing authorization has expired, you can still diagnose by checking the HTTP status code and response body without the detailed trace.

**What to Check in Trace:**
```json
{
  "source": "set-backend-service",
  "data": {
    "message": "Backend service URL was changed.",
    "request": {
      "url": "https://admin-resource.openai.azure.com/openai/deployments/..."
    }
  }
}
```

✅ **CORRECT:** Backend URL includes `/openai/deployments/...`  
❌ **INCORRECT:** Backend URL is `/deployments/...` (missing `/openai`)

### Step 3: Test Backend URL Directly

Verify Azure OpenAI expects `/openai` in the path:

```bash
# This should work:
curl -X POST "https://your-azure-openai-endpoint.com/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-12-01-preview" \
  -H "api-key: YOUR_AZURE_OPENAI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": ["test"]}'

# This should fail with 404:
curl -X POST "https://your-azure-openai-endpoint.com/deployments/text-embedding-3-large/embeddings?api-version=2024-12-01-preview" \
  -H "api-key: YOUR_AZURE_OPENAI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": ["test"]}'
```

---

## Fix Instructions

### Option 1: Update APIM Policy via Azure CLI

#### Create Policy JSON File
Create `policy-update.json`:

```json
{
  "properties": {
    "value": "<policies><inbound><base /><set-backend-service base-url=\"{{azure-openai-endpoint}}/openai\" /><set-header name=\"api-key\" exists-action=\"override\"><value>{{azure-openai-api-key}}</value></set-header><set-header name=\"Ocp-Apim-Subscription-Key\" exists-action=\"delete\" /><set-query-parameter name=\"api-version\" exists-action=\"override\"><value>{{azure-openai-api-version}}</value></set-query-parameter><rate-limit-by-key calls=\"100\" renewal-period=\"60\" counter-key=\"@(context.Subscription.Id)\" /><set-variable name=\"requestTimestamp\" value=\"@(DateTime.UtcNow.ToString())\" /></inbound><backend><forward-request /></backend><outbound><base /><set-header name=\"X-APIM-Trace-Id\" exists-action=\"override\"><value>@(context.RequestId.ToString())</value></set-header><set-header name=\"X-Response-Time\" exists-action=\"override\"><value>@((DateTime.UtcNow - DateTime.Parse(context.Variables.GetValueOrDefault&lt;string&gt;(\"requestTimestamp\"))).TotalMilliseconds.ToString())</value></set-header></outbound><on-error><base /><return-response><set-status code=\"@(context.Response.StatusCode)\" reason=\"@(context.Response.StatusReason)\" /><set-header name=\"Content-Type\" exists-action=\"override\"><value>application/json</value></set-header><set-body>@{ var error = context.LastError; return new JObject(new JProperty(\"error\", new JObject(new JProperty(\"code\", context.Response.StatusCode.ToString()), new JProperty(\"message\", error != null ? error.Message : \"An error occurred\"), new JProperty(\"source\", error != null ? error.Source : \"APIM\")))).ToString(); }</set-body></return-response></on-error></policies>",
    "format": "xml"
  }
}
```

#### Apply the Policy
```bash
az rest --method put \
  --uri "https://management.azure.com/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.ApiManagement/service/{apim-service-name}/apis/{api-id}/policies/policy?api-version=2024-05-01" \
  --body @policy-update.json
```

**Example:**
```bash
az rest --method put \
  --uri "https://management.azure.com/subscriptions/49aed8c0-bab4-4bc2-a8dc-8d8ef44866ba/resourceGroups/rg-apim-aoai-gateway/providers/Microsoft.ApiManagement/service/apim-aoai-gateway-dev/apis/azure-openai-api/policies/policy?api-version=2024-05-01" \
  --body @policy-update.json
```

### Option 2: Update via Azure Portal

1. Navigate to Azure Portal → API Management Service
2. Select your APIM instance
3. Go to **APIs** → Select your Azure OpenAI API
4. Click **All operations**
5. Click **</> Code editor** (Policy code editor icon)
6. In the `<inbound>` section, find `<set-backend-service>` and update to:
   ```xml
   <set-backend-service base-url="{{azure-openai-endpoint}}/openai" />
   ```
7. Move `<forward-request />` from `<inbound>` to `<backend>` section:
   ```xml
   <backend>
     <forward-request />
   </backend>
   ```
8. Click **Save**

### Option 3: Update Bicep Template (For New Deployments)

Update your `apim-aoai-gateway.bicep` file:

```bicep
resource aoaiApiPolicy 'Microsoft.ApiManagement/service/apis/policies@2024-05-01' = {
  parent: aoaiApi
  name: 'policy'
  properties: {
    value: '<policies><inbound><base /><set-backend-service base-url="{{azure-openai-endpoint}}/openai" /><set-header name="api-key" exists-action="override"><value>{{azure-openai-api-key}}</value></set-header><set-header name="Ocp-Apim-Subscription-Key" exists-action="delete" /><set-query-parameter name="api-version" exists-action="override"><value>{{azure-openai-api-version}}</value></set-query-parameter><rate-limit-by-key calls="${rateLimitCalls}" renewal-period="${rateLimitRenewalPeriod}" counter-key="@(context.Subscription.Id)" /><set-variable name="requestTimestamp" value="@(DateTime.UtcNow.ToString())" /></inbound><backend><forward-request /></backend><outbound><base /><set-header name="X-APIM-Trace-Id" exists-action="override"><value>@(context.RequestId.ToString())</value></set-header><set-header name="X-Response-Time" exists-action="override"><value>@((DateTime.UtcNow - DateTime.Parse(context.Variables.GetValueOrDefault&lt;string&gt;("requestTimestamp"))).TotalMilliseconds.ToString())</value></set-header></outbound><on-error><base /><return-response><set-status code="@(context.Response.StatusCode)" reason="@(context.Response.StatusReason)" /><set-header name="Content-Type" exists-action="override"><value>application/json</value></set-header><set-body>@{ var error = context.LastError; return new JObject(new JProperty("error", new JObject(new JProperty("code", context.Response.StatusCode.ToString()), new JProperty("message", error != null ? error.Message : "An error occurred"), new JProperty("source", error != null ? error.Source : "APIM")))).ToString(); }</set-body></return-response></on-error></policies>'
    format: 'xml'
  }
}
```

**Key change:** `/openai` is included in the backend URL.

---

## Verification & Testing

### Test 1: Direct curl Test
```bash
curl -s -X POST "https://your-apim-gateway.azure-api.net/openai/deployments/text-embedding-3-large/embeddings?api-version=2024-12-01-preview" \
  -H "api-key: YOUR_SUBSCRIPTION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": ["test"]}' | jq .
```

**Expected Response:**
```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [-0.024726123, -0.008289817, ...]
    }
  ],
  "model": "text-embedding-3-large",
  "usage": {
    "prompt_tokens": 1,
    "total_tokens": 1
  }
}
```

### Test 2: Python OpenAI Client Test
```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="YOUR_SUBSCRIPTION_KEY",
    azure_endpoint="https://your-apim-gateway.azure-api.net",
    api_version="2024-12-01-preview"
)

response = client.embeddings.create(
    input="test",
    model="text-embedding-3-large"
)

print(f"✅ Success! Embedding dimension: {len(response.data[0].embedding)}")
```

### Test 3: LangChain + FAISS Test
```python
from langchain_openai import AzureOpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document

# Configure embeddings
embeddings = AzureOpenAIEmbeddings(
    azure_endpoint="https://your-apim-gateway.azure-api.net",
    api_key="YOUR_SUBSCRIPTION_KEY",
    azure_deployment="text-embedding-3-large",
    openai_api_version="2024-12-01-preview"
)

# Create test documents
docs = [
    Document(page_content="Azure OpenAI embeddings integration"),
    Document(page_content="FAISS vector store for semantic search"),
    Document(page_content="LangChain simplifies LLM orchestration")
]

# Build FAISS index - this should work without 404
db = FAISS.from_documents(docs, embeddings)

# Test query
results = db.similarity_search("How to use FAISS with Azure OpenAI?", k=2)

print("✅ FAISS index created successfully!")
for i, res in enumerate(results, start=1):
    print(f"{i}. {res.page_content}")
```

---

## Common Issues & Solutions

### Issue 1: Still Getting 404 After Policy Update

**Possible Causes:**
- Policy update didn't apply (APIM caches policies)
- Wrong API ID or subscription key
- API version mismatch

**Solution:**
```bash
# 1. Verify policy was updated
az rest --method get \
  --uri "https://management.azure.com/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/apis/{api-id}/policies/policy?api-version=2024-05-01" \
  --query properties.value -o tsv | grep "set-backend-service"

# Should show: <set-backend-service base-url="{{azure-openai-endpoint}}/openai" />

# 2. Wait 1-2 minutes for cache refresh, or restart APIM (Developer tier)

# 3. Check API version in named values
az apim nv show \
  --resource-group {rg} \
  --service-name {apim} \
  --named-value-id azure-openai-api-version \
  --query value -o tsv

# Should match: 2024-12-01-preview (or your target version)
```

### Issue 2: Authentication Error (401)

**Symptoms:**
```json
{"error": {"code": "401", "message": "Access denied due to missing subscription key"}}
```

**Solution:**
Verify subscription key header name in APIM configuration:

```bash
# Check subscription key parameter names
az rest --method get \
  --uri "https://management.azure.com/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/apis/{api-id}?api-version=2024-05-01" \
  --query properties.subscriptionKeyParameterNames
```

**Expected Output:**
```json
{
  "header": "api-key",
  "query": "subscription-key"
}
```

**Fix in Code:**
LangChain's `AzureOpenAIEmbeddings` sends the key as the `api-key` header by default, which matches the APIM configuration.

### Issue 3: Wrong API Version

**Symptoms:**
- 400 Bad Request
- "Invalid API version" error

**Solution:**
Ensure consistency across:
1. APIM named value: `azure-openai-api-version`
2. Python code: `openai_api_version` parameter
3. Azure OpenAI deployment support

```bash
# Update APIM named value if needed
az rest --method patch \
  --uri "https://management.azure.com/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/namedValues/azure-openai-api-version?api-version=2024-05-01" \
  --body '{"properties": {"value": "2024-12-01-preview"}}'
```

### Issue 4: Rate Limiting Errors

**Symptoms:**
```json
{"error": {"code": "429", "message": "Rate limit is exceeded"}}
```

**Solution:**
Check APIM rate limit policy:

```xml
<rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Id)" />
```

**Options:**
1. Increase rate limits in Bicep and redeploy
2. Create multiple subscriptions for different apps
3. Use Azure OpenAI's native rate limiting instead

---

## Architecture Diagram

```
┌─────────────────┐
│   LangChain     │
│  + FAISS Code   │
└────────┬────────┘
         │ AzureOpenAIEmbeddings(
         │   azure_endpoint="https://apim-gateway.../",
         │   api_key="SUBSCRIPTION_KEY",
         │   azure_deployment="text-embedding-3-large"
         │ )
         ▼
┌─────────────────────────────────────────────────────────┐
│  APIM Gateway: https://apim-gateway.azure-api.net       │
│                                                           │
│  Incoming: /openai/deployments/model/embeddings         │
│            ↓                                              │
│  API Config: path='openai' → strips /openai prefix      │
│            ↓                                              │
│  Policy: <set-backend-service                            │
│           base-url="{{azure-openai-endpoint}}/openai"/>  │
│            ↓                                              │
│  Backend Call: {{endpoint}}/openai/deployments/model/... │
└────────────────────┬────────────────────────────────────┘
                     ▼
         ┌───────────────────────────┐
         │  Azure OpenAI Service     │
         │  Deployment:              │
         │  text-embedding-3-large   │
         └───────────────────────────┘
```

---

## Quick Reference Commands

### Get Your APIM Configuration Details
```bash
# Get APIM gateway URL
az apim show --resource-group {rg} --name {apim-name} --query gatewayUrl -o tsv

# Get subscription key (note: uses POST method for listSecrets)
az rest --method post \
  --uri "https://management.azure.com/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/subscriptions/default-subscription/listSecrets?api-version=2024-05-01" \
  --query primaryKey -o tsv

# Get Azure OpenAI endpoint from named value
az apim nv show \
  --resource-group {rg} \
  --service-name {apim} \
  --named-value-id azure-openai-endpoint \
  --query value -o tsv
```

### Create a Test Script
Save this as `test-langchain-apim.py`:

```python
#!/usr/bin/env python3
import os
from langchain_openai import AzureOpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document

# Configuration
APIM_GATEWAY_URL = os.getenv("APIM_GATEWAY_URL", "https://your-apim-gateway.azure-api.net")
APIM_SUBSCRIPTION_KEY = os.getenv("APIM_SUBSCRIPTION_KEY", "your-key-here")
DEPLOYMENT_NAME = "text-embedding-3-large"
API_VERSION = "2024-12-01-preview"

print("Testing LangChain + FAISS with APIM...")
print(f"Gateway: {APIM_GATEWAY_URL}")
print(f"Deployment: {DEPLOYMENT_NAME}\n")

# Create embeddings wrapper
embeddings = AzureOpenAIEmbeddings(
    azure_endpoint=APIM_GATEWAY_URL,
    api_key=APIM_SUBSCRIPTION_KEY,
    azure_deployment=DEPLOYMENT_NAME,
    openai_api_version=API_VERSION
)

# Test documents
docs = [
    Document(page_content="Azure OpenAI provides powerful AI capabilities"),
    Document(page_content="FAISS enables fast similarity search"),
    Document(page_content="LangChain makes building AI applications easier")
]

try:
    # Build FAISS index
    print("Building FAISS index...")
    db = FAISS.from_documents(docs, embeddings)
    print("✅ FAISS index created successfully!\n")
    
    # Test query
    query = "How to search documents efficiently?"
    print(f"Query: {query}")
    results = db.similarity_search(query, k=2)
    
    print("\nTop matches:")
    for i, res in enumerate(results, start=1):
        print(f"{i}. {res.page_content}")
    
    print("\n✅ All tests passed!")
    
except Exception as e:
    print(f"\n❌ Error: {type(e).__name__}")
    print(f"   {str(e)}")
    print("\nTroubleshooting steps:")
    print("1. Verify APIM policy includes '/openai' in backend URL")
    print("2. Check subscription key is valid")
    print("3. Ensure API version matches APIM configuration")
    print("4. Enable APIM tracing for detailed diagnostics")
```

Run with:
```bash
export APIM_GATEWAY_URL="https://your-apim-gateway.azure-api.net"
export APIM_SUBSCRIPTION_KEY="your-subscription-key"
python3 test-langchain-apim.py
```

---

## Additional Resources

- [Azure APIM Policies Reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)
- [Azure OpenAI REST API Reference](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference)
- [LangChain Azure OpenAI Integration](https://python.langchain.com/docs/integrations/text_embedding/azureopenai)
- [FAISS Documentation](https://github.com/facebookresearch/faiss)

## Support

If you continue experiencing issues after following this guide:

1. ✅ Verify all policy changes are applied
2. ✅ Test with curl to isolate the issue
3. ✅ Enable APIM tracing and examine the request flow
4. ✅ Check Azure OpenAI service health and quotas
5. ✅ Review APIM diagnostic logs in Application Insights

For further assistance, provide:
- APIM trace output
- Error messages from Python
- Policy XML configuration
- API version information
