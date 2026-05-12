# ADR-013: Native RPITIT for Async Traits Instead of #[async_trait]

## Status
Accepted

## Date
2026-05-12

## Context
Rust historically required the `#[async_trait]` proc macro to define `async fn` in traits because async functions in traits were not supported by the language. This macro uses trait objects (boxing), which has runtime overhead and requires `dyn Trait + Send` explicitly. As of Rust 1.75, Return Position Impl Trait in Traits (RPITIT) is stabilized, allowing async-like patterns in traits without boxing.

## Decision
Discourage both `#[async_trait]` and `#[allow(async_fn_in_trait)]` in all `codex-rs` traits. Use native RPITIT with explicit `Send` bounds instead:

```rust
// Preferred
trait MyClient {
    fn fetch(&self, url: &str) -> impl std::future::Future<Output = Result<Response>> + Send;
}

// Implementation (still uses async fn — satisfies the contract)
impl MyClient for HttpClient {
    async fn fetch(&self, url: &str) -> Result<Response> { ... }
}
```

Reference commit `3c7f013f9735` / PR `#16630` established this convention.

## Alternatives Considered

### #[async_trait] macro
- Pros: Familiar ergonomics, works on stable Rust before 1.75
- Cons: Boxes every future (heap allocation per call); requires `dyn Trait` syntax; hides the actual return type from the compiler (worse optimization)
- Rejected: Runtime overhead and ergonomic downsides outweigh familiarity

### #[allow(async_fn_in_trait)]
- Pros: Uses native async fn syntax with no macro
- Cons: Suppresses the warning without providing explicit Send bounds; callers cannot use the trait across thread boundaries without explicit Send
- Rejected: The Send bound must be explicit for safe use in Tokio's multi-threaded runtime

## Consequences
- New traits with async methods must spell out the RPITIT form explicitly
- Implementations may use `async fn` which implicitly satisfies the `impl Future<...> + Send` contract
- Code review should flag any new `#[async_trait]` or `#[allow(async_fn_in_trait)]` usage
- The Rust toolchain requirement of 1.93.0 ensures RPITIT is available
