# Adaptive Rate Limit Scheduler - Architecture

## Overview

The adaptive rate limit scheduler automatically handles provider rate limits during evaluations. It's **zero-configuration** - users don't need to change anything. The scheduler transparently wraps all provider calls with intelligent rate limit detection, retry logic, and adaptive concurrency management.

## Design Goals

The scheduler addresses common challenges when running evaluations against rate-limited APIs:

- **No manual tuning**: Users shouldn't need to guess the right `-j` (concurrency) value
- **Automatic recovery**: Rate limit errors (429) should be retried, not fail permanently
- **Prevent cascading failures**: High concurrency shouldn't cause mass failures
- **Zero configuration**: Works out of the box with sensible defaults

## Architecture

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Evaluator                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     RateLimitRegistry                                │    │
│  │  (Central coordinator - one per evaluation)                          │    │
│  │                                                                      │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │    │
│  │  │ProviderRateLimit │  │ProviderRateLimit │  │ProviderRateLimit │   │    │
│  │  │     State        │  │     State        │  │     State        │   │    │
│  │  │  (openai/key1)   │  │  (openai/key2)   │  │  (anthropic)     │   │    │
│  │  │                  │  │                  │  │                  │   │    │
│  │  │ ┌─────────────┐  │  │ ┌─────────────┐  │  │ ┌─────────────┐  │   │    │
│  │  │ │  SlotQueue  │  │  │ │  SlotQueue  │  │  │ │  SlotQueue  │  │   │    │
│  │  │ │ (FIFO)      │  │  │ │ (FIFO)      │  │  │ │ (FIFO)      │  │   │    │
│  │  │ └─────────────┘  │  │ └─────────────┘  │  │ └─────────────┘  │   │    │
│  │  │ ┌─────────────┐  │  │ ┌─────────────┐  │  │ ┌─────────────┐  │   │    │
│  │  │ │  Adaptive   │  │  │ │  Adaptive   │  │  │ │  Adaptive   │  │   │    │
│  │  │ │ Concurrency │  │  │ │ Concurrency │  │  │ │ Concurrency │  │   │    │
│  │  │ └─────────────┘  │  │ └─────────────┘  │  │ └─────────────┘  │   │    │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

The scheduler uses a hierarchical architecture. At the top level, the Evaluator creates a single RateLimitRegistry that serves as the central coordinator for the entire evaluation run. The registry creates one ProviderRateLimitState for each unique provider-and-API-key combination — this is important because different API keys have independent rate limits even for the same provider. Each state object contains a SlotQueue (a FIFO queue that controls how many concurrent requests are in flight) and an AdaptiveConcurrency controller (which dynamically adjusts the concurrency limit based on success and failure patterns). This per-provider isolation ensures that a rate limit on one provider doesn't slow down requests to other providers.

## Component Responsibilities

### RateLimitRegistry

**File**: `src/scheduler/rateLimitRegistry.ts`

Central coordinator that:

- Creates/retrieves per-provider state based on rate limit keys
- Routes provider calls to the appropriate state
- Aggregates metrics across all providers
- Emits events for monitoring

```typescript
// Usage (automatic in evaluator)
const result = await registry.execute(provider, () => provider.callApi(...), {
  getHeaders: (result) => result.metadata?.headers,
  isRateLimited: (result, error) => error?.message?.includes('429'),
  getRetryAfter: (result, error) => parseRetryAfter(headers['retry-after']),
});
```

### ProviderRateLimitState

**File**: `src/scheduler/providerRateLimitState.ts`

Per-provider state manager that:

- Manages the slot queue for concurrency control
- Tracks rate limit headers from responses
- Implements retry logic with exponential backoff
- Adapts concurrency based on success/failure patterns
- Collects latency metrics

### SlotQueue

**File**: `src/scheduler/slotQueue.ts`

FIFO queue with concurrency limiting:

- Acquires/releases "slots" for concurrent requests
- Blocks when at max concurrency or quota exhausted
- Tracks remaining requests/tokens from headers
- Schedules queue processing after rate limit windows

Key insight: **Race-condition-free** slot allocation. All requests queue, then slots are allocated in FIFO order.

### AdaptiveConcurrency

**File**: `src/scheduler/adaptiveConcurrency.ts`

Dynamic concurrency adjustment:

- **On rate limit**: Reduce concurrency by 50% (multiplicative decrease)
- **On sustained success**: Increase by 1 (additive increase)
- **Proactive throttling**: Reduce when approaching limits (via headers)

