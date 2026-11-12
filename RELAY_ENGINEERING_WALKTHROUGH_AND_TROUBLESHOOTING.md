# Relay Assessment — Engineering Investigation, Troubleshooting and Improvement Plan

## 1. Executive summary

Relay is a Node.js/Express event-ingestion API. It accepts event payloads, assigns an event ID, creates an HMAC-SHA256 signature, persists the event in PostgreSQL, and archives the payload in Amazon S3.

The assessment work focused on four areas:

1. **Application correctness:** understand the request lifecycle and make duplicate handling safe across multiple application replicas.
2. **Kubernetes deployment reliability:** correct configuration mismatches that prevented the application from starting or being reached reliably.
3. **Performance validation:** investigate database connection acquisition timeouts and test the application at the stated traffic levels.
4. **Production readiness:** review the container and Terraform configuration, identify security and reliability gaps, and document what still needs to be fixed.

**Recommendation: DO NOT SHIP** in the current state. The application completed short local load tests at 150 and 400 requests per second after the connection-pool adjustment, but important production concerns remain. A successful local test is not equivalent to proving production readiness.

---

## 2. What the application does

### 2.1 Business purpose

Relay receives events from a client application through `POST /events`. An event could represent a transaction or another business activity. The client supplies a JSON payload and an `Idempotency-Key` header.

The application then:

1. Receives the HTTP request through Express.
2. Generates an event ID.
3. Calculates an HMAC-SHA256 signature using the configured signing secret.
4. Attempts to insert the event into PostgreSQL using the idempotency key.
5. If the insert succeeds, archives the payload in Amazon S3.
6. Returns the event ID and signature to the client.

A replay using an idempotency key that already exists returns HTTP `409` and the existing event ID rather than creating another event.

### 2.2 Component responsibilities

| Component | Responsibility |
|---|---|
| Client | Sends an event payload and idempotency key |
| Node.js | Runtime that executes the application |
| Express | HTTP server and request routing |
| PostgreSQL | Stores event records and enforces uniqueness of idempotency keys |
| Amazon S3 | Stores archived event payloads |
| Docker | Packages the application and its runtime dependencies |
| Kubernetes | Runs, schedules, exposes and monitors application pods |
| Kubernetes Service | Provides a stable network endpoint for the application pods |
| HPA | Configures replica scaling based on CPU utilization |
| AWS IAM | Controls what AWS resources and actions workloads can access |

**Key distinction:** PostgreSQL is the event persistence and idempotency layer; S3 is the archive; Kubernetes runs and manages the application. They are not interchangeable.

### 2.3 Request flow

```text
Client
  |
  | POST /events
  | JSON payload + Idempotency-Key
  v
Express API
  |
  | Generate event ID
  | Calculate HMAC-SHA256 signature
  v
PostgreSQL
  |
  | Insert event with unique idempotency key
  |
  +---- Existing key? ----> HTTP 409 + existing event ID
  |
  +---- New event? -------> Continue
                              |
                              v
                         Amazon S3
                         Archive payload
                              |
                              v
                       HTTP 201 response
                       Event ID + signature
```

The `/health` endpoint is used by Kubernetes readiness and liveness probes. In this implementation, the health check also tests database connectivity with a simple query. This means the endpoint checks more than whether the Node.js process is alive.

### 2.4 HMAC is not encryption

HMAC-SHA256 creates a message authentication code using the payload and a shared secret. A system with the same secret can independently calculate and compare the signature to verify integrity and authenticity.

It does **not** encrypt or hide the payload. Encryption would be a separate requirement.

---

## 3. Investigation and changes made

For each issue, the important story is: symptom → evidence → root cause → change → validation.

### 3.1 Application and Kubernetes port mismatch

**Symptom**

The application was required to listen on port `8080`, but the application and Kubernetes configuration were not consistently aligned. A mismatch can cause the Service to forward traffic to the wrong port or cause health probes to fail.

**Investigation**

I compared the application's `PORT` configuration with:

- `containerPort` in the Deployment.
- `targetPort` in the Service.
- The port and path used by readiness and liveness probes.

Useful inspection commands:

```bash
kubectl get deployment relay -n relay -o yaml
kubectl get service relay -n relay -o yaml
kubectl describe pod -n relay -l app=relay
```

**Change made**

Aligned the application and Kubernetes configuration to port `8080`. The health probes use `/health` on port `8080`, and the Service forwards to the correct container port.

**Why it matters**

A Kubernetes Service does not automatically discover the port on which an application listens. The application listener, Service target port and probe configuration must describe the same actual endpoint.

**Further improvement**

