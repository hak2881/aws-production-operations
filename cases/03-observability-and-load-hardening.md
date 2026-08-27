# Observability and Load Hardening

**Scope** · Kubernetes logs, Redis, PostgreSQL clients, queue workers, and memory-heavy endpoints  
**Owned** · post-load-test changes, log shipping, resource alignment, and operational evidence

## Context

A system that works in normal development traffic can still fail in production because several individually reasonable defaults interact: a Redis eviction policy with an unrealistically small memory limit, database sessions indistinguishable from each other, container logs that disappear with the pod, or an export that materializes the whole result in memory.

Load tests and production incidents were used as design input. The goal was not a generic observability platform; it was to make the next operational question answerable from retained evidence.

## Problem

**Logs without workload identity are hard to use.** A line needs environment, workload, and request context before it can answer which service failed and whether a retry was safe.

**Resource configuration can contradict itself.** Raising an application's cache allowance above the pod limit guarantees termination under the very load the cache is meant to absorb.

**Connection pools hide ownership.** When several services share a database, unidentified sessions make it hard to distinguish application pressure from a migration, worker, or scheduled reconciliation.

**Memory failures often come from response shape.** Increasing a limit can postpone an out-of-memory failure while leaving the unbounded allocation intact.

## Approach

### 1. Retain container logs centrally

Fluent Bit ships Kubernetes container output to CloudWatch with workload and environment dimensions. This created one place to follow an event across restarted pods and removed the assumption that the original container would still exist when an issue was investigated.

### 2. Align cache behavior with the container

After load testing, Redis memory policy and pod resources were changed together. The cache received an explicit eviction policy and a limit consistent with the container boundary. Resource requests and limits were treated as part of application behavior, not deployment decoration.

### 3. Identify database clients

Python database engines set an application name in their connection arguments. Database views could then separate API, worker, and scheduled-job sessions without relying on source IP or guesswork.

### 4. Remove unbounded allocations

Large exports moved to streaming rather than building an entire payload in memory. This addressed the allocation shape instead of only increasing the pod limit.

### 5. Keep secrets out of the evidence trail

Tracked Kubernetes secret values were removed and replaced with value-free examples. Operational documentation listed required variables while real values stayed in the deployment environment.

## Outcome

- post-incident investigation no longer depended on a surviving pod
- cache and container memory settings expressed one coherent limit
- database activity could be attributed to a workload
- large exports stopped competing with the whole process heap
- deployment requirements stayed discoverable without publishing credentials

## What I would revisit

Centralized logs were the correct first move because they answered immediate questions. As the service count grows, the next step is consistent request IDs and traces across queue boundaries; adding dashboards before the underlying identity is consistent would only visualize ambiguity.

---

## 한국어 요약

부하 테스트와 운영 장애 이후의 변경을 모았습니다. Fluent Bit로 컨테이너 로그를 CloudWatch에 보존하고, Redis 정책과 파드 메모리 한도를 함께 조정했으며, DB 연결에 workload identity를 넣고, 대용량 export는 메모리에 전부 올리지 않고 스트리밍하도록 바꿨습니다.

관측 도구를 많이 붙이는 것이 목적은 아니었습니다. 다음 장애 때 “어느 workload가, 어떤 요청에서, 어떤 리소스를 사용하다 실패했는가”를 남아 있는 증거로 답할 수 있게 만드는 것이 목적이었습니다.

