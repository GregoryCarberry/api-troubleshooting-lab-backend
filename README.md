# Backend API (Flask)

This service is the backend component of the API Troubleshooting Lab. It is responsible for processing XML-based order requests, validating input, simulating failures, and providing structured logging to support debugging and observability.

## Overview

The backend is intentionally designed to mimic a real-world service that:

- accepts structured input using XML
- validates requests and returns meaningful errors
- simulates downstream failures
- supports request tracing across services

It is consumed by the API Gateway, which handles authentication, rate limiting, and external access.

---

## Quick Start

From the backend repository root:

```bash
cd ~/api-troubleshooting-lab/api-troubleshooting-lab-backend
```

Create and activate a virtual environment:

```bash
python -m venv venv
source venv/bin/activate
```

Install dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Start the backend service:

```bash
python app.py
```

By default, the backend runs on:

```text
http://127.0.0.1:5000
```

Health check:

```bash
curl http://127.0.0.1:5000/health
```

Expected response:

```text
ok
```

---

## Local Test Request

Send a valid order directly to the backend:

```bash
curl -i -X POST http://127.0.0.1:5000/api/orders \
  -H "Content-Type: application/xml" \
  --data-binary @examples/valid-order.xml
```

Expected successful response:

```http
HTTP/1.1 201 CREATED
```

```xml
<OrderCreated><OrderID>generated-order-id</OrderID></OrderCreated>
```

The exact order ID is generated at runtime.

---

## Running the Full Lab

For normal end-to-end testing, run both services:

1. Start the backend on `http://127.0.0.1:5000`
2. Start the gateway on `http://127.0.0.1:8000`
3. Send client requests to the gateway, not directly to the backend

Normal client entry point:

```text
POST http://127.0.0.1:8000/orders
```

Direct backend testing is useful for isolating backend validation and failure behaviour, but the gateway is the intended external entry point for the full troubleshooting lab.

---

## Responsibilities

- XML request parsing and validation
- order-processing simulation
- failure simulation for testing resilience
- structured logging
- request ID propagation for tracing

---

## Request Flow

1. Request received, typically via the gateway
2. `X-Request-ID` extracted or generated
3. XML payload parsed and validated
4. Failure mode applied if specified
5. Response returned with the appropriate status code

---

## Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Backend health check |
| `POST` | `/api/orders` | Create an order from XML payload |
| `GET` | `/api/orders/<order_id>` | Retrieve an order from the in-memory store |

---

## Expected XML Payload

A valid order request uses the following structure:

```xml
<Order>
  <CustomerID>12345</CustomerID>
  <ProductID>ABC123</ProductID>
  <Quantity>2</Quantity>
</Order>
```

Required fields:

| Field | Description |
|---|---|
| `CustomerID` | Customer identifier for the order |
| `ProductID` | Product identifier for the requested item |
| `Quantity` | Positive integer quantity |

---

## Validation Rules

The API enforces strict validation:

- valid XML structure required
- required fields must be present
- required fields must not be empty
- `Quantity` must be a positive integer

Common responses:

| Scenario | Status |
|---|---|
| malformed XML | 400 |
| missing fields | 422 |
| empty required fields | 422 |
| invalid quantity | 422 |
| unsupported content type | 415 |

---

## Failure Modes

The backend supports controlled failure simulation via the `X-Failure-Mode` header.

| Mode | Behaviour | Status |
|---|---|---|
| `timeout` | Simulated delay | 504 |
| `dependency` | Simulated upstream/downstream dependency failure | 503 |
| `exception` | Simulated internal backend error | 500 |

This allows consistent reproduction of failure scenarios for testing and troubleshooting.

Example:

```bash
curl -i -X POST http://127.0.0.1:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Failure-Mode: dependency" \
  --data-binary @examples/valid-order.xml
```

---

## Observability

The backend emits structured JSON logs including:

- request ID (`X-Request-ID`)
- request details
- validation errors
- failure mode triggers

This enables:

- correlation with gateway logs
- end-to-end request tracing
- easier debugging of failures

To pass a known request ID during direct backend testing:

```bash
curl -i -X POST http://127.0.0.1:5000/api/orders \
  -H "Content-Type: application/xml" \
  -H "X-Request-ID: backend-demo-001" \
  --data-binary @examples/valid-order.xml
```

---

## Example Successful Response

A valid request returns an XML response similar to:

```xml
<OrderCreated>
  <OrderID>generated-order-id</OrderID>
</OrderCreated>
```

The exact order ID is generated at runtime.

---

## Testing

The backend includes a pytest-based test suite covering:

- valid requests
- validation errors
- failure modes
- request tracing behaviour

Run tests:

```bash
pytest -q
```

If `pytest` is not installed in the virtual environment, install it before running the test suite:

```bash
python -m pip install pytest
```

---

## Relationship to Gateway

This service is not intended to be accessed directly in normal operation.

The API Gateway provides:

- authentication
- rate limiting
- request routing
- upstream timeout handling
- unified entry point

The backend focuses purely on service logic, validation, failure simulation, and trace-aware responses.

---

## Purpose

This service exists to demonstrate:

- input validation and error handling
- controlled failure simulation
- observability and tracing
- behaviour under different failure conditions

It is designed as part of a multi-service system to replicate real-world troubleshooting scenarios.