Keep the port in one clear configuration contract and validate it during deployment. Add a deployment smoke test that checks the Service endpoint after rollout.

### 3.2 PostgreSQL Secret key mismatch

**Symptom**

The Deployment referenced a Secret key named `database-url`, while the Secret defined `DATABASE_URL`.

**Investigation**

Compared the Secret's key names with the Deployment's `secretKeyRef`. Kubernetes Secret keys are exact and case-sensitive.

Useful commands:

```bash
kubectl get secret relay-secrets -n relay
kubectl describe deployment relay -n relay
kubectl describe pod -n relay -l app=relay
```

Do not print or paste Secret values into terminal output shared publicly.

**Change made**

Updated the Deployment's `secretKeyRef.key` to `DATABASE_URL`.

**Why it matters**

The Secret can exist and still fail to populate an environment variable if the Deployment references the wrong key. This is a configuration/injection issue, not necessarily a database outage.

**Further improvement**

Use an external secret-management solution such as AWS Secrets Manager integrated with Kubernetes. Avoid storing production credentials in plain Kubernetes manifests in Git. Restrict access through Kubernetes RBAC and AWS IAM.

### 3.3 Kind `ImagePullBackOff`

**Symptom**

The application pod entered `ImagePullBackOff`. The image had been built locally, but the Kind node could not access it.

**Investigation**

Kind runs Kubernetes nodes inside containers. A Docker image available to the host is not automatically available inside a Kind node.

Useful commands:

```bash
docker images
kubectl get pods -n relay
kubectl describe pod -n relay -l app=relay
```

**Change made**

Loaded the local image into the Kind cluster:

```bash
kind load docker-image relay:latest --name relay
```

Set the local assessment Deployment to:

```yaml
imagePullPolicy: IfNotPresent
```

**Why it matters**

The image must either be present on the node or be available from a registry. `IfNotPresent` allows the node to use a local image if it already has one.

**Further improvement**

For production, publish versioned images to a private registry such as Amazon ECR. Prefer immutable tags or image digests over `latest`, and promote the same tested image between environments. `IfNotPresent` with a mutable `latest` tag can leave nodes running stale images.

### 3.4 PostgreSQL connection acquisition timeouts under load

**Symptom**

The first 150 RPS test with an application pool size of 5 produced:

- 48 HTTP 500 responses.
- 20 `ECONNRESET` transport errors.
- p95 latency of 2,489 ms.
- p99 latency of 3,128 ms.

The application logs showed database connection acquisition timeout messages. PostgreSQL logs did not show a corresponding database crash or clear server-side failure.

**Investigation**

I correlated the load-test output with the application logs and reviewed the connection-pool configuration in `app/src/db.js` and the `PG_POOL_MAX` setting supplied through Kubernetes configuration.

Useful commands:

```bash
kubectl logs deployment/relay -n relay --since=15m
kubectl get configmap relay-config -n relay -o yaml
kubectl get pods -n relay
```

The pool size controls how many database connections one application process can use. If requests need database connections faster than the pool can provide them, requests can wait and eventually time out.

**Change made**

Increased `PG_POOL_MAX` from 5 to 8 and repeated the 150 RPS and 400 RPS tests. Both subsequent runs completed without reported HTTP 500 responses or transport errors.

**Why it matters**

A pool that is too small can become a bottleneck under concurrent load. However, increasing the pool without considering the database's total connection budget can create a different problem.

The supplied PostgreSQL configuration allowed 100 connections. If the HPA reaches 10 application pods and each pod uses a pool of up to 8 connections:

```text
10 pods × 8 connections = 80 possible application connections
```

That leaves about 20 connections for migrations, administration and other clients, assuming every application pool reaches its maximum. It is a capacity estimate, not proof that all 80 connections were used during the test.

**Further improvement**

- Calculate the connection budget across the maximum replica count and all other database clients.
- Monitor pool usage, waiting requests, connection acquisition latency and PostgreSQL active/idle connections.
- Consider PgBouncer or another pooling/proxy strategy if the production topology requires it.
- Load-test with the production database tier and realistic concurrent traffic.
- Do not increase pool size blindly; tune it using measurements.

### 3.5 Replacing process-local idempotency with PostgreSQL enforcement

**Problem**

The original implementation used an in-memory JavaScript `Map` to remember processed idempotency keys.

That state exists only inside one Node.js process. If Kubernetes runs multiple replicas, each pod has its own memory. A retry could reach a different pod that has never seen the key and create a duplicate.

**Change made**

Moved duplicate enforcement to PostgreSQL. The application uses an insert with:

