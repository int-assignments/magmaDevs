# magmaDevs

# Lava Network – Provider Pairing System

Implement the **filtering** and **scoring** core of Lava’s decentralized provider-pairing engine.

---

## Background

Lava is a decentralized network that matches **RPC service providers** with **consumers**.
Consumers request a *pairing list* from the Lava blockchain; the chain consults on-chain provider metadata and the consumer’s policy to return the best-fit providers.

Your task: implement the **core filtering and ranking logic** that lives inside the chain node.

---

## Data Structures

```go
type Provider struct {
    Address  string   // Bech32 or Hex
    Stake    int64    // micro-LAVA
    Location string   // ISO-3166 alpha-2 (e.g., "US")
    Features []string // Supported capabilities, e.g., ["eth", "archive"]
}

type ConsumerPolicy struct {
    RequiredLocation string   // Empty string → no location preference
    RequiredFeatures []string // Must ALL be present
    MinStake         int64    // micro-LAVA
}

type PairingScore struct {
    Provider   *Provider
    Score      float64            // 0.0 … 1.0 (higher = better)
    Components map[string]float64 // "stake", "features", "location"
}
```

---

## Functional Requirements

1. **Filter** providers → keep only those satisfying the policy.
2. **Score** each candidate on a 0-to-1 scale.
3. **Rank** by total score and return the **top 5**.
4. Must be safe & efficient under concurrent access.

---

## Filters

| Filter             | Rule                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| **LocationFilter** | Keep provider if `policy.RequiredLocation == ""` **or** `provider.Location == RequiredLocation`. |
| **FeatureFilter**  | Provider must support **all** `policy.RequiredFeatures`.                                         |
| **StakeFilter**    | Provider’s `Stake ≥ policy.MinStake`.                                                            |

---

## Scoring

Scores are **normalized to `[0,1]`** and combined as you see fit (e.g., weighted average). Suggested component calculators:

| Component         | Formula (example)                                                                              |
| ----------------- | ---------------------------------------------------------------------------------------------- |
| **StakeScore**    | `stake / maxStakeAmongCandidates`                                                              |
| **FeatureScore**  | `(extraFeatures / totalFeatures)` where `extraFeatures = provider.Features - RequiredFeatures` |
| **LocationScore** | `1.0` if locations match, else `0.5` (or linear decay if you prefer)                           |

Store each component in `PairingScore.Components["stake"]`, etc.

---

## Concurrency Expectations

* `GetPairingList` may be invoked **concurrently** by many goroutines.
* Protect shared state (e.g., cached max-stake) with sync primitives *or* design the function as stateless.
* Use goroutines or worker pools if parallelism improves performance for large provider sets.

---

## Interface Contract

```go
type PairingSystem interface {
    // Step 1 – filtering
    FilterProviders(providers []*Provider, policy *ConsumerPolicy) []*Provider

    // Step 2 – scoring
    RankProviders(providers []*Provider, policy *ConsumerPolicy) []*PairingScore

    // Step 3 – top-5 (ties resolved by deterministic tiebreaker)
    GetPairingList(providers []*Provider, policy *ConsumerPolicy) ([]*Provider, error)
}
```

---

## Design Guidelines

1. **Normalised scores**: final `Score` must be `0.0 ≤ s ≤ 1.0`.
2. **Edge cases**:

   * Empty provider list → return `ErrNoProviders`.
   * Fewer than five valid providers → return what you have.
3. **Performance**: O(*N*) preferred; avoid quadratic scans.
4. **Thread-safety**: no data races under `go test -race`.
5. **Clarity**: idiomatic Go, descriptive names, godoc comments.

---

## Evaluation Criteria

| Aspect                          | Weight |
| ------------------------------- | ------ |
| Correct filtering & scoring     | ★★★★☆  |
| Concurrency safety & efficiency | ★★★★☆  |
| Code organisation & readability | ★★★☆☆  |
| Documentation / comments        | ★★☆☆☆  |

---

## Submission

1. Place your code under `pairing/` (or similar) and add tests.
2. Document design choices, weights, and concurrency strategy in **`README.md`**.
3. Zip the repo (or share a Git URL) and include the README.
