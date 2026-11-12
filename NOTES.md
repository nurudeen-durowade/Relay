DO NOT SHIP

# Relay Assessment - Engineering Notes

## 1. Executive decision

DO NOT SHIP to production in its current state.

The application passed local 150 RPS and 400 RPS load tests after correcting database pool sizing and implementing PostgreSQL-backed idempotency.

However, production infrastructure and security blockers remain.

## 2. Application improvements

- Corrected application port configuration to 8080.
- Corrected Kubernetes container ports, health probes and Service target port.
- Corrected the PostgreSQL Secret key reference.
- Changed local image pull policy to IfNotPresent and loaded the image into Kind.
- Added CPU and memory requests and limits.
- Reduced PostgreSQL pool allocation from 25 to 8 connections per application replica.
- Replaced in-memory idempotency with atomic PostgreSQL INSERT ... ON CONFLICT DO NOTHING RETURNING id.
- Removed obsolete in-memory deduplication configuration.
- Verified duplicate requests return HTTP 409 and preserve the original event ID.
- Hardened Docker build context using .dockerignore.
- Configured the container to run as the non-root node user.
- Adjusted Node.js heap configuration to 160 MiB.

## 3. Troubleshooting and incidents

1. ImagePullBackOff: local Kind nodes could not access relay:latest. Loaded image into all Kind nodes and configured IfNotPresent.
2. HPA metrics unavailable: Metrics API was unavailable in Kind. Autoscaling could not be verified.
3. PostgreSQL readiness delay: database initially needed time to start; subsequently became healthy.
4. Secret key mismatch: deployment referenced database-url instead of DATABASE_URL. Corrected.
5. Port mismatch: application listened on 8080 while Kubernetes resources initially targeted 3000. Corrected.
6. Database connection capacity: initial pool size of 5 produced connection acquisition timeouts during load testing. Increased pool to 8.
7. Distributed idempotency: replaced process-local Map with PostgreSQL unique-constraint handling. Sequential and concurrent tests passed.

## 4. Load test evidence

Tests were executed against a local Kind cluster for 30 seconds using approximately 8 KB payloads and 5% replay traffic.

Initial 150 RPS test, pool size 5:

total: sent=4494 status[201:4188 409:238 500:48] errors[ECONNRESET:20] p50=95ms p95=2489ms p99=3128ms

150 RPS after pool adjustment to 8:

total: sent=4494 status[201:4254 409:240] errors[-] p50=9ms p95=387ms p99=730ms

400 RPS after pool adjustment to 8:

total: sent=11992 status[201:11437 409:555] errors[-] p50=12ms p95=455ms p99=859ms

HTTP 409 responses represent expected duplicate/replay requests.

These short local tests do not establish sustained production capacity.

## 5. Requirements discrepancy

The README specifies expected launch traffic of approximately 400 RPS.

docs/HANDOFF.md specifies approximately 150 RPS.

Both targets were tested locally. The discrepancy should be clarified with the assessment owner.

## 6. Outstanding production risks

- PostgreSQL uses EmptyDir storage; database data is not durable across pod replacement.
- HPA metrics were unavailable in Kind; horizontal scaling was not verified.
- Database migrations execute in every application container at startup.
- S3 archival failure after database insertion can leave an event stored without a successful archive. A durable retry/outbox mechanism is not implemented.
- Terraform contains a hardcoded database password.
- Node IAM permissions are excessively broad.
- EKS API access and SSH ingress are open to 0.0.0.0/0.
- ALB target port and worker security-group ingress port do not match.
- ALB has no HTTPS listener or TLS configuration.
- S3 encryption, versioning and public-access-block configuration are not explicitly defined.
- EBS volumes are not encrypted.
- EKS worker nodes are configured in only one subnet/AZ.
- CI/CD uses mutable GitHub Action references and the mutable latest image tag.
- Production deployment workflow applies all Kubernetes manifests, including PostgreSQL.

## 7. Terraform review

Terraform was reviewed only. No infrastructure was applied or modified.

Production deployment is blocked pending security, networking, storage and availability corrections.

## 8. Final recommendation

DO NOT SHIP.

The application demonstrated successful short-duration local load tests at 150 and 400 RPS. However, production security, persistent storage, migration orchestration, archival reliability and infrastructure networking require remediation and verification before release.
