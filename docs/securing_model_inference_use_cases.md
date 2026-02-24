# Securing AI model inference endpoints with F5 Distributed Cloud WAAP

This guide walks you through configuring **F5 Distributed Cloud (XC) Web Application and API Protection (WAAP)** features to secure a generative AI model inference endpoint running on Red Hat OpenShift AI.

**Objective:** Secure the inference endpoint against cross-site scripting (XSS), shadow APIs, and denial-of-service (DoS) attacks.

## Table of contents

- [Prerequisites](#prerequisites)
- [Step 0: Initial load balancer configuration and endpoint verification](#step-0-initial-load-balancer-configuration-and-endpoint-verification)
  - [Task 0.1: Verify the LlamaStack service in OpenShift](#task-01-verify-the-llamastack-service-in-openshift)
  - [Task 0.2: Set up the HTTP Load Balancer](#task-02-set-up-the-http-load-balancer)
  - [Verification of inference endpoint access](#verification-of-inference-endpoint-access)
- [Use Case 1: Protecting the inference endpoint with WAF](#use-case-1-protecting-the-inference-endpoint-with-waf)
  - [Task 1.1: Simulate an unmitigated XSS attack (before WAF)](#task-11-simulate-an-unmitigated-xss-attack-before-waf)
  - [Task 1.2: Enable a WAF policy on the Load Balancer](#task-12-enable-a-waf-policy-on-the-load-balancer)
  - [Task 1.3: Simulate a mitigated XSS attack (after WAF)](#task-13-simulate-a-mitigated-xss-attack-after-waf)
  - [Notes and troubleshooting](#notes-and-troubleshooting)
  - [Appendix: Example cURL request](#appendix-example-curl-request)
- [Use Case 2: Enforcing API specification and preventing shadow APIs](#use-case-2-enforcing-api-specification-and-preventing-shadow-apis)
  - [Task 2.1: Simulate allowed access to a shadow API](#task-21-simulate-allowed-access-to-a-shadow-api)
  - [Task 2.2: Create an API Definition](#task-22-create-an-api-definition)
  - [Task 2.3: Enable API Inventory and block shadow APIs](#task-23-enable-api-inventory-and-block-shadow-apis)
  - [Task 2.4: Simulate blocked access to a shadow API](#task-24-simulate-blocked-access-to-a-shadow-api)
  - [Summary](#summary)
- [Use Case 3: Preventing DoS attacks with rate limiting](#use-case-3-preventing-dos-attacks-with-rate-limiting)
  - [Task 3.1: Simulate unmitigated excessive requests](#task-31-simulate-unmitigated-excessive-requests)
  - [Task 3.2: Configure rate limiting](#task-32-configure-rate-limiting)
  - [Task 3.3: Simulate mitigated excessive requests](#task-33-simulate-mitigated-excessive-requests)

---

## Prerequisites

- Operational **F5 Distributed Cloud** account and Console access
- **kubectl** or **oc** CLI installed locally
- HTTP Load Balancer and LLM inference service deployed on Red Hat OpenShift AI (see [F5 XC deployment guide](f5_xc_deployment.md))

---

## Step 0: Initial load balancer configuration and endpoint verification

This step ensures the model serving application is exposed via an F5 Distributed Cloud HTTP Load Balancer.

### Task 0.1: Verify the LlamaStack service in OpenShift

1. **Check service status**

   Ensure that the `llamastack` service is deployed and running in your OpenShift namespace:

   ```bash
   oc get pods -n <your-namespace> | grep llama
   oc get svc -n <your-namespace> | grep llama
   oc get endpoints -n <your-namespace> | grep llama
   ```

   Expected output:

   ```
   llamastack-<your-namespace>   ClusterIP   10.0.142.12   <none>   8080/TCP   2d
   llamastack-<your-namespace>-7d9c7b9d9f   1/1     Running   0     2d
   ```

2. **Confirm service accessibility (internal test)**

   Test the inference service from within the cluster:

   ```bash
   oc run test-client --rm -i --tty --image=registry.access.redhat.com/ubi9/ubi-minimal \
     -- curl -s http://llamastack-<your-namespace>.<your-namespace>.svc.cluster.local:8080/v1/openai/v1/models | jq
   ```

   The JSON response should list available models such as `Llama-3.2-1B-Instruct-quantized.w8a8`.

### Task 0.2: Set up the HTTP Load Balancer

1. Navigate to **Multi-Cloud App Connect → HTTP Load Balancers**.
2. Click **Add HTTP Load Balancer**.
   - **Name:** `ai-inference-lb`
   - **Domain Name:** `<your-xc-endpoint>`
3. **Configure Origin Pool:**
   - Click **Add Item** and name the pool.
4. **Configure Origin Server:**
   - Type: *K8s Service Name of Origin Server on given Sites*
   - Service Name: `llamastack.<your-namespace>`
   - Virtual Site Type: Select your site (e.g., `system/<your-site-name>`)
   - Network: `Outside Network`

   ![F5 XC Origin Pool configuration for LlamaStack service](images/llamastack-origin-pool.png)

   - Port: `8321`

   ![Origin Pool port configuration set to 8321](images/hipster-origin-pool_port.png)

5. **Save:** Continue → Apply → Save and Exit. Record the generated **CNAME**.

### Verification of inference endpoint access

```bash
curl -sS http://<your-xc-endpoint>/v1/openai/v1/models | jq
```

Expected output:

```json
{
  "data": [
    {
      "id": "remote-llm/RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8",
      "object": "model",
      "created": 1762644418,
      "owned_by": "llama_stack"
    },
    {
      "id": "sentence-transformers/all-MiniLM-L6-v2",
      "object": "model",
      "created": 1762644418,
      "owned_by": "llama_stack"
    }
  ]
}
```

---

## Use Case 1: Protecting the inference endpoint with WAF

**Scenario:** The LLM inference endpoint is susceptible to dynamic attacks such as Cross-Site Scripting (XSS), which could allow malicious scripts to be rendered and executed. This use case demonstrates how to apply an F5 XC Web Application Firewall (WAF) policy to block such attacks.

### Task 1.1: Simulate an unmitigated XSS attack (before WAF)

Simulate an XSS attack against the unprotected endpoint using Swagger UI.

1. **Navigate to the Swagger UI**

   Open a browser and go to:
   `http://<your-xc-endpoint>/docs#/default/chat_completion_v1_inference_chat_completion_post`

2. **Access the endpoint**

   Expand the Chat Completion endpoint (`/v1/inference/chat-completion`).

3. **Initiate testing**

   Click the **Try it out** button.

   ![Swagger UI showing the Chat Completion endpoint with Try it out button](images/swagger_chat.png)

4. **Insert malicious payload**

   Copy and paste the following JSON payload into the **Request body**. This payload injects a `<script>` tag into the `content` field:

   ```json
   {
     "model_id": "RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8",
     "messages": [
       {
         "role": "user",
         "content": "What is F5 API security? <script>alert(\"XSS\")</script>",
         "context": "Injection test"
       }
     ],
     "sampling_params": {
       "strategy": { "type": "greedy" },
       "max_tokens": 50,
       "repetition_penalty": 1,
       "stop": ["</script>"]
     },
     "stream": false,
     "logprobs": { "top_k": 0 }
   }
   ```

   > Replace `model_id` and other fields as needed for your deployment.

5. **Execute the attack**

   Click **Execute** in Swagger UI.

6. **Review unmitigated result**

   Inspect the **Server Response**. If the response body contains the injected `<script>` tag (or the script is rendered), the endpoint is vulnerable to XSS.

   ![Swagger UI response showing the unblocked XSS payload in the server response](images/swagger_chat_response.png)

### Task 1.2: Enable a WAF policy on the Load Balancer

Attach a WAF policy to the HTTP Load Balancer fronting the vLLM service.

1. **Navigate to Load Balancer management**

   In the F5 XC Console, go to **Web App & API Protection → Load Balancers → HTTP Load Balancers** (under *Manage*).

2. **Manage configuration**

   Find the HTTP Load Balancer for the vLLM endpoint. Click the action menu (`…`) → **Manage Configuration**.

3. **Edit configuration**

   Click **Edit Configuration**.

4. **Enable WAF**

   From the left navigation, select **Web Application Firewall**.

   ![Load Balancer configuration page showing the WAF section](images/ericji-gpu-ai-vllm-lb2.png)

5. **Create a new WAF object**

   Toggle **Enable** for the Web Application Firewall, then create a new WAF object (e.g., `waf-<your-prefix>-vllm`).

   ![WAF policy configuration with detection settings](images/waf-ericji-vllm.png)

   > In lab environments, settings such as *Suspicious* or *Good Bot* are sometimes set to *Ignore* to reduce false positives.

6. **Save changes**

   Go to **Other Settings** (left navigation), then click **Save and Exit**.

### Task 1.3: Simulate a mitigated XSS attack (after WAF)

Verify the WAF policy successfully blocks the XSS injection.

1. **Return to Swagger UI**

   Use the Swagger tab from Task 1.1 (or refresh the page):
   `http://<your-xc-endpoint>/docs#/default/chat_completion_v1_inference_chat_completion_post`

2. **Access the endpoint**

   Expand `/v1/inference/chat-completion` and click **Try it out**.

3. **Re-execute the attack**

   Paste the exact same malicious JSON payload from Task 1.1 into the Request body.

4. **Execute and review mitigated result**

   Click **Execute**. The WAF should intercept and block the malicious script. You will typically see a block message or an altered response indicating the request was rejected.

   ![Server response showing the WAF blocking the XSS attack](images/xss-response.png)

5. **Check the event log**

   View the detection event in **F5 Distributed Cloud Security Analytics**.

   ![F5 XC Security Analytics showing the XSS detection event](images/xss-event.png)

### Notes and troubleshooting

- If the block is not observed:
  - Confirm the WAF policy is attached to the **correct** HTTP Load Balancer (matching host/path).
  - Check policy precedence and any other policies that might override behavior.
  - Review WAF logs and attack telemetry in the F5 XC Console to confirm detection events.
  - Validate whether the WAF object is configured to **block** (not just log) for XSS rules.

- For lab-friendly testing, consider using a non-production model and a low-impact payload. Use caution when testing production systems.

### Appendix: Example cURL request

The following `curl` command sends the same malicious payload directly to the inference endpoint. Use only in controlled or test environments.

```bash
curl -X 'POST' 'http://<your-xc-endpoint>/v1/inference/chat-completion' \
  -H 'Content-Type: application/json' \
  -d '{
        "model_id": "RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8",
        "messages": [
          {
            "role": "user",
            "content": "What is F5 API security? <script>alert(\"XSS\")</script>",
            "context": "Injection test"
          }
        ],
          "max_tokens": 50
      }' | jq
```

---

## Use Case 2: Enforcing API specification and preventing shadow APIs

### Scenario

An updated model inference service (KServe/vLLM) introduced a new, unapproved API endpoint (e.g., a version check endpoint) that was not intended for external release. This unapproved endpoint is a **shadow API**. This use case demonstrates how to use **F5 XC API Security** to ensure that only documented, approved endpoints can be consumed.

**Prerequisite:** Ensure the HTTP Load Balancer and the LLM inference service are deployed on Red Hat OpenShift AI.

### Task 2.1: Simulate allowed access to a shadow API

Verify that the unapproved endpoint is accessible before security controls are applied.

1. Use a browser or `curl` to test the shadow API endpoint:

   ```bash
   curl -sS http://<your-xc-endpoint>/v1/version | jq
   ```

   The response should return version information, confirming the shadow API is currently accessible.

   ![Browser showing the version endpoint returning data before API spec enforcement](images/get_version_api.png)

2. *(Optional)* Use the Swagger UI at `http://<your-xc-endpoint>/docs` to confirm the endpoint is exposed.

### Task 2.2: Create an API Definition

Create an API Definition using the approved OpenAPI specification for the model inference service.

1. Navigate to **Web App & API Protection** in the F5 XC Console.
2. Go to **Manage → API Security → API Definition**.
3. Click **Add API Definition**.

   ![F5 XC API Definition creation page](images/API_definition.png)

4. Enter a name (e.g., `model-api-def`).
5. Under **OpenAPI Specification Files**, click **Add Item**.

   ![OpenAPI specification file upload dialog](images/OpenAPI_upload.png)

6. Upload or select the approved file: `openapi-swagger-v3-fixed2-version.json`.
7. Click **Save and Exit**.

### Task 2.3: Enable API Inventory and block shadow APIs

Enable API Inventory and Discovery on the Load Balancer fronting the inference endpoint.

1. Go to **Manage → Load Balancers → HTTP Load Balancers**.
2. Locate the Load Balancer for the LLM service → click **… → Manage Configuration**.
3. Click **Edit Configuration**.
4. Select **API Protection** from the left navigation.
5. In the first API Definition section, select **Enable**.
6. In the second API Definition section, select the definition created in Task 2.2.

   ![API Protection configuration with API Definition enabled](images/API_protection.png)

7. Under **Validation**, choose **API Inventory**, then click **View Configuration**.

   ![API Inventory validation settings](images/API_validation.png)

8. Change **Fall Through Mode** to **Custom**.
9. Under **Custom Fall Through Rule List**, choose **Configure**.

   ![Fall Through Mode configuration dialog](images/fall_through.png)

10. Add an item with:
    - **Name:** `block-shadow`
    - **Action:** Block
    - **Type:** Base Path
    - **Base Path:** `/v1`

    ![Shadow API blocking rule configured with base path /v1](images/block_shadow.png)

11. Click **Apply**.
12. Select **Other Settings**, then **Save and Exit**.

### Task 2.4: Simulate blocked access to a shadow API

After enforcing the API spec, the shadow API should be blocked.

1. Test using browser or `curl`:

   ```bash
   curl -sS http://<your-xc-endpoint>/v1/version
   ```

2. The request should now return a `403 Forbidden`, confirming that undocumented endpoints are blocked.

   ![Browser showing 403 Forbidden response when accessing the shadow API after enforcement](images/get_version_api_403.png)

### Summary

Enforcing the API specification acts as a strict **security manifest**. Any endpoint not explicitly approved in the OpenAPI file (`openapi-swagger-v3-fixed2-version.json`) is immediately rejected by F5 XC API Security, preventing exposure of shadow APIs.

---

## Use Case 3: Preventing DoS attacks with rate limiting

### Scenario

An LLM inference endpoint exposed at `http://<your-xc-endpoint>` experiences performance degradation due to a high volume of requests from a single client, potentially caused by accidental loops or intentional abuse. This use case configures a rate limit of **10 requests per client per minute** on the inference endpoint.

**Target endpoint:** `/v1/inference/chat-completion`

### Task 3.1: Simulate unmitigated excessive requests

Demonstrate that without rate limiting, the endpoint accepts unlimited requests from a single client.

1. **Navigate to the Swagger UI**

   Open a browser and go to `http://<your-xc-endpoint>/docs`.

2. **Access the target endpoint**

   Expand the `/v1/inference/chat-completion` endpoint and click **Try it out**.

   ![Swagger UI showing the chat completion endpoint](images/swagger_chat.png)

3. **Insert payload**

   Paste the following JSON payload into the **Request body**:

   ```json
   {
     "model_id": "RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8",
     "messages": [
       {"role": "user", "content": "Hello"}
     ]
   }
   ```

   > Replace `model_id` with the value for your deployment if different.

4. **Execute rapid requests**

   Click **Execute** repeatedly (10 or more times within 1 minute).

   ![Swagger UI Execute button for rapid request simulation](images/chat_execute.png)

5. **Review unmitigated result**

   Every request should return `200 OK`, confirming no rate limiting is enforced.

### Task 3.2: Configure rate limiting

Enable the **API Rate Limit** feature on the F5 XC HTTP Load Balancer.

1. **Access Load Balancer configuration**

   In the F5 XC Console, navigate to **Web App & API Protection → Load Balancers → HTTP Load Balancers** (under *Manage*).

2. **Manage and edit configuration**

   Locate the HTTP Load Balancer serving `<your-xc-endpoint>`. Click `…` → **Manage Configuration → Edit Configuration**.

3. **Navigate to Common Security Controls**

   Click the **Common Security Controls** link in the left navigation.

4. **Enable API Rate Limit**

   In the **Rate Limiting** section, select **API Rate Limit** from the drop-down.

   ![Rate Limiting configuration showing API Rate Limit option](images/API_rate_limit.png)

5. **View and configure**

   Under API Rate Limit, click **View Configuration**. Under *API Endpoints*, click **Configure**.

6. **Add item**

   Click **Add Item** within API Endpoints.

7. **Select the LLM endpoint**

   Use the drop-down under **API Endpoint** → **See Suggestions**, then select `/v1/inference/chat-completion`.

8. **Define threshold**

   Set the following values and click **Apply**:
   - **Method List:** ANY
   - **Threshold:** 10
   - **Duration:** Minute

   This limits each client to 10 requests per minute on the inference endpoint.

   ![API Rate Limit endpoint configuration with threshold of 10 requests per minute](images/API_rate_limit_endpoint.png)

9. **Apply and save**

   - Click **Apply** on the API Endpoint rule.
   - Click **Apply** on the API Rate Limit page.
   - Navigate to **Other Settings**, then click **Save and Exit**.

### Task 3.3: Simulate mitigated excessive requests

Verify that the rate limiting policy blocks requests exceeding the threshold.

1. **Return to Swagger UI**

   Navigate to `http://<your-xc-endpoint>/docs` (or refresh the page).

2. **Access the target endpoint**

   Expand `/v1/inference/chat-completion` and click **Try it out**.

3. **Execute rapid requests**

   Click **Execute** more than 10 times within 1 minute.

4. **Review mitigated result**

   Observe the **Server Response** for each execution:
   - Requests 1 through 10 should return `200 OK`.
   - Request 11 and subsequent requests within that minute should return `429 Too Many Requests` or a comparable block message.

   ![Rate-limited response showing 429 status after exceeding threshold](images/API_rate_limit_test.png)

5. **Review the dashboard**

   Confirm the rate-limiting events in the F5 XC Console dashboard.

   ![F5 XC dashboard showing rate limiting events for the inference endpoint](images/API_rate_limit_event.png)
