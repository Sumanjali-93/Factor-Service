# Factor-Service


A backend service for TOTP-based multi-factor authentication (RFC 6238), covering enrollment, verification, lockout, and recovery codes. Written in Go using Clean Architecture with DDD-lite domain modeling — the domain layer has zero external dependencies and is fully unit-testable in isolation.

[![Go Reference](https://pkg.go.dev/badge/github.com/your-org/mfa-service.svg)](https://pkg.go.dev/github.com/your-org/mfa-service)
[![Go Report Card](https://goreportcard.com/badge/github.com/your-org/mfa-service)](https://goreportcard.com/report/github.com/your-org/mfa-service)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

## Architecture

```
┌──────────────────────────────────────────────┐
│ Interface Layer                                │
│   adapters/grpc, adapters/http                 │
└────────────────────┬───────────────────────────┘
                     │ depends on
┌────────────────────▼───────────────────────────┐
│ Application Layer (use cases)                  │
│   EnrollFactor · ActivateFactor · VerifyCode    │
│   DisableFactor · RegenerateRecoveryCodes       │
│   defines ports: Repository, Clock, ReplayGuard,│
│   SecretEncryptor                               │
└────────────────────┬───────────────────────────┘
                     │ depends on
┌────────────────────▼───────────────────────────┐
│ Domain Layer (stdlib only)                      │
│   Factor (aggregate) · Secret, Code (VOs)       │
│   totp.Generator (domain service)               │
└────────────────────▲───────────────────────────┘
                     │ implements ports
┌────────────────────┴───────────────────────────┐
│ Infrastructure Layer                            │
│   postgres.FactorRepository · redis.ReplayGuard │
│   kms.EnvelopeEncryptor · clock.SystemClock     │
└──────────────────────────────────────────────┘
```

Dependencies point inward. The domain package imports nothing outside the standard library, so the TOTP algorithm and the `Factor` state machine are tested without Postgres, Redis, or a KMS emulator.

## Domain Model

| Type | Kind | Responsibility |
|---|---|---|
| `Factor` | Aggregate root | Lifecycle state machine, owns its `Secret` |
| `Secret` | Value object | Immutable key material, never logged or serialized in plaintext |
| `Code` | Value object | 6-digit OTP, valid within a bounded time-step window |
| `RecoveryCode` | Entity | Single-use hashed backup code |
| `totp.Generator` | Domain service | RFC 6238 code generation and validation |

```
pending --(activate with valid code)--> active --(disable)--> disabled
```

## API

Base path: `/v1`

| Method | Path | Purpose |
|---|---|---|
| POST | `/factors` | Enroll a new TOTP factor; returns secret + `otpauth://` URI |
| POST | `/factors/{id}/activate` | Confirm enrollment with the first generated code |
| POST | `/factors/{id}/verify` | Verify a code at authentication time |
| DELETE | `/factors/{id}` | Disable a factor |
| POST | `/factors/{id}/recovery-codes` | Issue a new set of single-use recovery codes |
| POST | `/recovery/verify` | Redeem a recovery code in place of TOTP |

```
POST /v1/factors
{"account_id": "acc_123"}

201
{
  "factor_id": "fct_9f2a",
  "status": "pending",
  "provisioning_uri": "otpauth://totp/MFA:acc_123?secret=...&issuer=MFA&algorithm=SHA1&digits=6&period=30"
}
```

```
POST /v1/factors/fct_9f2a/verify
{"code": "493217"}

200 {"valid": true}
401 {"valid": false, "reason": "code_reused" | "code_invalid" | "factor_locked"}
```

## Security Model

- **Secret storage** — each TOTP secret is encrypted with a per-record data key (DEK), wrapped by a KMS-managed key (KEK). Plaintext secrets exist only in memory during enrollment and verification.
- **Replay protection** — every accepted code is recorded in Redis as `(factor_id, time_step)` with a TTL equal to the validation window, rejecting repeats even if the code is still mathematically valid.
- **Timing-safe comparison** — code comparison uses `crypto/subtle.ConstantTimeCompare` to eliminate timing side-channels.
- **Lockout** — 5 consecutive failed verifications lock a factor with exponential backoff (1m → 5m → 30m, capped at 24h), tracked per `factor_id`.
- **Recovery codes** — 10 cryptographically random codes per set, stored as Argon2id hashes, each single-use and invalidated on redemption.
- **Audit trail** — enroll, activate, verify (success/failure), disable, and lockout events are emitted as structured log entries (`factor_id`, `account_id`, `result`, `client_ip`) for an external audit pipeline.

## Reliability

- **Clock skew tolerance** — codes are validated against a configurable ±1 time-step window (default step 30s → 90s effective tolerance).
- **Idempotent enrollment** — `POST /factors` honors an `Idempotency-Key` header; retries return the original result instead of creating duplicate factors.
- **Fail-closed on cache outage** — if the Redis-backed replay guard is unreachable, verification requests are rejected rather than allowed, favoring lockout over a security bypass.
- **Observability** — Prometheus metrics (`mfa_verify_total{result}`, `mfa_verify_latency_seconds`, `mfa_lockouts_total`) and OpenTelemetry tracing across the use-case → repository boundary.

## Design Decisions

- **TOTP over SMS** — avoids SIM-swap and SS7 interception risk; no telephony provider sits on the security-critical path.
- **Domain has zero external imports** — `internal/domain` depends only on the standard library, keeping the algorithm and state machine deterministically testable.
- **Ports live in the application layer, not the domain** — the domain stays focused on business rules; the application layer owns the contracts (`Repository`, `Clock`, `ReplayGuard`, `SecretEncryptor`) infrastructure must satisfy.
- **Envelope encryption over per-call KMS** — a short-lived DEK is decrypted once per factor load rather than on every verification, trading a brief in-memory plaintext window for lower latency and KMS cost under load.
- **Clock injected as a port** — all time-dependent logic reads from an injected `Clock`, making step-boundary edge cases reproducible in tests.

## Project Layout

```
mfa-service/
├── cmd/server/main.go
├── internal/
│   ├── domain/
│   │   ├── factor/      # Factor aggregate, state transitions
│   │   ├── totp/         # RFC 6238 generator/validator (stdlib only)
│   │   └── recovery/      # Recovery code entity
│   ├── application/
│   │   ├── enroll_factor.go
│   │   ├── activate_factor.go
│   │   ├── verify_code.go
│   │   ├── disable_factor.go
│   │   └── ports.go       # Repository, Clock, ReplayGuard, SecretEncryptor
│   ├── infrastructure/
│   │   ├── postgres/      # FactorRepository implementation
│   │   ├── redis/         # ReplayGuard implementation
│   │   ├── kms/            # SecretEncryptor implementation
│   │   └── clock/          # SystemClock implementation
│   └── adapters/
│       ├── grpc/
│       └── http/
├── api/proto/mfa.proto
├── migrations/
├── deploy/
│   ├── Dockerfile
│   └── k8s/
└── go.mod
```

## Stack

Go 1.22 · gRPC + grpc-gateway · PostgreSQL · Redis · KMS (AWS/GCP, pluggable) · Prometheus · OpenTelemetry

## Running Locally

```bash
docker compose up -d postgres redis
make migrate
go run ./cmd/server
```

```bash
grpcurl -plaintext -d '{"account_id": "acc_123"}' \
  localhost:8080 mfa.v1.FactorService/EnrollFactor
```

## Testing

```bash
go test ./internal/domain/...       # pure unit tests, no infra
go test ./internal/application/...  # use cases against in-memory port fakes
go test ./... -tags=integration     # full stack via testcontainers
```

## License

MIT
