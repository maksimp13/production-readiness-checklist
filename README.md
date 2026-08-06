# Production Readiness Checklist

> Canonical version: **https://ohmyops.dev/checklist/**

A practical readiness checklist for services running on Kubernetes and cloud infrastructure generally.

73 checks, assembled from what actually breaks — not from vendor documentation.

## How to read it

Every item carries one of two marks:

| | Meaning |
|---|---|
| 🤖 | An automated scanner catches this — `popeye`, `polaris`, `kube-score`, `trivy`, `goldilocks`. Run them before you read any further. It is free and takes half an hour. |
| 👤 | No scanner sees this. It needs knowledge of the system, its incident history, or verification by doing. |

**53 of the 73 items are marked 👤.** That is not padding. Tools are good at finding *missing configuration* and bad at finding *wrong configuration*, because "correct" depends on what the system does and what an outage costs.

Here is the difference in one example. A scanner will confirm you have a readiness probe. It will not notice that the probe returns `200 OK` whether or not the database is reachable — and therefore guarantees nothing at all.

## How to use it

1. Run the automated scanners and close the 🤖 items. That is the cheap half.
2. Walk the 👤 items by hand. Record not "configured" but **"verified by doing"**. The gap between those two answers is the content of most postmortems.
3. Do not fix everything. Sort by consequence to the user and take the top five.

The items where your answer is "probably, yes" are the findings. A confident no is safer than an unsure yes: a known gap gets closed, an unknown one gets discovered at three in the morning.

---

## 1. Failure behaviour

- [ ] 🤖 Every workload has liveness, readiness and — where startup is slow — startup probes configured
- [ ] 👤 **Readiness genuinely reflects readiness** — it checks the critical dependencies rather than always returning `200`. The most common false guarantee in a cluster
- [ ] 👤 **Liveness does NOT check external dependencies.** Otherwise an unreachable database restarts every pod at once and turns degradation into a total outage
- [ ] 👤 Slow-starting applications have a startup probe separate from liveness — without it, a cold start becomes a restart loop
- [ ] 🤖 Critical workloads have a PodDisruptionBudget
- [ ] 👤 The PDB does not block drains forever (`minAvailable` equal to the replica count means the node never drains and the cluster never upgrades)
- [ ] 🤖 Stateless services run more than one replica
- [ ] 🤖 `podAntiAffinity` or `topologySpreadConstraints` are set — replicas do not share a node
- [ ] 👤 **The dependency graph has been walked: what exactly breaks when one node is lost, and when a zone is lost.** Not "it should survive" but which services degrade, by name
- [ ] 👤 Single points of failure outside the cluster have been found — a lone database, one NAT gateway, Redis without a replica, a single queue instance
- [ ] 👤 `terminationGracePeriodSeconds` matches how long requests actually take to finish, and the application handles `SIGTERM`
- [ ] 👤 Pods leave the load balancer **before** the process stops (`preStop` hook) — otherwise every deploy drops a share of requests
- [ ] 👤 Timeouts and retries use exponential backoff **and jitter**. Retries without jitter synchronise themselves into a storm
- [ ] 🤖 No `latest` tag and no floating tags in production

## 2. Resources and load

- [ ] 🤖 `requests` and `limits` are set on every container
- [ ] 👤 **`requests` are based on measured consumption, not copied from a neighbouring manifest.** Cross-check against VPA recommendations or `goldilocks`
- [ ] 👤 The decision on CPU limits was made deliberately. CPU throttling under limits is a routine and nearly invisible cause of latency growth
- [ ] 🤖 Every container has a memory limit: one pod's OOM must not take the node with it
- [ ] 👤 Critical services are QoS class Guaranteed or Burstable, never BestEffort — those are evicted first
- [ ] 🤖 HPA is configured wherever load varies
- [ ] 👤 The HPA metric reflects real saturation. CPU lies on I/O-bound workloads
- [ ] 👤 The cluster autoscaler is fast enough: node start plus image pull is shorter than the degradation you can tolerate
- [ ] 👤 **There is headroom for losing a zone.** Three zones at 90% utilisation give you an outage on zone loss, not degradation
- [ ] 👤 Load testing has been done, and its profile resembles production traffic
- [ ] 👤 ResourceQuota and LimitRange are set per namespace — one service cannot eat the cluster

