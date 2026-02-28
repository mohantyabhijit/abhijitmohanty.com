+++
title = "A Practical Guide to Designing Resilient Systems"
date = 2025-12-08T09:00:00+05:30
description = "How to build backend systems that survive failures gracefully. Covers retries, circuit breakers, bulkheads, timeouts, and graceful degradation with real-world examples."
tags = ["architecture", "systems", "backend"]
slug = "designing-resilient-systems"
draft = false
+++

Every system fails. The difference between a good system and a bad one is not whether it fails, but how it fails. A resilient system degrades gracefully. A fragile system collapses entirely.

After spending years building and operating backend services, I have a working list of patterns that consistently make systems more reliable. None of them are new. Most of them are boring. That is the point.

## Start with the Failure Modes

Before writing any code, I ask three questions:

1. What are the external dependencies?
2. What happens when each one is unavailable?
3. What is the acceptable user experience during a failure?

If you cannot answer these before you ship, you are going to answer them at 2AM when something breaks.

## Timeouts: The Most Underrated Defense

A missing timeout is a resource leak waiting to happen. When a downstream service hangs, your threads pile up, your connection pool drains, and eventually your entire service goes down because of someone else's problem.

**Every outbound call needs a timeout.** HTTP requests, database queries, cache lookups, gRPC calls. All of them.

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

resp, err := client.Do(req.WithContext(ctx))
if err != nil {
    // handle timeout or connection failure
    return fallbackResponse(), nil
}
```

I default to aggressive timeouts and loosen them only when I have data showing they need to be longer. A 3-second timeout that occasionally fires is better than a 30-second timeout that lets failures cascade.

## Retries with Backoff

Transient failures are common. A single retry with exponential backoff handles most of them:

```go
func retryWithBackoff(fn func() error, maxAttempts int) error {
    for i := 0; i < maxAttempts; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        if i < maxAttempts-1 {
            delay := time.Duration(math.Pow(2, float64(i))) * 100 * time.Millisecond
            jitter := time.Duration(rand.Int63n(int64(delay / 2)))
            time.Sleep(delay + jitter)
        }
    }
    return fmt.Errorf("all %d attempts failed", maxAttempts)
}
```

The jitter is critical. Without it, all your retries hit the failing service at the same time, making the problem worse. This is called the **thundering herd** problem.

### When Not to Retry

Not all errors are transient. Do not retry:

- **4xx errors** (except 429) -- the request is wrong, retrying won't fix it
- **Validation failures** -- same input will produce the same error
- **Non-idempotent operations** without idempotency keys -- you might create duplicates

## Circuit Breakers

Retries handle transient failures. Circuit breakers handle sustained failures.

The idea is simple: if a dependency has failed N times in the last M seconds, stop calling it entirely for a cooldown period. This prevents your service from wasting resources on a call that is almost certainly going to fail.

A circuit breaker has three states:

- **Closed** (normal): requests flow through
- **Open** (tripped): requests are immediately rejected without calling the dependency
- **Half-open** (testing): a single request is allowed through to test if the dependency has recovered

```
Closed ──(failures exceed threshold)──► Open
  ▲                                       │
  │                                  (cooldown expires)
  │                                       │
  └────(test request succeeds)──── Half-Open
```

In production, I use libraries like `gobreaker` or `resilience4j` rather than building my own. The edge cases around concurrent state transitions are tricky to get right.

## Bulkheads: Isolate Failure Domains

A bulkhead prevents a failure in one part of the system from taking down everything else. The name comes from ship design: compartments that prevent a hull breach from flooding the entire vessel.

In practice, this means:

- **Separate connection pools** per dependency -- if the payment service is slow, it should not exhaust connections needed for the user service
- **Separate thread pools** for critical vs. non-critical work
- **Rate limiting** per tenant in multi-tenant systems

```yaml
# Example: separate connection pool config
databases:
  primary:
    max_connections: 20
    timeout: 5s
  analytics:
    max_connections: 5
    timeout: 10s
  
redis:
  sessions:
    pool_size: 10
  cache:
    pool_size: 15
```

If your analytics database is having a bad day, your primary database connections remain unaffected.

## Graceful Degradation

The best resilience pattern is having a plan for what to show users when things break. Not an error page. Not a spinner that never stops. A degraded but functional experience.

Examples:

- **Search service is down?** Show recent/popular items instead of search results
- **Recommendation engine is slow?** Show a generic "trending" list
- **Payment processor timeout?** Queue the payment and confirm later via email
- **Profile image service is down?** Show initials in a colored circle

Each of these requires thinking about the failure mode in advance and writing the fallback code before the failure happens. It is not glamorous work, but it is the difference between "the site was slow for a minute" and "the site was completely down."

## Observability is Not Optional

None of the patterns above are useful if you cannot see what is happening. At minimum, you need:

1. **Structured logging** with correlation IDs across services
2. **Metrics** on latency percentiles (p50, p95, p99), error rates, and saturation
3. **Alerting** on symptoms (high error rate, elevated latency) not causes

```json
{
  "timestamp": "2025-12-08T09:15:23Z",
  "level": "warn",
  "msg": "circuit breaker opened",
  "service": "payment-gateway",
  "failure_count": 12,
  "cooldown_seconds": 30,
  "correlation_id": "req-abc-123"
}
```

I have seen teams build elaborate retry and circuit breaker logic but skip the logging that tells you when those mechanisms activate. You end up with a system that silently degrades and nobody notices until a customer complains.

## The Testing Problem

Resilience patterns are hard to test because you need to simulate failures. A few approaches that work:

- **Chaos engineering**: deliberately inject failures in staging (kill pods, add latency, drop packets)
- **Fault injection in integration tests**: mock dependencies to return errors or timeouts
- **Load testing with failure scenarios**: run load tests while simultaneously failing a dependency

The simplest starting point is adding a flag to your mocked dependencies that makes them fail a percentage of the time:

```go
type unreliableClient struct {
    real      Client
    failRate  float64
}

func (c *unreliableClient) Get(ctx context.Context, key string) (string, error) {
    if rand.Float64() < c.failRate {
        return "", errors.New("simulated failure")
    }
    return c.real.Get(ctx, key)
}
```

## Summary

Building resilient systems is not about preventing all failures. It is about:

1. Knowing your failure modes before they happen
2. Setting timeouts on everything
3. Retrying with backoff and jitter for transient failures
4. Using circuit breakers for sustained failures
5. Isolating failure domains with bulkheads
6. Having a graceful degradation plan for each dependency
7. Making failures visible with good observability

None of this is exciting. That is by design. Boring, predictable systems are the ones that stay up at 2AM while you stay in bed.
