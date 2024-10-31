# Title

- RFC PR: https://github.com/tikv/rfcs/pull/115
- Tracking Issue: https://github.com/tikv/pd/issues/8678

## Summary

Implement a [circuit breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) pattern to protect the PD leader from overloading due to retry storms or similar feedback loops.

## Motivation

In large TiDB clusters with hundreds of tidb and tikv nodes, the PD leader can become overwhelmed during certain failure conditions, leading to a "retry storm" or other feedback loop scenarios. 
Once triggered, PD transitions into a metastable state and cannot recover autonomously, leaving the cluster in a degraded or unavailable state.
This feature is especially critical for large clusters where a high number of nodes can continuously hammer a single PD instance during failures, leading to cascading effects that worsen recovery times and overall cluster health.

## Detailed design

### Configuration

The key part of the circuit breaker is its configuration. 
In order to limit number of system variables, we will use a single system variables per circuit breaker type to define error rate threshold. 

#### System variables

`tidb_cb_pd_metadata_error_rate_threshold_pct`:
- Scope: GLOBAL
- Persists to cluster: Yes
- Applies to hint SET_VAR: No
- Type: Integer
- Default value: 0
- Range: [0, 100]
- This variable is used to set percent of errors to trip the circuit breaker.`0` means circuit breaker is disabled.

#### Hardcoded configs

* `error_rate_window`
* Defines how long to track errors before evaluating error_rate_threshold.
* Default: 10
* Unit: seconds
---
* `min_qps_to_open`
* Defines the average qps over the `error_rate_window` that must be met before evaluating the error rate threshold.
* Default: 10
* Unit: integer
---
* `cooldown_interval`
* Defines how long to wait after circuit breaker is open before go to half-open state to send a probe request. This interval always equally jittered in the `[value/2, value]` interval.
* Default: 60
* Unit: seconds
---
* `half_open_success_count`
* Defines how many subsequent requests to test after cooldown period before fully close the circuit. All request in excess of this count will be errored till the circuit is fully closed pending results of the firsts `half_open_success_count` requests. 
* Default: 10
* Unit: integer

### Circuit Breaker API

#### Config

All configs described above will be encapsulated in a `Settings` struct with ability to change error rate threshold dynamically:

```go
type Settings struct {
	Type                  string
	ErrorRateThresholdPct uint32
	ErrorRateWindow       time.Duration
	CoolDownInterval      time.Duration
	HalfOpenSuccessCount  uint32
}

// ChangeErrorRateThresholdPct changes the error rate threshold for the circuit breaker. 
func (s *Settings) ChangeErrorRateThresholdPct(threshold uint32)
```

#### Call wrapper function

The circuit breaker will be implemented as a state machine with Closed, Opened and HalfOpen states and expose the following wrapper function for calls which we want to protect: 

```go
func (cb *CircuitBreaker[T]) Execute(run func() (T, error, boolean)) (T, error)
```

There is a third boolean parameter in the function argument above, in case the provided function doesn't return an error, but the caller still wants to treat it as a failure for the circuit breaker counting purposes (e.g. empty or corrupted result). 
Some other implementation return a callback, so that a caller can provide an interpretation result later, but this pattern opens the door to miss the failure count, hence proposed implementation does not offer that.

#### Errors

Sometimes implementations provide different error types for open state and half open states, but in this proposal defines a single error type for simplicity.

#### Integration

The implementation will live in a new package `circuitbreaker` package under https://github.com/tikv/pd/tree/master/client so that it can be referred from both http and grpc path. 

## Drawbacks

It is hard to configure the circuit breaker thresholds correctly. 
If the thresholds are too low, legitimate requests may be rejected. 
If they are too high, the circuit breaker may not trigger in time to prevent overload.
The proposal addressed this concern byt allowing for dynamic adjustment of thresholds so that they can be tuned based on observed load patterns.

## Alternatives

While existing solutions like rate-limiting in [Issue #4480](https://github.com/tikv/pd/issues/4480) and [PR #6834](https://github.com/tikv/pd/pull/6834) provide some protection, they are reactive and dependent on the server-side limiter thresholds being hit. These protections do not adequately account for sudden traffic spikes or complex feedback loops that can overload PD before those thresholds are reached. A proactive circuit breaker would mitigate these scenarios by preemptively tripping before PD becomes overwhelmed, ensuring a smoother recovery process.

There is an alternative approach of mitigating retry storm by limiting number of retries using different strategies (e.g. see https://brooker.co.za/blog/2022/02/28/retries.html), but this will not protect PD from external traffic spikes. 

## Unresolved questions

- Do we need to apply same pattern for TiKV?
- What is the good default configuration for the circuit breaker thresholds?