## 3. Data and recovery

- [ ] 🤖 Backups are configured (Velero, disk snapshots, database dumps)
- [ ] 👤 **A backup has been restored at least once.** Not "is it configured" but "have you done it". In most reviews the answer is no
- [ ] 👤 **The actual RPO and RTO are known** — measured in a drill, not written in a document
- [ ] 👤 Backups live outside the same account and region: "the account was deleted" and "ransomware" are not covered by a snapshot sitting next door
- [ ] 👤 Backups are encrypted, and access to them is granted separately from access to production
- [ ] 👤 Backup integrity is verified automatically, not at the moment the backup is needed
- [ ] 🤖 PersistentVolumes holding valuable data do not have `reclaimPolicy: Delete`
- [ ] 👤 For StatefulSets it is clear what happens when a volume is lost, and whether the application replicates at its own level
- [ ] 👤 Database migrations are reversible, or at least compatible with the previous application version — otherwise rollback is impossible
- [ ] 👤 A disaster recovery plan exists and somebody other than its author knows about it

## 4. Security

- [ ] 🤖 `securityContext`: `runAsNonRoot`, `readOnlyRootFilesystem`, unnecessary capabilities dropped
- [ ] 🤖 No privileged containers without an explicit justification
- [ ] 🤖 `hostNetwork`, `hostPID`, `hostPath` — only deliberately and narrowly
- [ ] 🤖 NetworkPolicies exist at all. By default everything in a cluster talks to everything
- [ ] 👤 Policies are built from default-deny rather than as a set of point permissions on top of an open network
- [ ] 🤖 No `cluster-admin` on application service accounts in RBAC
- [ ] 👤 **It is known which humans have production access and how that access is revoked when they leave**
- [ ] 🤖 No secrets in git — the whole history has been checked, not just the current state
- [ ] 👤 Secrets are encrypted in etcd; an external manager is in use (External Secrets, SOPS, Vault)
- [ ] 👤 Secret rotation is technically possible and has been performed at least once
- [ ] 🤖 Images are scanned for vulnerabilities, and there is a process for reacting to a finding rather than only a report
- [ ] 👤 `automountServiceAccountToken` is disabled where the token is not needed
- [ ] 👤 A policy engine is in place (Kyverno, Gatekeeper) — or the rules exist only in people's heads and in review

## 5. Observability

- [ ] 🤖 Metrics are collected from every workload, infrastructure included
- [ ] 👤 **The exact list of alerts that wake a human at night is known.** Without that list there is no on-call, only hope
- [ ] 👤 **The false positive rate has been measured.** An on-call engineer who has learned to ignore alerts is not observability, it is noise with a salary
- [ ] 👤 Alerts fire on symptoms ("users are getting errors") and not only on causes ("CPU is high")
- [ ] 👤 SLOs exist, along with an alert on error budget burn rate
- [ ] 🤖 Logs are centralised, retention is set, search works
- [ ] 👤 One incident can be traced end to end: logs, metrics and traces share a correlation identifier
- [ ] 👤 A dashboard exists that answers "what exactly is broken" within five minutes, and people actually use it
- [ ] 👤 Alerts exist for approaching limits: disk, cloud quotas, API rate limits, **TLS certificate expiry**
- [ ] 👤 Monitoring survives the failure it monitors — it does not live in the same cluster and the same zone

## 6. Releases and rollback

- [ ] 👤 **Rollback has been timed and performed in production at least once.** "We can roll back" without a single precedent is a hypothesis
- [ ] 👤 Production state is reproducible from git; there are no manual `kubectl edit` changes in production
- [ ] 🤖 The update strategy is a deliberate choice: RollingUpdate with a sensible `maxUnavailable`, not Recreate on critical services
- [ ] 👤 Expensive changes go out as a canary or in stages
- [ ] 👤 It is known who can deploy to production and whether review is required
- [ ] 👤 The scenario of a failed database migration mid-rollout has been thought through
- [ ] 👤 Feature flags are available as an alternative to rollback where rollback is expensive

