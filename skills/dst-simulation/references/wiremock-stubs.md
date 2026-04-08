# WireMock Stub Patterns for DST

Mock external APIs so the generated system can run locally without real credentials.
Place stub files in `./wiremock/mappings/`.

## Stripe

```json
{
  "request": {"method": "POST", "urlPath": "/v1/charges"},
  "response": {
    "status": 200,
    "headers": {"Content-Type": "application/json"},
    "jsonBody": {"id": "ch_test_123", "status": "succeeded", "amount": 2000, "currency": "usd"}
  }
}
```

## SendGrid / Email

```json
{
  "request": {"method": "POST", "urlPath": "/v3/mail/send"},
  "response": {"status": 202, "body": ""}
}
```

## Auth0 / OAuth token

```json
{
  "request": {"method": "POST", "urlPath": "/oauth/token"},
  "response": {
    "status": 200,
    "headers": {"Content-Type": "application/json"},
    "jsonBody": {"access_token": "test_token_123", "token_type": "Bearer", "expires_in": 86400}
  }
}
```

## S3-compatible (MinIO as alternative)

For S3, prefer using MinIO as a real local service instead of WireMock:
```yaml
minio:
  image: minio/minio
  command: server /data
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin
  ports: ["9000:9000"]
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 5s
    timeout: 3s
    retries: 5
```

## Generic REST API (catchall)

```json
{
  "request": {"method": "ANY", "urlPattern": "/api/.*"},
  "response": {
    "status": 200,
    "headers": {"Content-Type": "application/json"},
    "jsonBody": {"status": "ok", "mock": true}
  }
}
```

## Fault responses (for testing error handling)

```json
{
  "request": {"method": "ANY", "urlPattern": "/v1/.*"},
  "response": {
    "status": 500,
    "headers": {"Content-Type": "application/json"},
    "jsonBody": {"error": {"type": "api_error", "message": "Internal server error"}},
    "fixedDelayMilliseconds": 3000
  }
}
```
