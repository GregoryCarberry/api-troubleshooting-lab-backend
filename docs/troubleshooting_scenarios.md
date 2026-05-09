# Troubleshooting Scenarios

This document captures reproducible troubleshooting scenarios for the **backend service** in the API Troubleshooting Lab.

The backend is a Flask service that processes XML order requests, validates payloads, simulates controlled failures, and returns trace-aware responses using `X-Request-ID`.

These scenarios are designed to support:

- manual testing with curl
- automated test design
- debugging practice
- comparison between direct backend behaviour and gateway-mediated behaviour

---

## Scope

This document focuses on failures that originate within the backend application itself:

- malformed XML payloads
- request validation failures
- unsupported content types
- invalid failure simulation headers
- simulated dependency failures
- simulated timeouts
- simulated internal exceptions
- resource lookup failures

Gateway-specific behaviour such as API key authentication and rate limiting is covered by the gateway service and the hub Postman collection.

---

## Backend Endpoint Reference

### Health check

```text
GET /health
```

### Create order

```text
POST /api/orders
```

### Retrieve order

```text
GET /api/orders/<order_id>
```

---

## Valid Order Payload

A valid order request must use this XML structure:

```xml
<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>
```

Required fields:

| Field | Requirement |
|---|---|
| `CustomerID` | must exist and must not be empty |
| `ProductID` | must exist and must not be empty |
| `Quantity` | must exist, must not be empty, and must be a positive integer |

---

## Scenario 1: Successful Order Creation

### Symptom

The backend returns `201 Created`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-success-001" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 201 CREATED
Content-Type: application/xml
X-Request-ID: demo-success-001
```

```xml
<OrderCreated><OrderID>generated-order-id</OrderID></OrderCreated>
```

### Notes

The exact `OrderID` is generated at runtime.

---

## Scenario 2: Malformed XML

### Symptom

The backend returns `400 Bad Request`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-malformed-xml-001" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>'
```

### Expected response

```http
HTTP/1.1 400 BAD REQUEST
Content-Type: text/plain
X-Request-ID: demo-malformed-xml-001
```

```text
Bad Request: Malformed XML: no element found: line 4, column 24
```

### Cause

The XML body is not well formed. In this example, the closing `</Order>` tag is missing.

### Troubleshooting approach

1. Check that every opening tag has a valid closing tag.
2. Validate the payload in an XML-aware editor.
3. Retest with `examples/valid-order.xml`.
4. Check backend logs using the `X-Request-ID`.

---

## Scenario 3: Missing Required Field

### Symptom

The backend returns `422 Unprocessable Entity`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-missing-product-001" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 422 UNPROCESSABLE ENTITY
Content-Type: text/plain
X-Request-ID: demo-missing-product-001
```

```text
Unprocessable Entity: Invalid XML: missing or empty <ProductID>
```

### Cause

The XML is well formed, but the backend requires a `ProductID` element.

### Troubleshooting approach

1. Compare the request against the valid payload structure.
2. Confirm all required fields are present.
3. Check for incorrect field names or casing.
4. Retest with `examples/missing-product-id.xml`.

---

## Scenario 4: Empty Required Field

### Symptom

The backend returns `422 Unprocessable Entity`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-empty-field-001" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID></ProductID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 422 UNPROCESSABLE ENTITY
Content-Type: text/plain
X-Request-ID: demo-empty-field-001
```

```text
Unprocessable Entity: Invalid XML: missing or empty <ProductID>
```

### Cause

The `ProductID` element exists but contains no usable value.

### Troubleshooting approach

1. Check for empty XML elements.
2. Confirm the client is not submitting blank form/input values.
3. Retest with a known-good payload.

---

## Scenario 5: Invalid Quantity

### Symptom

The backend returns `422 Unprocessable Entity`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-invalid-quantity-001" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>zero</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 422 UNPROCESSABLE ENTITY
Content-Type: text/plain
X-Request-ID: demo-invalid-quantity-001
```

```text
Unprocessable Entity: Invalid XML: <Quantity> must be a positive integer
```

### Cause

The `Quantity` value must be a positive integer.

### Troubleshooting approach

1. Confirm `Quantity` is numeric.
2. Confirm `Quantity` is greater than zero.
3. Check for whitespace, decimal values, negative values, or words.
4. Retest with `examples/invalid-quantity.xml`.

---

## Scenario 6: Wrong Root Element

### Symptom

The backend returns `422 Unprocessable Entity`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-wrong-root-001" \
  -d '<Purchase>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Purchase>'
```

### Expected response

```http
HTTP/1.1 422 UNPROCESSABLE ENTITY
Content-Type: text/plain
X-Request-ID: demo-wrong-root-001
```

```text
Unprocessable Entity: Invalid XML: root element must be <Order>
```

### Cause

The backend expects the XML root element to be `<Order>`.

### Troubleshooting approach

1. Confirm the root element is exactly `<Order>`.
2. Check XML casing.
3. Retest with `examples/wrong-root.xml`.

---

## Scenario 7: Unsupported Content Type

### Symptom

The backend returns `415 Unsupported Media Type`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: demo-wrong-content-type-001" \
  -d '{"CustomerID":"12345","ProductID":"ABC123","Quantity":2}'