```sql
INSERT INTO events (id, idempotency_key, payload, signature)
VALUES ($1, $2, $3, $4)
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id;
```

The table's unique constraint is the shared enforcement point. If the insert returns no row, the application looks up the existing event ID and returns HTTP `409`.

I also removed the obsolete in-memory deduplication configuration.

**Why it matters**

The database unique constraint works across application processes and pods. It avoids relying on local memory for a correctness guarantee that must be shared across replicas.

**Validation**

The table definition was inspected with PostgreSQL's `\d events` command, which showed the unique index on `idempotency_key`. The load tests included 5% replay traffic; the resulting HTTP `409` responses were expected duplicate responses.

**Further improvement**

- Keep the unique constraint in a version-controlled database migration.
- Add automated tests for sequential and concurrent duplicate requests.
- Define the idempotency retention period and behaviour if old records are archived or deleted.
- Consider returning a documented response body and status consistently for all duplicate requests.
- Review the insert-then-select behaviour under the chosen transaction isolation and deletion policy.

### 3.6 HPA metrics unavailable in Kind

**Symptom**

The HPA showed unknown CPU utilization, and `kubectl top pods` could not retrieve metrics.

**Investigation commands**

```bash
kubectl get hpa -n relay
kubectl describe hpa -n relay
kubectl top pods -n relay
```

The local Kind cluster did not have a working Metrics API.

**Change made**

Documented the limitation and did not claim that horizontal scaling had been validated.

**Why it matters**

An HPA resource can be correctly defined but still be unable to make scaling decisions if the metrics provider is unavailable. CPU-based HPA also requires CPU requests to be configured on the target containers; those requests were included in the Deployment.

**Further improvement**

Install and validate Metrics Server in the test cluster, then generate load and observe HPA recommendations and replica changes. In production, validate scaling behaviour, pod startup time, readiness, node capacity and database connection limits together.

### 3.7 Docker image hardening

**Problem**

The original Docker build context and runtime privileges could be improved.

**Change made**

- Added `app/.dockerignore` to exclude environment files, dependencies, logs and Git metadata.
- Updated the Dockerfile to install production dependencies.
- Copied application files with appropriate ownership.
- Configured the container to run as the non-root `node` user.
- Set the Node.js heap limit to 160 MiB.

**Build verification**

```bash
docker build -t relay:latest app
```

The image build completed successfully.

**Why it matters**

A smaller build context reduces unnecessary files and the risk of accidentally packaging sensitive local files. Running as a non-root user reduces the privileges available to the application process.

**Important validation boundary**

A successful Docker build proves that the image can be built. It does not, by itself, prove that the rebuilt image was loaded into Kind, rolled out, and successfully exercised there. That runtime verification should be performed after rebuilding.

**Further improvement**

- Pin the base image to a reviewed version or digest.
- Scan the image for vulnerabilities.
- Use a multi-stage build if it reduces the final image and attack surface.
- Consider a read-only root filesystem, dropped Linux capabilities and a restrictive seccomp profile where compatible.
- Verify the application can write only to explicitly mounted writable paths.
- Add image signing and provenance to the CI/CD pipeline.

### 3.8 Git and environment-file hygiene

**Problem**

`app/.env` was tracked by Git.

**Change made**

Removed `app/.env` from the committed project and added environment-file patterns to `.gitignore`. The assessment commit showed the `.env` file deleted.

Useful verification commands:

```bash
git ls-files app/.env
git status --short
git show --stat --oneline HEAD
```

If `git ls-files app/.env` prints the path, the file is still tracked.

**Why it matters**

`.gitignore` prevents untracked files from being added accidentally; it does not untrack a file that Git already tracks. Removing a file in a new commit also does not erase it from earlier Git history.

**Further improvement**

If the file ever contained real credentials, rotate them. Use environment injection or a secret manager. Review repository history and access permissions before sharing the repository publicly.

---

## 4. Load-test evidence and interpretation

The local Kind tests used approximately 8 KB payloads and 5% replay traffic.

### 4.1 Initial test: 150 RPS, pool size 5

```text
total: sent=4494 status[201:4188 409:238 500:48] errors[ECONNRESET:20] p50=95ms p95=2489ms p99=3128ms
```

This test exposed the database connection acquisition timeout issue. The HTTP 409 responses were replay responses; the HTTP 500 and transport errors were the failures.

### 4.2 Retest: 150 RPS, pool size 8

```text
total: sent=4494 status[201:4254 409:240] errors[-] p50=9ms p95=387ms p99=730ms
```

The retest completed without reported HTTP 500 responses or transport errors.

### 4.3 Retest: 400 RPS, pool size 8

