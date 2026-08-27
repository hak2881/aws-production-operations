# Operating Services on EKS

**Scope** · Go and Python APIs, workers, scheduled jobs, and admin applications  
**Owned** · container delivery, Kubernetes manifests, ingress, cluster migration, and rollout documentation

## Context

Several production backends had grown independently. Some began as a single API, others as small services around membership, rewards, orders, or external integrations. They ran in more than one AWS environment and were deployed by different generations of scripts.

The services themselves were not the hardest part of moving them. The risk sat in the state surrounding deployment: whichever AWS identity happened to be active, whichever Kubernetes context a developer last used, an image tag copied from another service, and a target cluster mentioned only in a chat thread.

## Problem

**A successful command can still deploy to the wrong place.** `docker push` and `kubectl apply` returning zero says nothing about whether the selected account, registry, context, and namespace were intended.

**Manifests drift when the delivery path is not one contract.** Image repository, service account, probes, resource requests, ingress annotations, and scheduled jobs need to move together. Updating only the deployment manifest leaves a configuration that looks valid file by file and fails as a system.

**Small services accumulate disproportionate operational variance.** Every one still needs build architecture, health checks, secret injection, rollback, and logs even when the application is only a few endpoints.

## Approach

### 1. Make the target observable before mutation

Deployment scripts print and validate the active AWS identity, registry, Kubernetes context, namespace, service name, and image tag before pushing or applying anything. The target is also documented in the script header so review can catch a mismatch without executing it.

This turns a dangerous hidden precondition into reviewable input. A developer can still choose the wrong target, but the script no longer quietly chooses one for them.

### 2. Treat the Kubernetes surface as one delivery unit

For each workload, the tracked deployment surface includes:

- namespace and service account
- ConfigMap plus a value-free secret example
- Deployment or CronJob
- Service and ALB ingress where required
- readiness and liveness probes
- image build and rollout script

Real secret values stay outside Git. The repository records the required keys, not their values.

### 3. Separate application boundaries from pod boundaries

One backend retained multiple packages for member, order, ledger, payout, and administration responsibilities while consolidating deployment where traffic did not justify a pod and database pool per package. Other workloads remained separate because their triggers and retry behavior differed.

The rule was operational: split deployments when they need different scaling, isolation, or failure handling—not merely because the code has modules.

### 4. Migrate with both routes written down

Cluster moves changed registry targets, workload identity, ingress, image architecture, and rollout commands as one reviewed change. Cutover and rollback artifacts were prepared together, and the old runtime was not treated as disposable until the new path had been observed under production traffic.

## Outcome

- deployment targets became visible in code review and command output
- several services moved between EKS environments without embedding account or cluster identifiers in handover documents
- Go and Python workloads shared a repeatable delivery shape while keeping service-specific scaling and CronJobs
- operational ownership became transferable because build, deploy, secret requirements, and rollback lived with the service

## What I would revisit

Shell scripts were the smallest useful step for teams already operating with `kubectl`, but the next level is policy enforcement: CI checks that reject an unapproved registry, namespace, or workload identity before a human can run the deployment at all.

---

## 한국어 요약

EKS 배포에서 가장 위험한 것은 명령 실패보다 **성공한 명령이 잘못된 계정이나 클러스터에 적용되는 것**이었습니다. 그래서 배포 전에 AWS identity, registry, Kubernetes context, namespace, image tag를 출력하고 검증하도록 만들고, 대상 환경을 스크립트와 문서에 함께 남겼습니다.

서비스 경계와 파드 경계도 분리해서 판단했습니다. 코드상 책임은 나누되 트래픽이 작은 서비스는 배포를 합쳤고, 장애·재시도·스케일링 방식이 다른 워커와 CronJob은 따로 운영했습니다.