```

### Expected response

```http
HTTP/1.1 415 UNSUPPORTED MEDIA TYPE
Content-Type: text/plain
X-Request-ID: demo-wrong-content-type-001
```

```text
Unsupported Media Type: Content-Type must be application/xml
```

### Cause

The backend expects XML but the request declares JSON.

### Troubleshooting approach

1. Confirm the `Content-Type` header is set to `application/xml`.
2. Confirm the request body is actually XML.
3. Check client defaults in Postman, curl, scripts, or application code.

---

## Scenario 8: Empty Request Body

### Symptom

The backend returns `400 Bad Request`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-empty-body-001"
```

### Expected response

```http
HTTP/1.1 400 BAD REQUEST
Content-Type: text/plain
X-Request-ID: demo-empty-body-001
```

```text
Bad Request: request body is empty
```

### Cause

The request contains no XML body.

### Troubleshooting approach

1. Confirm the client is sending a body.
2. Check curl `-d`, Postman Body settings, or application request construction.
3. Retest with `examples/valid-order.xml`.

---

## Scenario 9: Invalid Failure Mode Header

### Symptom

The backend returns `400 Bad Request`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-invalid-failure-mode-001" \
  -H "X-Failure-Mode: database" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 400 BAD REQUEST
Content-Type: text/plain
X-Request-ID: demo-invalid-failure-mode-001
```

```text
Bad Request: Invalid X-Failure-Mode. Allowed values: none, timeout, dependency, exception
```

### Cause

The backend only accepts these failure modes:

```text
none, timeout, dependency, exception
```

### Troubleshooting approach

1. Check the spelling of the `X-Failure-Mode` value.
2. Confirm the client is not sending an unexpected default value.
3. Remove the header entirely if no failure simulation is required.

---

## Scenario 10: Simulated Dependency Failure

### Trigger

Set:

```text
X-Failure-Mode: dependency
```

### Symptom

The backend returns `503 Service Unavailable`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-dependency-001" \
  -H "X-Failure-Mode: dependency" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 503 SERVICE UNAVAILABLE
Content-Type: text/plain
X-Request-ID: demo-dependency-001
```

```text
Service Unavailable: Simulated upstream dependency failure
```

### What this simulates

This represents a dependency becoming unavailable, such as:

- a database outage
- an internal service failure
- a third-party API outage

### Troubleshooting approach

1. Confirm the failure only occurs when the dependency mode is triggered.
2. Check backend logs using the request ID.
3. Distinguish dependency failure from malformed input or application bugs.

---

## Scenario 11: Simulated Timeout

### Trigger

Set:

```text
X-Failure-Mode: timeout
```

### Symptom

The backend returns `504 Gateway Timeout` after a simulated delay.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-timeout-001" \
  -H "X-Failure-Mode: timeout" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 504 GATEWAY TIMEOUT
Content-Type: text/plain
X-Request-ID: demo-timeout-001
```

```text
Gateway Timeout: Simulated upstream timeout occurred
```

### What this simulates

This represents a slow backend path or slow dependency.

### Troubleshooting approach

1. Confirm whether the request fails immediately or only after a delay.
2. Compare backend processing time with gateway timeout settings.
3. Review gateway and backend logs using the same request ID.
4. Check whether timeout behaviour differs when accessed through the gateway.

---

## Scenario 12: Simulated Internal Exception

### Trigger

Set:

```text
X-Failure-Mode: exception
```

### Symptom

The backend returns `500 Internal Server Error`.

### Example request

```bash
curl -i -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: demo-exception-001" \
  -H "X-Failure-Mode: exception" \
  -d '<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>'
```

### Expected response

```http
HTTP/1.1 500 INTERNAL SERVER ERROR
Content-Type: text/plain
X-Request-ID: demo-exception-001
```

```text
Internal Server Error: simulated backend exception
```

### What this simulates

This represents an unhandled backend runtime error.

### Troubleshooting approach

1. Check backend logs for exception details.
2. Use the request ID to correlate the failed request.
3. Distinguish bad input from a genuine server-side defect.
4. Reproduce with the smallest possible request.

---

## Scenario 13: Order Not Found

### Symptom

The backend returns `404 Not Found`.

### Example request

```bash
curl -i http://localhost:5000/api/orders/does-not-exist \
  -H "X-Request-ID: demo-not-found-001"
```

### Expected response

```http
HTTP/1.1 404 NOT FOUND
Content-Type: text/plain
X-Request-ID: demo-not-found-001
```

```text
Not Found
```

### Cause

The requested order ID does not exist in the backend's in-memory store.

### Troubleshooting approach

1. Confirm the order was created successfully.
2. Check for typos or truncated order IDs.
3. Remember that in-memory storage resets when the backend restarts.
4. Recreate the order and retry the lookup.

---

## Direct Backend Testing vs Gateway Testing

These examples call the backend directly on:

```text
http://localhost:5000
```

In normal lab usage, clients call the gateway on:

```text
http://localhost:8000
```

The gateway then forwards valid requests to the backend.

For end-to-end API demonstrations, use the Postman collection in the hub repository:

```text
api-troubleshooting-lab/postman/API Troubleshooting Lab.postman_collection.json
```

---

## Notes

These scenarios are intentionally simple and reproducible.

They are designed to support:

- manual testing with curl
- automated tests
- troubleshooting documentation
- gateway/backend comparison
- future observability improvements