```text
total: sent=11992 status[201:11437 409:555] errors[-] p50=12ms p95=455ms p99=859ms
```

The 400 RPS test also completed without reported HTTP 500 responses or transport errors.

### 4.4 What the latency percentiles mean

- **p50:** 50% of measured requests completed at or below this latency.
- **p95:** 95% of measured requests completed at or below this latency.
- **p99:** 99% of measured requests completed at or below this latency.

p95 and p99 help reveal slower requests that an average can hide.

### 4.5 What the tests prove — and what they do not

The results demonstrate that the local Kind deployment completed short tests at 150 and 400 RPS with the adjusted pool size and the stated payload/replay settings.

They do **not** prove sustained production capacity. The tests did not validate the complete production EKS environment, production database tier, real S3 performance, production networking, HPA behaviour or long-duration stability.

The supplied assessment material also contained two traffic expectations: approximately 400 RPS in the README and approximately 150 RPS in `docs/HANDOFF.md`. Both were tested locally, and the discrepancy should be clarified with the assessment owner.

---

## 5. Production risks identified

These are not all changes made in the assessment. They are outstanding items that should be resolved before a production release.

### 5.1 Data durability

The supplied PostgreSQL Kubernetes manifest uses `EmptyDir` for database storage. `EmptyDir` is ephemeral and is not suitable for durable production database storage.

**Recommendation:** use a managed PostgreSQL service or a properly operated StatefulSet with persistent volumes, backups, point-in-time recovery, restore testing and a defined recovery objective. The supplied `k8s/postgres.yaml` was treated as out of scope and was not changed.

### 5.2 S3 archival reliability

The application inserts the event into PostgreSQL before archiving to S3. If S3 fails after the database insert, a retry can be treated as a duplicate and may not retry the archive.

**Recommendation:** implement a transactional outbox or durable archival state machine. Persist the need to archive in the same database transaction as the event, then have a worker retry S3 delivery with backoff, observability and a dead-letter/reconciliation process.

### 5.3 Database migrations

The container entrypoint runs migrations when the application starts. With multiple replicas, multiple pods may attempt to run migrations concurrently.

**Recommendation:** run migrations as a controlled one-off Kubernetes Job or release pipeline step. Use migration locking and backward-compatible expand/contract changes.

### 5.4 Secrets and IAM

The supplied Kubernetes Secret manifest contains credentials, and Terraform includes overly broad permissions and a hardcoded database password.

**Recommendation:** use AWS Secrets Manager or another approved secret manager, least-privilege IAM policies, workload identity/IRSA, credential rotation and restricted access. Never commit production credentials.

### 5.5 Network security and TLS

The Terraform review identified public management access, SSH exposure, missing HTTPS/TLS configuration and an ALB target port versus worker security-group ingress mismatch.

**Recommendation:** restrict EKS API access to approved networks, remove public SSH, use private worker subnets where appropriate, align security-group rules with the actual NodePort traffic path, and configure HTTPS with managed certificates.

### 5.6 Storage security and availability

The Terraform review identified missing explicit S3 encryption/versioning/public-access controls, unencrypted EBS configuration and worker placement in only one subnet/AZ.

**Recommendation:** enable encryption at rest, S3 Block Public Access, versioning and lifecycle policies; distribute worker capacity across availability zones; and test recovery procedures.

### 5.7 CI/CD safety

The supplied workflow used mutable action references, a mutable `latest` image tag, and applied all Kubernetes manifests, including the PostgreSQL manifest.

**Recommendation:** pin GitHub Actions to reviewed versions or commit SHAs, build and scan an immutable image, deploy a specific digest, wait for rollout status, and deploy only the intended application resources. Separate database lifecycle management from application deployment.

### 5.8 Autoscaling and observability

HPA behaviour was not verified in Kind because metrics were unavailable. Short load tests also do not establish long-term stability.

**Recommendation:** validate Metrics Server/HPA in a representative cluster. Monitor request rate, error rate, p50/p95/p99 latency, event insert failures, pool wait time, PostgreSQL saturation, S3 archival failures, pod restarts and replica counts. Define service-level objectives and alert thresholds.

---

## 6. Commands and checks used during the investigation

The following are the key commands associated with the work. Some are commands used during the investigation; others are repeatable diagnostic/verification commands to use when reproducing the checks.

### Docker and Kind

```bash
# Build the application image
docker build -t relay:latest app

# Confirm the image exists locally
docker images

# Load the image into the Kind cluster
kind load docker-image relay:latest --name relay
```

### Kubernetes inspection and troubleshooting