## 7. Cost

- [ ] 👤 The gap between `requests` and actual consumption has been measured — the main source of overspend in Kubernetes
- [ ] 👤 Idle nodes, forgotten load balancers, unattached disks and stale snapshots have been found
- [ ] 👤 Spot or preemptible instances are used where interruption is acceptable
- [ ] 👤 An alert exists for anomalous bill growth — earlier than the email from finance

## 8. Operational maturity

- [ ] 👤 A runbook exists for at least the three most likely failures
- [ ] 👤 **Bus factor:** how many people could rebuild the system from scratch. If the answer is one, that is the largest risk on this list, and it is not a technical one
- [ ] 👤 Postmortems are written and, more importantly, read
- [ ] 👤 There is a separate list of items the team knew about but never got to. That list describes priorities rather than technical debt — and it is more useful than the rest

---

## Questions people ask about this

### What is a production readiness checklist?

A list of checks a service should pass before it carries real user traffic — and, more usefully, before you promise anyone it will stay up. A good one covers failure behaviour, resource limits, data recovery, security, observability, rollback, cost and the human side of operations. A bad one is a list of settings with no reference to what happens when they are wrong.

### How do I know whether my service is production ready?

Go through the checks and mark each one "verified by doing" rather than "configured". Anything you cannot demonstrate — a restore you have performed, a rollback you have timed, an alert you have watched fire — is not ready, regardless of what the manifests say. The count of unsure answers is your real readiness score.

### Which production readiness checks can be automated?

Roughly a quarter. Scanners such as popeye, polaris, kube-score, trivy and goldilocks reliably catch missing probes, absent resource limits, privileged containers, missing NetworkPolicies, `latest` tags and over-broad RBAC. On this list that is 20 items out of 73. Run them first — it is free.

### Why isn't running a scanner enough?

Because a scanner checks whether a setting exists, not whether it is right. It sees a readiness probe and reports success; it cannot see that the probe returns 200 while the database is unreachable. It sees a PodDisruptionBudget and reports success; it cannot see that `minAvailable` equals the replica count, so no node will ever drain. Correctness depends on what the system does and what an outage costs, and neither fact is in the YAML.

### What should a Kubernetes readiness probe actually check?

Whether this instance can serve a request right now — and nothing more. Keep dependency checks out of it. If readiness verifies the database, a thirty-second database blip marks every replica NotReady simultaneously, Kubernetes removes all of them from the Service, and slow queries become connection refused. Put the deep check on a separate endpoint and point monitoring at that.

### Why shouldn't a liveness probe check external dependencies?

Because liveness failure kills the container. A dependency check in a liveness probe means that when the database has trouble, every pod restarts in a loop while the actual problem is somewhere else entirely — and the restarts make recovery harder, not easier. Liveness should answer one question: is this process wedged and unable to recover on its own.

### How often should you test a database restore?

Quarterly is a reasonable floor for most teams, and the test has to produce two numbers: how long the restore took, and whether the data was complete. A backup that has never been restored is a hypothesis. In practice, the first restore drill on a system that has never had one usually fails — which is exactly why it is worth doing before an incident rather than during one.

### What is a safe PodDisruptionBudget setting?

One that still allows voluntary disruption. `minAvailable` equal to the replica count — or `maxUnavailable: 0` — blocks node drains permanently, which means node upgrades, autoscaler scale-down and cluster maintenance all silently stop working. With three replicas, `minAvailable: 2` protects availability and still lets one pod move.

---

## Licence

Use it freely, including inside your company and in commercial work. Corrections and additions are welcome.

## If the list produced a lot of unsure answers

I run this review as a discrete piece of work: two weeks, fixed price, read-only access. The output is a report prioritised by consequence to the user, plus a few ready-made pull requests for the cheapest fixes.

Write to **hello@ohmyops.dev** with your stack and what worries you most. See [ohmyops.dev/services](https://ohmyops.dev/services/) for the other ways I work with teams.
