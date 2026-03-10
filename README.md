
# API Integration Troubleshooting Lab

A small backend API built with Flask that simulates common API
integration failures such as authentication errors, malformed XML
payloads, and incorrect request headers.

This repository is part of the **API Troubleshooting Lab Series**, a
multi-service environment designed to demonstrate real-world API
integration debugging and platform support workflows.

Companion repository:

API Gateway Troubleshooting Lab  
https://github.com/GregoryCarberry/api-gateway-troubleshooting-lab

---

# Lab Series Architecture

![API Troubleshooting Lab Series Architecture](docs/api-troubleshooting-lab-shared-architecture.svg)

The backend API sits behind the gateway layer. The gateway handles
platform-level concerns such as authentication, rate limiting and
request tracing before forwarding requests to this backend service.

---

# Quick Start

1. Start the backend API
2. Start the API Gateway
3. Send requests via Postman or curl
4. Reproduce troubleshooting scenarios

---

# What This Project Demonstrates

This lab simulates **real-world API integration troubleshooting** rather
than simple CRUD development.

Skills demonstrated:

- diagnosing authentication failures
- validating XML payloads
- identifying incorrect request headers
- tracing requests using correlation IDs
- isolating gateway vs backend failures
- analysing structured logs

---

# Failure Scenarios

| Scenario | Layer | Response |
|--------|--------|--------|
| Missing API key | Gateway | 401 |
| Invalid API key | Gateway | 403 |
| Rate limit exceeded | Gateway | 429 |
| Malformed XML | Backend | 400 |
| Backend unavailable | Gateway | 502 |
| Backend timeout | Gateway | 504 |

---

# Troubleshooting Workflow Example

Example debugging sequence:

Client request fails  
↓  
Gateway returns **502 Bad Gateway**  
↓  
Check gateway logs for **X-Request-ID**  
↓  
Trace the same request ID in backend logs  
↓  
Backend log reveals malformed XML payload  
↓  
Fix request payload and retry successfully

---

# Observability

Requests include a correlation ID using the **X-Request-ID** header.

The same ID appears in:

- gateway logs
- backend logs
- HTTP responses

This allows requests to be traced across services during troubleshooting.

---

# Repository Structure

api-integration-troubleshooting-lab/

├── app.py  
├── requirements.txt  
├── README.md  

├── docs/  
│   └── api-troubleshooting-lab-shared-architecture.svg  

├── examples/  
│   ├── valid-order.xml  
│   └── malformed-order.xml  

├── postman/  
├── screenshots/  
└── tests/  
    └── api_test.py  

---

# Running the Backend API

Create a virtual environment and install dependencies:

    python -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt

Start the API:

    python app.py

Run troubleshooting tests:

    python tests/api_test.py

---

# Lab Exercises

Example troubleshooting exercises:

- Diagnose a malformed XML payload
- Identify a missing API key error
- Trace a request using correlation IDs
- Investigate backend log messages

---

# System Design Notes

The project intentionally separates the **gateway** and **backend**
services into different repositories.

This reflects how real API platforms separate:

- platform infrastructure (authentication, rate limiting)
- application logic (payload validation, business processing)

---

# Future Improvements

Possible extensions:

- OAuth authentication support
- JSON API support
- distributed tracing integration
- containerised test environment
