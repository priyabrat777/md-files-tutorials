# Amazon EventBridge Cheat Sheet

**Interview preparation: event buses, routing, subscribers, Pipes, Scheduler, retries, and replay.**

**Verified:** 2026-10-13. Current AWS documentation distinguishes the newer Custom Event Bus from Custom Event Bus - Classic; their APIs and defaults differ.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Product selection](#2-product-selection)
- [3. Current correction: two custom-bus models](#3-current-correction-two-custom-bus-models)
- [4. Classic event patterns and envelopes](#4-classic-event-patterns-and-envelopes)
- [5. Publishing, payloads, and partial failure](#5-publishing-payloads-and-partial-failure)
- [6. Retry and failure behavior](#6-retry-and-failure-behavior)
- [7. FIFO and idempotency](#7-fifo-and-idempotency)
- [8. Architecture patterns and tradeoffs](#8-architecture-patterns-and-tradeoffs)
- [9. Security and observability](#9-security-and-observability)
- [10. Troubleshooting and practical inspection](#10-troubleshooting-and-practical-inspection)
- [11. Interview questions and quick revision](#11-interview-questions-and-quick-revision)
- [12. Message formats and Python and Java examples](#12-message-formats-and-python-and-java-examples)
- [13. AWS sources](#13-aws-sources)

---

## 1. Fundamentals and mental model

EventBridge connects event producers to interested targets. Start by naming the product and bus generation before describing ordering, retention, or retry behavior.

```text
Classic bus: producer → event bus → matching rules → targets
Custom bus:  producer → retained event bus → matching subscribers → one target each
Pipes:       source → filter → optional enrichment → target
Scheduler:   time/rate/cron → target API invocation
```

## 2. Product selection

| Need | Candidate | Key distinction |
|---|---|---|
| Route events by content to independent consumers | Event bus | Many matching consumers |
| Connect a queue/stream to one downstream service | Pipes | Point-to-point integration |
| Run at a time or recurring schedule | Scheduler | Time-driven invocation |
| Buffer work for competing workers | SQS | Consumer-controlled processing backlog |
| Fan out notifications/messages | SNS | Topic/subscription model and protocols |

Pipes combines filtering, enrichment, and a target without requiring custom glue code for every step. Scheduler supports one-time and recurring schedules, time zones, flexible windows, retries, and DLQs. Neither should be confused with Classic bus rules. [Pipes](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html), [Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)

## 3. Current correction: two custom-bus models

| Capability | Custom Event Bus - Classic | Newer Custom Event Bus |
|---|---|---|
| Routing resource | Rule | Subscriber |
| Targets per routing resource | Up to 5 | One |
| Retention | Configure archive separately | Built-in configurable 1–365 days |
| Ordering | No ordering guarantee | Optional FIFO per event group/subscriber |
| Publisher deduplication | Not built in | Five-minute window |
| Replay | Archive replay | Subscriber starting position over retained data |
| Cross-account model | Resource policies / bus forwarding | AWS RAM sharing and consumer-owned subscribers |

Do not say “EventBridge never supports FIFO” or “all buses retain every event.” Those statements omit the generation. [Custom Event Bus overview](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-what-is.html)

The new CLI service is `eventsv2`, while Classic uses `events`. IAM actions still use `events:` and the service principal remains `events.amazonaws.com`; `eventsv2:PutEvents` is not a valid permission. Use current SDK/CLI support and full returned ARNs. [Names, endpoints, and permissions](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-names.html)

## 4. Classic event patterns and envelopes

A typical Classic event includes `source`, `detail-type`, `id`, `time`, `account`, `region`, and a `detail` payload. Consumers should depend on a versioned contract and business event ID, not incidental serialization details.

Example **Classic rule pattern**, not a publish request:

```json
{
  "source": ["com.example.orders"],
  "detail-type": ["OrderCreated"],
  "detail": {
    "country": ["IN"],
    "total": [{"numeric": [">=", 1000]}]
  }
}
```

Test both matching and nonmatching examples. Field types, nesting, and exact values matter. Input transformation changes the target payload, not the producer's original contract.

## 5. Publishing, payloads, and partial failure

For Classic `PutEvents`, AWS documents up to 10 entries and a combined entry size below **1 MiB per request**. It is not a separate 1 MiB allowance for every entry in a ten-entry batch.

Inspect `FailedEntryCount` and individual entries even after an HTTP-success response. Retry failed entries with bounded backoff and idempotency. Validate bus identity: Classic publishing to a nonexistent bus can return success while dropping events. [PutEvents behavior and sizes](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-putevents.html)

The newer bus uses its own publish API/resource model; do not reuse Classic request schemas without adapting them. Large events should normally carry a durable object reference when target limits or fanout costs make embedding data impractical.

## 6. Retry and failure behavior

| Path | Default retry window/policy |
|---|---|
| Classic rule target | Up to 24 hours and 185 retry attempts for retryable delivery failures |
| New Custom Event Bus subscriber | `MaxRetryAttempts=5`, `MaxEventAgeInSeconds=300` |

Retries stop when the applicable policy limit is reached. Explicitly migrate retry settings when changing bus models. Configure a DLQ and monitor failed/dropped delivery. [Classic delivery monitoring](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-monitoring-events-best-practices.html), [new subscriber retries](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-retry.html)

New-bus DLQ entries are failure records, not simply the original event body. Replay/redrive tooling must understand the record and retained-event model.

Separate **delivery accepted by target** from **business processing completed**. A target may accept asynchronously and fail later under its own retry system.

## 7. FIFO and idempotency

New Custom Event Bus FIFO ordering is per subscriber and event group. Publisher deduplication and subscriber ordering are separate settings.

For Lambda targets, asynchronous `EVENT` delivery preserves ordering only to the handoff into Lambda's queue; use the documented `REQUEST_RESPONSE` mode when ordered processing is required. Failing events can block their group while unrelated groups continue. [Ordering and deduplication](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-ordering.html)

Regardless of bus, consumers need durable idempotency for retries, replays, and side effects. A replay after the deduplication window can repeat a previously successful business operation.

## 8. Architecture patterns and tradeoffs

**Order workflow:** commit order and outbox event together; relay to EventBridge; route to fulfillment, billing, and analytics. The outbox closes the gap between committing a database transaction and publishing its event.

**Buffer a fragile target:** bus → SQS → bounded workers. This lets target processing lag without tying all recovery to the bus's delivery window.

**Scheduled reminder:** Scheduler → queue/function with an idempotent reminder ID. Model time zone and daylight-saving rules explicitly; a scheduling service is not a hard real-time system.

**Replay analytics:** retain/archive the necessary events, choose the starting point, limit replay rate, and prevent replayed events from reissuing payments or customer notifications unintentionally.

## 9. Security and observability

Apply least-privilege publisher/subscriber/target permissions and service trust policies. For Classic targets, the required mechanism varies between resource-based policies and execution roles. New-bus RAM sharing and delivery roles require their own configuration.

Monitor publishing failures, routing matches, delivery attempts, target failures, event age/latency, DLQ arrivals, and dropped events. Trace a business event ID across publisher, bus, queue, and consumer.

Avoid event loops: an action triggered by an event can produce another event that matches the same route. Make predicates specific and design consumers to recognize their own generated updates.

## 10. Troubleshooting and practical inspection

| Symptom | First check |
|---|---|
| Event published but no processing | Correct bus/API, pattern/filter, target permissions |
| Works in one account only | Resource policy/RAM sharing, delivery role, KMS |
| DLQ suddenly fills after migration | New shorter default retry policy or wrong target payload |
| Order not preserved | Bus generation, subscriber type/group, asynchronous handoff |
| Duplicate side effects during replay | Missing idempotency or replay-aware handling |
| Many invocations from one change | Broad patterns or event loop |

Read-only **Classic** inspection examples:

```bash
aws events describe-event-bus --name orders
aws events list-rules --event-bus-name orders
aws events list-targets-by-rule --event-bus-name orders --rule order-created
aws events test-event-pattern --event-pattern file://pattern.json --event file://event.json
```

The final command requires local sample JSON files. Do not use `events` commands to infer that a new `eventsv2` subscriber exists.

## 11. Interview questions and quick revision

**SNS, SQS, or EventBridge?** State whether the need is fanout, buffered work, content routing, replay, ordering, or scheduled invocation; then choose the product and configuration.

**Does HTTP 200 prove publish success?** Not for every batched entry; inspect per-entry results. End-to-end completion requires consumer evidence too.

**Can EventBridge preserve order?** New Custom Event Bus supports FIFO subscribers per group; Classic does not. Downstream invocation mode still affects processing order.

**Why use an outbox?** A database commit and event publish are separate operations. The outbox makes committed business changes eventually publishable after a crash.

Remember: product + generation + target invocation mode determine the guarantee. State all three in an interview answer.

## 12. Message formats and Python and Java examples

### Setup and which API these examples use

The runnable examples below use the **default/Classic event-bus API**: Python `boto3.client("events")` and Java SDK 2.x `EventBridgeClient`. They assume an existing bus, matching rule, and authorized SQS or Lambda target. Publishing alone does not create routing or attach consumers.

Set `AWS_REGION` and `EVENT_BUS_NAME` (bus name or ARN). Use a profile/workload role through the SDK default credential chain; the publisher needs `events:PutEvents`. Python needs Boto3. Java 17+ needs `software.amazon.awssdk:eventbridge`, Jackson 2.x `com.fasterxml.jackson.core:jackson-databind`, and `com.amazonaws:aws-lambda-java-core` for the handler. Pin compatible dependencies; align AWS modules with an approved SDK 2.x BOM.

**Java SDK 2.x is a library version; EventBridgeV2 is a different service API.** For the newer Custom Event Bus, use its `eventsv2`/`EventBridgeV2` client, publisher schema, and subscriber configuration. Do not send the Classic request below through the newer client unchanged. See [API naming](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-names.html).

### Request, response, and target event are three different shapes

**1. Publish request:** `Source`, `DetailType`, and `Detail` use API field names. `Detail` is a **JSON-encoded string**, not a Python dictionary or Java map passed directly to the SDK field.

```json
{
  "Entries": [
    {
      "EventBusName": "orders",
      "Source": "com.example.orders",
      "DetailType": "OrderCreated",
      "Detail": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}"
    }
  ]
}
```

**2. Publish response:** entries correspond by position to request entries. Successful entries contain an `EventId`; failed entries contain `ErrorCode`/`ErrorMessage`. This reduced example shows success:

```json
{
  "FailedEntryCount": 0,
  "Entries": [{"EventId": "eventbridge-generated-id"}]
}
```

**3. Full original event at the target:** EventBridge adds envelope metadata and deserializes `Detail` into the nested **`detail` object**. This is an illustrative target event; generated IDs, account, time, and Region depend on the actual publish.

```json
{
  "version": "0",
  "id": "eventbridge-generated-id",
  "detail-type": "OrderCreated",
  "source": "com.example.orders",
  "account": "123456789012",
  "time": "2026-10-13T10:00:00Z",
  "region": "ap-south-1",
  "resources": [],
  "detail": {
    "schemaVersion": 1,
    "eventId": "evt-order-456-created",
    "eventType": "OrderCreated",
    "orderId": "456",
    "country": "IN",
    "total": 1200
  }
}
```

`id` is the transport event ID; `detail.eventId` is your stable business ID. The application schema version is `detail.schemaVersion`, distinct from the envelope's `version`. [Event structure](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-events.html), [Boto3 PutEvents](https://docs.aws.amazon.com/boto3/latest/reference/services/events/client/put_events.html)

### Python: publish and surface partial failures

```python
import json
import os
import boto3


def publish_order():
    order = {
        "schemaVersion": 1, "eventId": "evt-order-456-created",
        "eventType": "OrderCreated", "orderId": "456",
        "country": "IN", "total": 1200,
    }
    entries = [{
        "EventBusName": os.environ["EVENT_BUS_NAME"],
        "Source": "com.example.orders",
        "DetailType": "OrderCreated",
        "Detail": json.dumps(order),
    }]
    events = boto3.client("events", region_name=os.environ["AWS_REGION"])
    response = events.put_events(Entries=entries)
    failed = [
        {"entry": entry, "errorCode": result["ErrorCode"],
         "errorMessage": result.get("ErrorMessage")}
        for entry, result in zip(entries, response["Entries"])
        if result.get("ErrorCode")
    ]
    if failed:
        # Caller must persist/classify these failures and retry only eligible entries.
        raise RuntimeError({"failedEntries": failed})
    print("EventBridge accepted:", [r["EventId"] for r in response["Entries"]])


if __name__ == "__main__":
    publish_order()
```

The one-entry example uses the batch API so the same check scales to multiple entries. HTTP-success responses can contain failures; automatic SDK retries of request-level errors do not replace per-entry inspection. For retryable entry errors, retry only those entries with bounded backoff/jitter and the same business ID. Preserve failed work durably, for example through an outbox. Do not blindly rerun an entire partly successful batch. [PutEvents failure handling](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-putevents.html)

### Python: receive in a Lambda target

```python
def lambda_handler(event, context):
    if event["source"] != "com.example.orders" or event["detail-type"] != "OrderCreated":
        raise ValueError("Unexpected EventBridge event")
    order = event["detail"]  # Already a dict: no json.loads() here.
    if order["schemaVersion"] != 1:
        raise ValueError("Unsupported schema version")
    # Demo only: replace with durable, idempotent business processing.
    print("Order:", order["orderId"], "business event:", order["eventId"])
```

There is no SQS-style `receive_message` call for this Classic bus. EventBridge routes events to the target. A direct Lambda target receives the event itself, **not** `Records[0]`; an SQS target stores the serialized event in its message body.

### Java: publish and receive in Lambda

Save as `EventBridgeExample.java`. `main` publishes; configure `EventBridgeExample::handleRequest` for a Lambda target receiving the original event.

```java
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequest;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequestEntry;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class EventBridgeExample implements RequestHandler<Map<String, Object>, Void> {
    private static final ObjectMapper JSON = new ObjectMapper();

    public static void main(String[] args) throws Exception {
        String detail = JSON.writeValueAsString(Map.of(
            "schemaVersion", 1, "eventId", "evt-order-456-created",
            "eventType", "OrderCreated", "orderId", "456",
            "country", "IN", "total", 1200));
        List<PutEventsRequestEntry> entries = List.of(PutEventsRequestEntry.builder()
            .eventBusName(System.getenv("EVENT_BUS_NAME"))
            .source("com.example.orders").detailType("OrderCreated")
            .detail(detail).build());
        try (EventBridgeClient events = EventBridgeClient.builder()
                .region(Region.of(System.getenv("AWS_REGION"))).build()) {
            var response = events.putEvents(PutEventsRequest.builder().entries(entries).build());
            List<PutEventsRequestEntry> failedEntries = new ArrayList<>();
            for (int i = 0; i < response.entries().size(); i++) {
                var result = response.entries().get(i);
                if (result.errorCode() != null) {
                    failedEntries.add(entries.get(i));
                    System.err.println("Entry " + i + ": " + result.errorCode()
                        + " " + result.errorMessage());
                } else {
                    System.out.println("EventBridge accepted: " + result.eventId());
                }
            }
            if (!failedEntries.isEmpty()) {
                // Persist/classify these entries before retrying only retryable failures.
                throw new IllegalStateException("Publish failed for "
                    + failedEntries.size() + " entry/entries");
            }
        }
    }

    @Override
    public Void handleRequest(Map<String, Object> event, Context context) {
        JsonNode root = JSON.valueToTree(event);
        if (!"com.example.orders".equals(root.required("source").asText())
            || !"OrderCreated".equals(root.required("detail-type").asText())) {
            throw new IllegalArgumentException("Unexpected EventBridge event");
        }
        JsonNode order = root.required("detail"); // Already a nested object.
        if (order.required("schemaVersion").asInt() != 1) {
            throw new IllegalArgumentException("Unsupported schema version");
        }
        // Demo only: replace with durable, idempotent business processing.
        context.getLogger().log("Order " + order.required("orderId").asText()
            + ", business event " + order.required("eventId").asText() + "\n");
        return null;
    }
}
```

[Java EventBridge SDK examples](https://docs.aws.amazon.com/code-library/latest/ug/java_2_eventbridge_code_examples.html), [Java Lambda handler contract](https://docs.aws.amazon.com/lambda/latest/dg/java-handler.html)

### Target configuration determines the received shape

| Target setup | Received value | Python extraction |
|---|---|---|
| Lambda, original entire event | EventBridge envelope as an object | `event["detail"]` |
| SQS, original entire event | Envelope serialized inside SQS `Body` | `json.loads(message["Body"])["detail"]` |
| Target `InputPath` selecting `$.detail` | Only the business payload | Lambda: `event`; SQS: parse `Body` once |
| Target input transformer | Your configured output template | Parse the deliberately defined target contract |
| EventBridge → SQS → Lambda | SQS `Records[].body` contains the event envelope | Parse record `body`, then take `detail` |

For a whole-event SQS target, use `BODY_MODE=eventbridge` with the [Python/Java SQS examples](aws-sqs-cheatsheet.md#12-message-formats-and-python-and-java-examples). If you deliberately select only `$.detail`, use `BODY_MODE=direct`. Matching is performed against the event; target transformations affect delivery format. [Target input transformation](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-transform-target-input.html)

The Classic rule pattern in section 4 matches this sample (`country=IN`, `total=1200`). Configure target permissions separately: a queue policy for the SQS target, or a Lambda invocation resource policy for the rule. A successful publish with zero matching rules is not a consumer success.

### Fast interview comparison

| Service | Send field | Receive location | Acknowledge/complete |
|---|---|---|---|
| SNS | `Message` string | Depends on subscription; e.g. `Records[].Sns.Message` | Subscriber-specific; no SNS polling/delete API |
| SQS | `MessageBody` string | SDK `Messages[].Body` or Lambda `Records[].body` | Poller deletes receipt handle; mapping handles successful Lambda records |
| EventBridge Classic | `Detail` JSON string plus routing fields | Target event `detail` object unless transformed | Target-specific delivery and processing policies |

## 13. AWS sources

- [New Custom Event Bus overview](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-what-is.html)
- [API/CLI/IAM naming](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-names.html)
- [Ordering and deduplication](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-ordering.html)
- [Subscriber retries and DLQs](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-retry.html)
- [Classic publishing and payload limits](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-putevents.html)
- [Classic delivery monitoring](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-monitoring-events-best-practices.html)
- [Pipes](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html)
- [Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [Event envelope](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-events.html)
- [Boto3 PutEvents](https://docs.aws.amazon.com/boto3/latest/reference/services/events/client/put_events.html)
- [Java SDK EventBridge examples](https://docs.aws.amazon.com/code-library/latest/ug/java_2_eventbridge_code_examples.html)
- [Java Lambda handlers](https://docs.aws.amazon.com/lambda/latest/dg/java-handler.html)
- [Target input transformation](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-transform-target-input.html)

**Related:** [SNS](aws-sns-cheatsheet.md), [SQS](aws-sqs-cheatsheet.md), [Lambda](aws-lambda-cheatsheet.md).