This implements AIMD (Additive Increase, Multiplicative Decrease) - the same algorithm TCP uses for congestion control.

### HeaderParser

**File**: `src/scheduler/headerParser.ts`

Parses rate limit headers from multiple providers:

- **OpenAI**: `x-ratelimit-remaining-requests`, `x-ratelimit-limit-requests`
- **Anthropic**: `anthropic-ratelimit-requests-remaining`
- **Generic**: `retry-after`, `retry-after-ms`, `ratelimit-reset`

### RetryPolicy

**File**: `src/scheduler/retryPolicy.ts`

Determines retry behavior:

- Exponential backoff with jitter
- Respects server `retry-after` headers
- Configurable max retries (default: 3)
- Retries on: rate limits, timeouts, 502/503/504

## Data Flow

```text
1. Evaluator calls registry.execute(provider, callFn)
       │
       ▼
2. Registry gets/creates ProviderRateLimitState for this provider
       │
       ▼
3. State.executeWithRetry() is called
       │
       ▼
4. SlotQueue.acquire() - wait for available slot
       │
       ▼
5. Execute callFn() - actual provider API call
       │
       ▼
6. Parse response headers → update rate limit state
       │
       ▼
7. Check if rate limited:
   ├─ Yes → retry with backoff, reduce concurrency
   └─ No  → record success, maybe increase concurrency
       │
       ▼
8. SlotQueue.release() - free slot for next request
       │
       ▼
9. Return result (or throw after max retries)
```

This nine-step flow shows what happens for every single LLM API call. The evaluator asks the registry to execute a provider call. The registry looks up (or creates) the rate limit state for that specific provider and API key. The state's retry logic wraps the actual call. Before the call executes, it must acquire a slot from the queue — if the concurrency limit is reached, the request waits in FIFO order. After the call completes, response headers are parsed for rate limit information (remaining requests, reset times). If a rate limit error (HTTP 429) is detected, the concurrency is reduced by 50% (multiplicative decrease) and the request is retried with exponential backoff. If the call succeeds, the system may gradually increase concurrency by 1 (additive increase). The slot is released for the next waiting request. This AIMD (Additive Increase, Multiplicative Decrease) algorithm — the same algorithm that TCP uses for congestion control — converges quickly to the optimal concurrency level without overshooting.

## Rate Limit Key Generation

Each provider gets a unique "rate limit key" based on:

- Provider ID (e.g., "openai:chat:gpt-4o")
- API key hash (different keys = different rate limits)
- Organization ID (if applicable)

This ensures:

- Same provider + same key = shared rate limit state
- Same provider + different keys = separate rate limits
- Different providers = completely isolated

## Key Design Decisions

### 1. Zero Configuration

Users shouldn't need to tune rate limit settings. The scheduler learns from response headers and adapts automatically.

### 2. Fail-Safe Defaults

- Default max concurrency: 4 (conservative)
- Default retry delay: 60 seconds (when no header)
- Max retries: 3 (prevents infinite loops)

### 3. Proactive Throttling

Don't wait for 429 errors. When headers show <10% remaining quota, proactively reduce concurrency.

### 4. Per-Provider Isolation

Different providers have different rate limits. Don't let OpenAI rate limits affect Anthropic calls.

### 5. Transparent Integration

The scheduler wraps `provider.callApi()` without changing the interface. Existing code works unchanged.

## Metrics

The scheduler tracks:

- `totalRequests` - All requests attempted
- `completedRequests` - Successful completions
- `failedRequests` - Permanent failures (after retries)
- `rateLimitHits` - Times 429 was encountered
- `retriedRequests` - Requests that required retry
- `avgLatencyMs`, `p50LatencyMs`, `p99LatencyMs` - Latency distribution

## Events

For monitoring/debugging, the scheduler emits:

- `slot:acquired` / `slot:released` - Concurrency tracking
- `ratelimit:hit` - Rate limit encountered
- `ratelimit:learned` - First time seeing provider's limits
- `ratelimit:warning` - Approaching rate limit
- `concurrency:increased` / `concurrency:decreased` - Adaptive changes
- `request:retrying` - Retry in progress

## Testing

256 tests covering:

- Unit tests for each component
- Edge cases (negative values, zero values, overflow)
- Race condition prevention
- Integration with evaluator

## Performance Characteristics

- **Overhead**: Minimal - just slot acquisition and header parsing
- **Memory**: O(providers) - one state object per unique rate limit key
- **Latency buffer**: Circular buffer, last 100 requests, O(1) insertion
