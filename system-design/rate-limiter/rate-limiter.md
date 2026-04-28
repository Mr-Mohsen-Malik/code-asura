# Rate Limiter Algorithms: Complete Interview & System Design Reference

> A clean reference for understanding, teaching, and preparing interview answers around rate limiter algorithms.

---

## Table of Contents

1. [Quick Comparison](#quick-comparison)
2. [Fixed Window Counter](#1-fixed-window-counter)
3. [Sliding Window Log](#2-sliding-window-log)
4. [Sliding Window Counter](#3-sliding-window-counter)
5. [Token Bucket](#4-token-bucket)
6. [Leaky Bucket](#5-leaky-bucket)
7. [Final Recommendation](#final-recommendation)

---

# Quick Comparison

| Algorithm              | Best For                                  | Main Strength                  | Main Weakness                      |
| ---------------------- | ----------------------------------------- | ------------------------------ | ---------------------------------- |
| Fixed Window Counter   | Simple quotas                             | Very simple and fast           | Boundary burst problem             |
| Sliding Window Log     | Login, OTP, fraud-sensitive actions       | Most accurate and fair         | High memory usage                  |
| Sliding Window Counter | Large-scale API rate limits               | Balance of fairness and memory | Approximate, not exact             |
| Token Bucket           | Public APIs and burst-friendly systems    | Allows controlled bursts       | Not exact for rolling-window rules |
| Leaky Bucket           | Downstream protection and traffic shaping | Smooths traffic                | Can add delay or drop bursts       |

---

# 1. Fixed Window Counter

## What It Means

Fixed Window Counter divides time into fixed windows and counts requests inside the current window.

Example:

```text
Limit: 100 requests per minute
Window 1: 12:00:00 - 12:00:59
Window 2: 12:01:00 - 12:01:59
```

If the user sends more than 100 requests in the current window, extra requests are rejected.

---

## When To Use

Use Fixed Window Counter when you need a very simple and low-cost rate limiter.

Good use cases:

* Simple API quotas
* Free-plan usage limits
* Hourly or daily request limits
* Basic internal service protection
* First solution in coding interviews
* Low-traffic systems

Example rules:

```text
100 requests per minute per user
1000 requests per hour per API key
5000 requests per day per account
```

---

## Pros

* Very easy to implement
* Very fast: `O(1)` per request
* Low memory usage
* Easy to explain in interviews
* Works well for simple quotas
* Easy to implement with Redis `INCR` and `EXPIRE`

---

## Cons

* Has boundary burst problem
* Not fair near window reset
* Can allow almost double traffic around a boundary
* Not smooth
* Not ideal for strict abuse prevention

Boundary burst example:

```text
Limit = 100 requests per minute

User sends:
100 requests at 12:00:59
100 requests at 12:01:00

Result:
200 requests in around 2 seconds
```

---

## Companies / Products / Features

> Note: many companies publicly document rate limits but do not always disclose the exact internal algorithm. These are examples of fixed-window-style limits or products that support this style.

| Company / Product            | Feature / Usage                                                             |
| ---------------------------- | --------------------------------------------------------------------------- |
| GitHub REST API              | Hourly API request quota with reset timestamp                               |
| X / Twitter API              | Per-endpoint limits using fixed time windows such as 15 minutes or 24 hours |
| Azure API Management         | `rate-limit` and `rate-limit-by-key` policies with renewal periods          |
| Google Cloud API Gateway     | Request quotas such as per-100-seconds limits                               |
| Cloudflare WAF Rate Limiting | Counts matching requests over a configured time period                      |

---

## What It Resolves Better Than Other Algorithms

Fixed Window Counter solves the basic quota problem with minimum complexity.

Compared with other algorithms:

| Compared To            | Fixed Window Advantage          |
| ---------------------- | ------------------------------- |
| Sliding Window Log     | Uses much less memory           |
| Sliding Window Counter | Simpler to implement            |
| Token Bucket           | Easier to explain and configure |
| Leaky Bucket           | No queue or leak process needed |

Best one-line explanation:

```text
Use Fixed Window when simplicity matters more than perfect fairness.
```

---

## 10 Interview Questions

1. Implement Fixed Window Counter.
2. What is the boundary burst problem?
3. Why can Fixed Window allow almost 2x traffic near reset?
4. What data structure would you use?
5. What is the time complexity?
6. What is the space complexity?
7. How would you implement it in Redis?
8. How do you expire old counters?
9. Is Fixed Window fair?
10. When should you avoid Fixed Window Counter?

---

## Simple Pseudocode

```text
CLASS FixedWindowLimiter
    limit
    windowSize
    map userToWindowStart
    map userToCount

    FUNCTION allow(userId, currentTime)
        windowStart = floor(currentTime / windowSize) * windowSize

        IF userId not in userToWindowStart
            userToWindowStart[userId] = windowStart
            userToCount[userId] = 0

        IF userToWindowStart[userId] != windowStart
            userToWindowStart[userId] = windowStart
            userToCount[userId] = 0

        IF userToCount[userId] < limit
            userToCount[userId] = userToCount[userId] + 1
            RETURN true

        RETURN false
```

---

# 2. Sliding Window Log

## What It Means

Sliding Window Log stores the timestamp of every request. For every new request, it removes expired timestamps and checks how many requests happened in the last time window.

Example:

```text
Limit: 5 requests in the last 10 seconds
Current time: 15s
Valid window: 5s to 15s
Stored timestamps: [7, 9, 11, 13, 14]
```

If another request arrives at `15s`, the count becomes 6 and may be rejected.

---

## When To Use

Use Sliding Window Log when accuracy matters more than memory efficiency.

Good use cases:

* Login attempts
* OTP requests
* Password reset requests
* Payment attempts
* Fraud-sensitive actions
* Security-sensitive endpoints
* Small-scale strict rate limits

Example rules:

```text
5 login attempts in the last 10 minutes
3 OTP requests in the last 5 minutes
10 payment attempts in the last 1 hour
```

---

## Pros

* Very accurate
* Very fair
* No boundary burst problem
* Easy to reason about
* Great for security-sensitive endpoints
* Gives exact rolling-window behavior

---

## Cons

* Stores every request timestamp
* High memory usage for high traffic
* Cleanup is required
* Can be expensive for millions of users
* More complex than Fixed Window

---

## Companies / Products / Features

> Exact algorithm disclosure is not common, but these products or engineering examples publicly support or demonstrate Sliding Window Log style implementations.

| Company / Product              | Feature / Usage                                                          |
| ------------------------------ | ------------------------------------------------------------------------ |
| Tyk API Gateway                | Redis rate limiter using Sliding Window Log style behavior               |
| Redis                          | Official tutorials demonstrate Sliding Window Log rate limiting          |
| Redis Sorted Sets              | Commonly used for timestamp-based sliding-window limiters                |
| ASP.NET Core + Redis tutorials | Demonstrate precise sliding-window rate limiting using Redis sorted sets |
| ClassDojo Engineering          | Published rolling rate limiter approach using request timestamps         |

---

## What It Resolves Better Than Other Algorithms

Sliding Window Log solves Fixed Window's boundary burst problem by using a true rolling window.

Compared with other algorithms:

| Compared To            | Sliding Window Log Advantage                              |
| ---------------------- | --------------------------------------------------------- |
| Fixed Window           | More accurate and fair                                    |
| Sliding Window Counter | Exact, not approximate                                    |
| Token Bucket           | Better for strict `N requests in last T seconds` rules    |
| Leaky Bucket           | Better for allow/reject decisions based on recent history |

Best one-line explanation:

```text
Use Sliding Window Log when you need the most accurate rate limiter.
```

---

## 10 Interview Questions

1. Implement Sliding Window Log.
2. Why is it more accurate than Fixed Window?
3. What data structure would you use?
4. Why is a queue or deque useful here?
5. How do you remove expired timestamps?
6. What is the memory problem?
7. How would you implement it in Redis?
8. Why are Redis sorted sets useful?
9. How do you make the check atomic?
10. When would Sliding Window Log be too expensive?

---

## Simple Pseudocode

```text
CLASS SlidingWindowLogLimiter
    limit
    windowSize
    map userToTimestamps

    FUNCTION allow(userId, currentTime)
        IF userId not in userToTimestamps
            userToTimestamps[userId] = empty queue

        timestamps = userToTimestamps[userId]

        WHILE timestamps not empty AND timestamps.front <= currentTime - windowSize
            timestamps.removeFront()

        IF timestamps.size < limit
            timestamps.addBack(currentTime)
            RETURN true

        RETURN false
```

---

# 3. Sliding Window Counter

## What It Means

Sliding Window Counter is an approximate version of Sliding Window Log. It uses the current window count and part of the previous window count to estimate the request count in the rolling window.

Example:

```text
Limit = 100 requests per minute
Previous window count = 80
Current window count = 30
Current window is 75% complete
Remaining overlap from previous window = 25%

Estimated count = 80 * 0.25 + 30
Estimated count = 20 + 30 = 50
```

---

## When To Use

Use Sliding Window Counter when you need a balance between fairness and memory efficiency.

Good use cases:

* Public API rate limits
* Per-user API limits
* Distributed rate limiters
* Large-scale systems
* SaaS product limits
* When Sliding Window Log is too memory-heavy

---

## Pros

* More fair than Fixed Window
* Lower memory than Sliding Window Log
* Good for distributed systems
* Good practical interview answer
* Reduces boundary burst problem
* Works well for large-scale request limiting

---

## Cons

* Approximate, not exact
* More complex than Fixed Window
* Needs careful time calculation
* Can allow small errors
* Harder to explain than Fixed Window

---

## Companies / Products / Features

> These are products that publicly support sliding-window-style rate limiting or rolling-window behavior.

| Company / Product           | Feature / Usage                                                    |
| --------------------------- | ------------------------------------------------------------------ |
| Kong Rate Limiting Advanced | Sliding window rate limiting for API gateway traffic               |
| Kong Gateway                | Rolling-window examples such as 100 requests per rolling hour      |
| Microsoft ASP.NET Core      | Built-in middleware supports Sliding Window rate limiting          |
| Upstash Ratelimit           | Supports sliding window algorithm for Redis/serverless apps        |
| Arcjet                      | Supports fixed window, sliding window, and token bucket algorithms |

---

## What It Resolves Better Than Other Algorithms

Sliding Window Counter reduces Fixed Window's unfair boundary spike while avoiding the high memory cost of Sliding Window Log.

Compared with other algorithms:

| Compared To        | Sliding Window Counter Advantage                        |
| ------------------ | ------------------------------------------------------- |
| Fixed Window       | Fairer and smoother                                     |
| Sliding Window Log | Much lower memory usage                                 |
| Token Bucket       | Better for `N requests per rolling window` requirements |
| Leaky Bucket       | Does not queue requests; simply allow/reject            |

Best one-line explanation:

```text
Use Sliding Window Counter when you want fairness close to Sliding Window Log but with much less memory.
```

---

## 10 Interview Questions

1. Implement Sliding Window Counter.
2. How is it different from Sliding Window Log?
3. Why is it approximate?
4. How do you calculate weighted previous count?
5. What problem does it solve in Fixed Window?
6. What state do you store per user?
7. How much memory does it use?
8. How would you store it in Redis?
9. What happens at window rollover?
10. When would you choose Sliding Window Log instead of Sliding Window Counter?

---

## Simple Pseudocode

```text
CLASS SlidingWindowCounterLimiter
    limit
    windowSize
    map userToCurrentWindow
    map userToCurrentCount
    map userToPreviousCount

    FUNCTION allow(userId, currentTime)
        currentWindow = floor(currentTime / windowSize) * windowSize

        IF userId not exists
            userToCurrentWindow[userId] = currentWindow
            userToCurrentCount[userId] = 0
            userToPreviousCount[userId] = 0

        IF userToCurrentWindow[userId] != currentWindow
            userToPreviousCount[userId] = userToCurrentCount[userId]
            userToCurrentCount[userId] = 0
            userToCurrentWindow[userId] = currentWindow

        elapsed = currentTime - currentWindow
        remainingRatio = (windowSize - elapsed) / windowSize

        estimatedCount =
            userToPreviousCount[userId] * remainingRatio
            + userToCurrentCount[userId]

        IF estimatedCount < limit
            userToCurrentCount[userId] = userToCurrentCount[userId] + 1
            RETURN true

        RETURN false
```

---

# 4. Token Bucket

## What It Means

Token Bucket uses a bucket that holds tokens. Tokens are refilled over time. Every request consumes one or more tokens.

Example:

```text
Bucket capacity = 100 tokens
Refill rate = 10 tokens per second
Each request costs = 1 token
```

If a token is available, the request is allowed. If no token is available, the request is rejected or delayed.

---

## When To Use

Use Token Bucket when you want to allow short bursts while controlling long-term average traffic.

Good use cases:

* Public APIs
* API Gateway throttling
* Paid API plans
* Burst-friendly user traffic
* Service-to-service throttling
* AI API usage control
* Cloud infrastructure limits

---

## Pros

* Allows controlled bursts
* Controls average request rate
* Very popular in production APIs
* Memory efficient
* Fast: `O(1)` per request
* Good for distributed implementation
* Better user experience than strict fixed limits

---

## Cons

* Not exact for `N requests in last T seconds`
* Requires token refill calculation
* Needs per-user bucket state
* Slightly harder than Fixed Window
* Can be misconfigured if capacity is too high

---

## Companies / Products / Features

| Company / Product      | Feature / Usage                                              |
| ---------------------- | ------------------------------------------------------------ |
| AWS API Gateway        | Throttling using token bucket-style rate and burst limits    |
| AWS Step Functions     | State transition throttling using token bucket scheme        |
| Envoy Proxy            | Local HTTP rate limiting with token bucket configuration     |
| Istio                  | Local rate limiting through Envoy token bucket configuration |
| Microsoft ASP.NET Core | Built-in middleware supports Token Bucket rate limiting      |

Other products such as Arcjet, Upstash, and Tyk also document Token Bucket support.

---

## What It Resolves Better Than Other Algorithms

Token Bucket solves the natural burst problem. Real users often send traffic in small bursts, and Token Bucket allows that without losing long-term control.

Compared with other algorithms:

| Compared To            | Token Bucket Advantage                           |
| ---------------------- | ------------------------------------------------ |
| Fixed Window           | Avoids hard reset boundaries                     |
| Sliding Window Log     | Uses much less memory                            |
| Sliding Window Counter | Better for burst-friendly traffic                |
| Leaky Bucket           | Allows bursts instead of forcing constant output |

Best one-line explanation:

```text
Use Token Bucket when you want to allow bursts but still control the average rate.
```

---

## 10 Interview Questions

1. Implement Token Bucket.
2. What are capacity and refill rate?
3. Why does Token Bucket allow bursts?
4. How is it different from Leaky Bucket?
5. What state is stored per user?
6. What happens when the bucket is empty?
7. How do you calculate new tokens?
8. How do you implement Token Bucket in Redis?
9. How do you make token refill atomic?
10. Why is Token Bucket popular for API gateways?

---

## Simple Pseudocode

```text
CLASS TokenBucketLimiter
    capacity
    refillRatePerSecond
    map userToTokens
    map userToLastRefillTime

    FUNCTION allow(userId, currentTime)
        IF userId not exists
            userToTokens[userId] = capacity
            userToLastRefillTime[userId] = currentTime

        tokens = userToTokens[userId]
        lastTime = userToLastRefillTime[userId]

        elapsed = currentTime - lastTime
        newTokens = elapsed * refillRatePerSecond

        tokens = minimum(capacity, tokens + newTokens)

        userToTokens[userId] = tokens
        userToLastRefillTime[userId] = currentTime

        IF userToTokens[userId] >= 1
            userToTokens[userId] = userToTokens[userId] - 1
            RETURN true

        RETURN false
```

---

# 5. Leaky Bucket

## What It Means

Leaky Bucket accepts incoming requests into a queue or bucket and processes them at a constant rate.

Example:

```text
Bucket capacity = 100 queued requests
Leak rate = 10 requests per second
```

Incoming traffic may be bursty, but outgoing traffic is smooth.

---

## When To Use

Use Leaky Bucket when your main goal is to protect a downstream system by smoothing traffic.

Good use cases:

* Protecting databases
* Protecting payment services
* Protecting third-party APIs
* Message processing
* Queue-based throttling
* Traffic shaping
* Background job processing

---

## Pros

* Smooths traffic
* Protects downstream systems
* Good for queues
* Good for constant processing rate
* Easy mental model
* Reduces traffic spikes

---

## Cons

* Can add latency
* Queue can overflow
* Bursts may be dropped
* Less flexible than Token Bucket
* Not ideal when users should be allowed short bursts
* Requires queue/bucket management

---

## Companies / Products / Features

| Company / Product             | Feature / Usage                                                    |
| ----------------------------- | ------------------------------------------------------------------ |
| Shopify Admin API             | Publicly explains API rate limits using Leaky Bucket behavior      |
| NGINX                         | Rate limiting based on Leaky Bucket style traffic control          |
| Tyk API Gateway               | Documents Leaky Bucket-style limiter behavior                      |
| F5 / NGINX-based gateways     | Commonly use NGINX rate limiting for ingress/API protection        |
| API traffic shapers generally | Used when fixed-rate output is more important than burst allowance |

---

## What It Resolves Better Than Other Algorithms

Leaky Bucket solves the bursty downstream traffic problem. It protects systems that cannot handle sudden spikes.

Compared with other algorithms:

| Compared To            | Leaky Bucket Advantage                  |
| ---------------------- | --------------------------------------- |
| Fixed Window           | No sudden reset spike                   |
| Sliding Window Log     | Does not need to store every timestamp  |
| Sliding Window Counter | Better for controlling output flow      |
| Token Bucket           | Smooths bursts instead of allowing them |

Best one-line explanation:

```text
Use Leaky Bucket when you want smooth outgoing traffic at a fixed rate.
```

---

## 10 Interview Questions

1. Implement Leaky Bucket.
2. How is Leaky Bucket different from Token Bucket?
3. What does leak rate mean?
4. What happens when the bucket is full?
5. Does Leaky Bucket reject or delay requests?
6. Why does it smooth traffic?
7. Where would you use it in system design?
8. How does it protect downstream services?
9. What are its latency trade-offs?
10. When is Token Bucket better than Leaky Bucket?

---

## Simple Pseudocode

```text
CLASS LeakyBucketLimiter
    bucketCapacity
    leakRatePerSecond
    map userToQueueSize
    map userToLastLeakTime

    FUNCTION allow(userId, currentTime)
        IF userId not exists
            userToQueueSize[userId] = 0
            userToLastLeakTime[userId] = currentTime

        elapsed = currentTime - userToLastLeakTime[userId]
        leaked = elapsed * leakRatePerSecond

        userToQueueSize[userId] =
            maximum(0, userToQueueSize[userId] - leaked)

        userToLastLeakTime[userId] = currentTime

        IF userToQueueSize[userId] < bucketCapacity
            userToQueueSize[userId] = userToQueueSize[userId] + 1
            RETURN true

        RETURN false
```

---

# Final Recommendation

## For Coding Interviews

Start with simple and clear algorithms:

```text
Beginner-friendly answer:
    Fixed Window Counter

Better accurate answer:
    Sliding Window Log

Senior-level answer:
    Token Bucket or Sliding Window Counter
```

---

## For System Design Interviews

Recommended choices:

| Scenario                            | Recommended Algorithm                        |
| ----------------------------------- | -------------------------------------------- |
| Public API rate limiting            | Token Bucket                                 |
| Login attempt protection            | Sliding Window Log                           |
| OTP / password reset                | Sliding Window Log or Sliding Window Counter |
| Large-scale API fairness            | Sliding Window Counter                       |
| Simple quota system                 | Fixed Window Counter                         |
| Protect downstream service          | Leaky Bucket                                 |
| API gateway rate + burst limits     | Token Bucket                                 |
| Memory-sensitive distributed system | Sliding Window Counter or Token Bucket       |

---

## Best Interview Answer Template

```text
I would first clarify what we are limiting:
    user, IP, API key, tenant, endpoint, or service.

Then I would choose the algorithm:
    Token Bucket for burst-friendly API traffic.
    Sliding Window Log for strict login/security limits.
    Sliding Window Counter for large-scale fairness.
    Leaky Bucket for downstream traffic smoothing.
    Fixed Window for simple quotas.

For distributed systems:
    I would store rate-limit state in Redis.
    I would make check-and-update atomic using Lua script or atomic operations.
    I would set TTLs to clean inactive keys.
    If the limit is exceeded, I would return HTTP 429 Too Many Requests.
```

---

## One-Line Summary

```text
Fixed Window = simplest
Sliding Window Log = most accurate
Sliding Window Counter = balanced
Token Bucket = best for burst-friendly APIs
Leaky Bucket = best for smoothing traffic
```

---

# Appendix: Generic Rate Limiter Middleware

```text
FUNCTION handleRequest(request)
    identifier = getIdentifier(request)
    endpoint = request.endpoint
    currentTime = getCurrentServerTime()

    rule = getRateLimitRule(identifier, endpoint)
    limiter = getLimiterForRule(rule)

    allowed = limiter.allow(identifier, currentTime)

    IF allowed
        forward request to service
    ELSE
        return response:
            statusCode = 429
            message = "Too many requests"
            retryAfter = calculateRetryAfter()
```

---

# Appendix: Redis Fixed Window Pseudocode

```text
FUNCTION allow(userId, currentTime)
    limit = 100
    windowSize = 60

    windowId = floor(currentTime / windowSize)
    key = "rate_limit:" + userId + ":" + windowId

    count = REDIS.INCREMENT(key)

    IF count == 1
        REDIS.EXPIRE(key, windowSize)

    IF count <= limit
        RETURN true

    RETURN false
```

---

# Appendix: Atomic Redis Token Bucket Pseudocode

```text
FUNCTION allowAtomic(userId, currentTime)
    RUN_AS_ATOMIC_OPERATION:

        bucket = GET_BUCKET(userId)

        IF bucket does not exist
            tokens = capacity
            lastRefillTime = currentTime
        ELSE
            tokens = bucket.tokens
            lastRefillTime = bucket.lastRefillTime

        elapsed = currentTime - lastRefillTime
        newTokens = elapsed * refillRate

        tokens = minimum(capacity, tokens + newTokens)

        IF tokens >= 1
            tokens = tokens - 1
            allowed = true
        ELSE
            allowed = false

        SAVE_BUCKET(userId, tokens, currentTime)

        RETURN allowed
```