```bash
# Check pods
kubectl get pods -n relay

# Inspect the application Deployment
kubectl get deployment relay -n relay -o yaml

# Inspect the Service
kubectl get service relay -n relay -o yaml

# Describe a pod to inspect events and startup failures
kubectl describe pod -n relay -l app=relay

# Read application logs
kubectl logs deployment/relay -n relay --since=15m

# Inspect the ConfigMap
kubectl get configmap relay-config -n relay -o yaml

# Inspect HPA configuration and status
kubectl get hpa -n relay
kubectl describe hpa -n relay

# Check pod resource metrics (requires Metrics API)
kubectl top pods -n relay
```

### Application health check

```bash
curl -i http://localhost:8080/health
```

Expected outcome: HTTP 200 when the application and its database health check are successful.

### PostgreSQL inspection

From a PostgreSQL client connected to the assessment database:

```sql
-- Inspect the events table, columns, indexes and constraints in psql
\d events

-- Inspect current database connections
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

The table inspection showed a unique index on `idempotency_key`. Connection counts observed after a test are only a snapshot; they do not establish the peak number of connections used during the test.

### Application syntax and Git checks

```bash
# Check JavaScript syntax
node --check app/src/server.js

# Confirm app/.env is no longer tracked
git ls-files app/.env

# Review working tree
git status --short

# Review the assessment commit
git show --stat --oneline HEAD
```

### Load test

The recorded test evidence is in Section 4. The load generator is `tools/load.js`. Before reproducing the tests, inspect its supported options so the exact command matches the checked-in script:

```bash
node tools/load.js --help
```

Use the script's supported options to reproduce the 30-second runs at 150 and 400 RPS, with approximately 8 KB payloads and 5% replay traffic. Preserve the complete output for the assessment record.

---

## 7. Suggested spoken explanation

> Relay is an event-ingestion API built with Node.js and Express. A client sends a JSON event to `POST /events` with an idempotency key. The application generates an event ID, signs the payload with HMAC-SHA256, persists the event in PostgreSQL and then archives the payload in S3.
>
> I began by tracing the application configuration through Kubernetes. I found a port mismatch between the application and Kubernetes resources, and a Secret key mismatch that prevented the database connection string from being injected. I corrected those configurations.
>
> I also encountered an ImagePullBackOff because the local Kind node could not access the image built on my machine. I loaded the image into Kind and used `IfNotPresent` for the local assessment environment.
>
> During load testing, the initial pool size of five produced database connection acquisition timeouts, HTTP 500 responses and connection-reset errors. I correlated the application logs with the test results, increased the pool size to eight, and repeated the tests at 150 and 400 requests per second. Both subsequent runs completed without reported HTTP 500 responses or transport errors.
>
> I replaced process-local idempotency with PostgreSQL unique-constraint enforcement because an in-memory Map is not shared across application replicas. The database now provides the shared enforcement point, and duplicate requests return HTTP 409 with the existing event ID.
>
> I hardened the Docker build context and configured the container to run as a non-root user. I also reviewed the HPA configuration, but I could not validate scaling in Kind because the Metrics API was unavailable.
>
> Finally, I reviewed the Terraform configuration without applying infrastructure. I identified production concerns involving durable database storage, migration execution, S3 retry reliability, IAM permissions, network exposure, TLS, storage encryption and deployment safety.
>
> The local tests demonstrate encouraging application behaviour at the tested rates, but they do not prove production readiness. My recommendation is DO NOT SHIP until the outstanding risks are remediated and validated.

---

## 8. Final assessment position

### Work completed

- Understood and documented the event-ingestion request flow.
- Aligned application and Kubernetes port configuration.
- Corrected the PostgreSQL Secret key reference.
- Resolved local Kind image availability.
- Adjusted the application database pool from 5 to 8 after investigating connection acquisition timeouts.
- Replaced in-memory idempotency with PostgreSQL unique-constraint enforcement.
- Added Docker build-context exclusions and non-root container configuration.
- Ran short local load tests at 150 and 400 RPS.
- Reviewed Terraform without applying infrastructure.
- Documented production risks and the remaining validation gaps.

### Work still required before production

- Durable PostgreSQL storage, backups and restore testing.
- Reliable S3 archival retry/outbox processing.
- Controlled migration execution.
- External secret management and least-privilege IAM.
- Private/restricted management access, correct network rules and HTTPS.
- Storage encryption and multi-AZ availability.
- Verified HPA behaviour and production-like sustained load testing.
- Safer CI/CD with pinned actions, immutable image references and controlled manifest deployment.

**Final recommendation: DO NOT SHIP.**
