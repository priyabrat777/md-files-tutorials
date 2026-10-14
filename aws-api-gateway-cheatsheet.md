# Amazon API Gateway Cheat Sheet

**Interview preparation: API products, authorization, integrations, throttling, streaming, and timeouts.**

**Verified:** 2026-10-13. State the API type, endpoint type, and buffered/streaming mode before quoting limits.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. REST APIs, HTTP APIs, and WebSocket APIs](#2-rest-apis-http-apis-and-websocket-apis)
- [3. Current timeout and payload distinctions](#3-current-timeout-and-payload-distinctions)
- [4. Authentication and authorization](#4-authentication-and-authorization)
- [5. Integration choices](#5-integration-choices)
- [6. Throttling, quotas, and caching](#6-throttling-quotas-and-caching)
- [7. Long jobs and large payload architecture](#7-long-jobs-and-large-payload-architecture)
- [8. Deployments and observability](#8-deployments-and-observability)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Practical inspection](#10-practical-inspection)
- [11. Interview questions and quick revision](#11-interview-questions-and-quick-revision)
- [12. AWS sources](#12-aws-sources)

---

## 1. Fundamentals and mental model

API Gateway is a managed frontend for APIs. It handles request routing and integrations and offers API-type-specific security, monitoring, and management capabilities.

```text
Client → domain/stage → route or method → authorization → integration → backend
                                └── throttling / logging / supported validation
```

An API route, deployment/stage, integration, and backend function are separate resources. A correct function does not prove the deployed API points to it correctly.

## 2. REST APIs, HTTP APIs, and WebSocket APIs

| Requirement | Typical product |
|---|---|
| API keys/usage plans, caching, request validation, direct WAF integration | REST API |
| Lower-cost basic HTTP routes with built-in JWT authorizer | HTTP API |
| Private client-facing API endpoint | REST API |
| Persistent bidirectional client communication | WebSocket API |
| Supported response streaming | REST API |

Both REST and HTTP APIs support IAM and Lambda authorizers, Lambda integrations, and private backend integrations. HTTP APIs do not thereby become private *client-facing* endpoints. REST API JWT validation can use a Lambda authorizer; HTTP APIs have a native JWT authorizer. [Product comparison](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)

## 3. Current timeout and payload distinctions

| Path | Relevant documented behavior |
|---|---|
| Buffered REST integration | Traditional 29-second limit; Regional/private REST APIs can request an increase |
| HTTP API integration | Maximum 30 seconds |
| REST proxy response streaming | Up to 15 minutes with supported configuration |
| Ordinary buffered API payload | Common 10 MB boundary; backend integration can impose a smaller one |

A longer Regional/private REST timeout may require reduced account-level throttle quota. This is not a universal increase for every API type. [REST timeout increase](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/), [HTTP API quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-quotas.html)

For REST streaming, use supported `HTTP_PROXY` or `AWS_PROXY` integrations and the `STREAM` response transfer mode. Streaming can exceed the buffered response-size boundary, but **request streaming is not supported**. Idle timeouts also apply: currently five minutes for Regional/private endpoints and 30 seconds for edge-optimized endpoints. Buffer-dependent features such as response transformation/caching are restricted. [Response streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-transfer-mode.html)

**Trap:** Lambda's 90-minute Managed Instances async limit does not make a synchronous API request wait 90 minutes. The gateway, client, and integration have independent limits.

## 4. Authentication and authorization

| Mechanism | Good fit | Caution |
|---|---|---|
| IAM/SigV4 | AWS identities and service-to-service calls | Caller permissions and signing must match |
| JWT authorizer on HTTP API | OAuth/OIDC tokens | Validate issuer, audience, scopes, and token lifetime |
| Cognito user-pool authorizer on REST API | Cognito-backed users | Still enforce business authorization |
| Lambda authorizer | Custom logic | Caching scope and failure behavior matter |
| mTLS on supported custom domains | Client certificate authentication | Does not encode all business permissions |
| API key/usage plan | Client identification/metering | Not a substitute for authentication |

Business authorization belongs in the application as well: a valid token does not mean its holder may read any order ID. Scope authorizer caching keys so cached decisions do not authorize unrelated routes or tenants.

## 5. Integration choices

| Integration | Use |
|---|---|
| Lambda proxy | Function handles API event and returns required response shape |
| HTTP proxy | Forward to HTTP backend |
| Direct AWS service integration | Queue or service action without a pass-through function |
| Private integration through VPC Link | Reach supported private load-balancer/service targets |
| Mock integration where supported | Return configured response without backend |

**Current nuance:** the current REST/HTTP comparison lists private ALB integrations for both products. Older “REST VPC Link always requires NLB” answers omit newer support. Check VPC Link version and endpoint compatibility. [Integration comparison](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)

Lambda proxy payload formats differ between REST and HTTP APIs and between HTTP API payload versions. Match the handler and test headers, cookies, binary data, and errors.

## 6. Throttling, quotas, and caching

Throttling protects shared capacity with rate/burst limits; excessive requests receive 429 responses. Clients should use bounded retries with jitter and respect application deadlines.

Usage-plan quotas/throttling are best-effort controls, not hard financial spending caps. Enforce strict business entitlements or spend limits in a durable application mechanism.

REST caching can reduce backend load but requires correct cache keys and invalidation. Never mix personalized/tenant-specific responses under an insufficient cache key.

Quotas vary by Region and account. Avoid treating one memorized requests-per-second default as a guaranteed throughput entitlement. [API Gateway quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html)

## 7. Long jobs and large payload architecture

**Long-running job pattern:** authenticate and validate → persist job/idempotency key → enqueue → return `202 Accepted` plus job ID → client polls status or receives a completion signal.

This survives client disconnects and separates work completion from connection lifetime. A streaming API is useful for progressive output, but should not be the only durable record of an important job.

**Large upload pattern:** authorize upload → issue constrained S3 presigned access → client uploads directly → validate completion → process asynchronously.

The smallest limit across gateway, function, queue, and client governs a pipeline. Account for encoding and envelope expansion.

## 8. Deployments and observability

Manage APIs with OpenAPI/IaC, explicit stages, and controlled deployments. Verify the custom-domain mapping and the stage actually receiving traffic.

Use access logs with request ID, route, status, integration status/latency, and authorizer outcomes. Keep tokens and secrets out of logs. Separate total latency from backend integration latency and distinguish gateway-generated errors from application responses.

For REST streaming, test the deployed endpoint with a real streaming client; the console test path can buffer output and behave differently. [Streaming troubleshooting](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-streaming-troubleshoot.html)

## 9. Troubleshooting

| Symptom | Investigate |
|---|---|
| 401/403 | Token/signature, authorizer, resource policy, API key requirement, route |
| Browser fails but curl works | CORS preflight and actual-response headers |
| 429 | Stage/account/usage throttles and downstream limits |
| 502 from Lambda proxy | Response shape, exception, binary/header format |
| 504 | Gateway integration deadline versus backend/dependency latency |
| Old behavior after change | Stage deployment, alias/version, domain mapping, cache |
| Private integration unavailable | VPC Link state, SG/routes, listener/target health |

CORS is browser policy, not access control. A non-browser client is not prevented from calling an API merely because a CORS header is absent.

## 10. Practical inspection

Read-only examples; the CLI namespaces differ by API product:

```bash
aws apigateway get-rest-apis
aws apigateway get-stages --rest-api-id REST_API_ID
aws apigateway get-integration --rest-api-id REST_API_ID --resource-id RESOURCE_ID --http-method GET
aws apigatewayv2 get-apis
aws apigatewayv2 get-routes --api-id HTTP_API_ID
aws apigatewayv2 get-integrations --api-id HTTP_API_ID
```

## 11. Interview questions and quick revision

**REST versus HTTP API?** Choose by required features and cost; both can expose RESTful HTTP resources. The product name is not a statement that HTTP APIs cannot be RESTful.

**API keys secure the API?** They identify/meter clients for supported features, but use proper authentication and authorization.

**Why did a 60-second Lambda time out at the gateway?** Backend execution timeout and gateway integration timeout are independent; the function may continue after the client receives an error.

**How would you prevent duplicate order creation on retry?** Require an idempotency key and persist request/outcome semantics atomically with order creation where possible.

**Can an HTTP API call a private backend?** Yes through supported private integration, while its client-facing endpoint remains public/Regional.

## 12. AWS sources

- [REST versus HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)
- [REST timeout increase](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/)
- [HTTP API quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-quotas.html)
- [General quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html)
- [Response streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-transfer-mode.html)
- [Streaming troubleshooting](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-streaming-troubleshoot.html)

**Related:** [Lambda](aws-lambda-cheatsheet.md), [S3](aws-s3-cheatsheet.md), [ELB](aws-elb-cheatsheet.md).
