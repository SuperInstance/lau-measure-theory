# lau-measure-theory

A Rust library for measure-theoretic probability: sigma-algebras, measures, measurable functions, Lebesgue integration, convergence theorems (MCT, Fatou, DCT), product measures, Radon-Nikodym derivatives, Lᵖ spaces, and agent probability spaces — all on finite discrete sample spaces.

## What This Does

`lau-measure-theory` implements the core constructions of measure theory and probability theory as explicit, computable Rust data structures:

- **Sigma-algebras** — trivial, power set, generated from seed sets, Borel (finite). Full axiom verification.
- **Measures** — probability measures, uniform, counting, Dirac (point mass). Axiom verification, monotonicity, subadditivity.
- **Measurable functions** — pointwise operations (add, multiply, scale, abs, power), preimages, composition, essential supremum.
- **Lebesgue integral** — simple function integration, general Lebesgue integral on finite spaces, expectation, variance, covariance, simple function approximation.
- **Convergence theorems** — Monotone Convergence Theorem (MCT), Fatou's Lemma, Dominated Convergence Theorem (DCT) — all with explicit verification.
- **Product measures** — product sigma-algebras, product measures via point-mass multiplication, Fubini's theorem for iterated integration.
- **Radon-Nikodym** — absolute continuity check, density computation dν/dμ, verification that ∫_A f dμ = ν(A).
- **Lᵖ spaces** — Lᵖ norms, L^∞ norm, Hölder's inequality, Minkowski's inequality.
- **Agent probability** — agent-scoped probability spaces, total variation distance, KL divergence.

## Key Idea

The crate works on **finite discrete sample spaces** where points are `enum Point { Int(i64), Str(String), Real(u64) }`. This makes every construction computable: sigma-algebras are `HashSet<BTreeSet<Point>>`, measures are `HashMap` lookups, integrals are finite sums, and convergence theorems can be checked explicitly. The trade-off is that you can't represent ℝ directly — but you get a fully verified, runnable implementation of every major theorem in a first measure theory course.

## Install

```toml
[dependencies]
lau-measure-theory = { git = "https://github.com/SuperInstance/lau-measure-theory" }
```

Requires `serde` and `nalgebra`.

## Quick Start

### Sigma-Algebra

```rust
use lau_measure_theory::{SigmaAlgebra, Point, MeasurableSet};

let space: MeasurableSet = (0..4).map(|i| Point::Int(i)).collect();
let sa = SigmaAlgebra::power_set(space.clone());
assert!(sa.verify_axioms());
assert_eq!(sa.len(), 16); // 2^4
```

### Probability Measure

```rust
use lau_measure_theory::{SigmaAlgebra, Measure, Point};

let space: MeasurableSet = (0..3).map(|i| Point::Int(i)).collect();
let sa = SigmaAlgebra::power_set(space);
let mu = Measure::uniform(sa);
assert!(mu.is_probability());
assert!(mu.verify_axioms());

// Custom probability
let sa2 = SigmaAlgebra::power_set(space);
let p = Measure::probability(sa2, &[
    (Point::Int(0), 0.2),
    (Point::Int(1), 0.3),
    (Point::Int(2), 0.5),
]);
```

### Lebesgue Integral & Expectation

```rust
use lau_measure_theory::*;

let space: MeasurableSet = (0..4).map(|i| Point::Int(i)).collect();
let sa = SigmaAlgebra::power_set(space);
let mu = Measure::uniform(sa);

let f = MeasurableFunction::new(mu.algebra().clone(), vec![
    (Point::Int(0), 1.0),
    (Point::Int(1), 2.0),
    (Point::Int(2), 3.0),
    (Point::Int(3), 4.0),
]);

let integral = lebesgue_integral(&f, &mu); // = 2.5
let ev = expectation(&f, &mu);             // = 2.5
let var = variance(&f, &mu);               // E[X²] - E[X]²
```

### Convergence Theorems

```rust
use lau_measure_theory::*;

// Monotone Convergence: fₙ ↑ f ⟹ lim ∫fₙ = ∫f
let result = monotone_convergence(&[f1, f2, f3], &limit, &mu);
assert!(result.holds);

// Fatou: ∫liminf fₙ ≤ liminf ∫fₙ
let result = fatou_lemma(&[f1, f2], &liminf_f, &mu);
assert!(result.holds);

// Dominated Convergence: |fₙ| ≤ g, fₙ → f ⟹ lim ∫fₙ = ∫f
let result = dominated_convergence(&[f1, f2], &limit, &dominating, &mu);
assert!(result.holds);
```

### Radon-Nikodym Derivative

```rust
let density = radon_nikodym_derivative(&nu, &mu).expect("ν ≪ μ");
// density(x) = ν({x}) / μ({x})
assert!(verify_radon_nikodym(&nu, &mu, &density));
```

### Agent Probability Spaces

```rust
let space = AgentProbabilitySpace::new("agent-1", measure);
let ev = space.expect(&f);
let tv = total_variation_distance(&space1, &space2);
let kl = kl_divergence(&space1, &space2);
```

## API Reference

### `sigma_algebra` — SigmaAlgebra

| Method | Description |
|---|---|
| `trivial(space)` | {∅, Ω} |
| `power_set(space)` | All subsets |
| `generate(space, generators)` | Close under complement and union |
| `borel_finite(n)` | Power set on {0, ..., n-1} |
| `contains(&set)` | Is the set measurable? |
| `complement(&set)` | Ω \ A |
| `union(&a, &b)`, `intersection(&a, &b)` | Set operations |
| `verify_axioms()` | Check all sigma-algebra axioms |

### `measure` — Measure

| Method | Description |
|---|---|
| `new(name, algebra, values)` | Explicit construction |
| `probability(algebra, point_masses)` | From point masses |
| `uniform(algebra)` | 1/n per point |
| `counting(algebra)` | μ(A) = |A| |
| `dirac(algebra, point)` | δ_x(A) = 𝟙{x ∈ A} |
| `measure(&set)` | Evaluate μ(A) |
| `is_probability()`, `is_finite()`, `is_sigma_finite()` | Classification |
| `verify_axioms()`, `verify_monotonicity()`, `verify_subadditivity()` | Axiom checks |

### `measurable_function` — MeasurableFunction

| Method | Description |
|---|---|
| `new(algebra, values)` | Point-value mapping |
| `eval(&point)` | Evaluate at a point |
| `add(&other)`, `multiply(&other)`, `scale(c)` | Pointwise operations |
| `abs_function()`, `pow_const(c)` | |f| and f^c |
| `preimage(&set)`, `preimage_interval(a, b)` | f⁻¹(B) and f⁻¹([a,b]) |
| `compose(&other)` | g ∘ f |
| `is_measurable(&codomain)` | Check measurability |
| `ess_sup(&measure)` | Essential supremum |

### `lebesgue_integral`

| Function | Description |
|---|---|
| `lebesgue_integral(&f, &μ)` | ∫f dμ |
| `expectation(&f, &P)` | E[f] (requires probability measure) |
| `variance(&f, &P)` | Var[f] = E[f²] - (E[f])² |
| `covariance(&f, &g, &P)` | Cov[f,g] = E[fg] - E[f]E[g] |
| `simple_approximation(&f, &𝒜, n)` | φₙ ↑ f approximations |

### `convergence` — Convergence Theorems

| Function | Returns | Description |
|---|---|---|
| `monotone_convergence(&[fₙ], &f, &μ)` | `ConvergenceResult` | fₙ ↑ f ⟹ lim∫fₙ = ∫f |
| `fatou_lemma(&[fₙ], &liminf, &μ)` | `ConvergenceResult` | ∫liminf ≤ liminf∫ |
| `dominated_convergence(&[fₙ], &f, &g, &μ)` | `ConvergenceResult` | |fₙ|≤g ⟹ lim∫fₙ = ∫f |

Each returns `{ theorem, limit_integral, integral_of_limit, holds, error }`.

### `product_measure`

| Function | Description |
|---|---|
| `ProductSpace::new(𝒜₁, 𝒜₂)` | Product measurable space |
| `product_algebra()` | 𝒜₁ ⊗ 𝒜₂ |
| `product_measure(&μ₁, &μ₂)` | μ₁ ⊗ μ₂ |
| `fubini_integral(&[(x,y,val)], &μ₁, &μ₂)` | Iterated integration |

### `radon_nikodym`

| Function | Description |
|---|---|
| `is_absolutely_continuous(&ν, &μ)` | ν ≪ μ? |
| `radon_nikodym_derivative(&ν, &μ)` | Compute dν/dμ |
| `verify_radon_nikodym(&ν, &μ, &f)` | Check ∫_A f dμ = ν(A) ∀A |

### `lp_spaces`

| Function | Description |
|---|---|
| `lp_norm(&f, &μ, p)` | ‖f‖ₚ = (∫|f|ᵖ dμ)^{1/p} |
| `linf_norm(&f, &μ)` | ‖f‖_∞ = ess sup |
| `holders_inequality(&f, &g, &μ, p)` | (‖fg‖₁, holds?) |
| `minkowski_inequality(&f, &g, &μ, p)` | (‖f+g‖ₚ, holds?) |

### `agent_probability`

| Type/Function | Description |
|---|---|
| `AgentProbabilitySpace::new(id, P)` | Agent-scoped (Ω, 𝒜, P) |
| `.probability(&A)`, `.expect(&f)`, `.var(&f)` | Convenience methods |
| `total_variation_distance(&s1, &s2)` | sup_A \|P₁(A) - P₂(A)\| |
| `kl_divergence(&s1, &s2)` | D_KL(P₁ \|\| P₂) |

## How It Works

1. **Sigma-algebras** are stored as a `HashSet` of `BTreeSet<Point>` — literally the set of measurable sets. `generate()` starts with the seed sets and iteratively closes under complement and union until no new sets appear. `verify_axioms()` checks ∅ ∈ 𝒜, Ω ∈ 𝒜, closure under complement, and closure under union.

2. **Measures** store a `HashMap<String, f64>` mapping encoded sets to their measure values. `verify_axioms()` checks μ(∅) = 0, non-negativity, and additivity for disjoint pairs.

3. **Measurable functions** are `Vec<(Point, f64)>` mappings. Preimages are computed by filtering points whose value falls in the target set. Operations (add, multiply, scale) are pointwise.

4. **Lebesgue integral** on a finite space is simply: ∫f dμ = Σᵢ f(ωᵢ) · μ({ωᵢ}). For simple functions: ∫φ dμ = Σₖ aₖ · μ(Aₖ).

5. **Convergence theorems** accept explicit sequences of measurable functions and check the theorem conditions (monotonicity, domination) and conclusion (limit of integrals = integral of limit).

6. **Product measures** encode pair points as `Point::Str("(p1,p2)")` and compute point masses as μ₁({x}) · μ₂({y}). Fubini's theorem reduces to a double sum.

7. **Radon-Nikodym** checks absolute continuity (μ(A) = 0 ⟹ ν(A) = 0 for all A), then computes the density f(x) = ν({x}) / μ({x}) pointwise. Verification confirms ∫_A f dμ = ν(A) for every measurable A.

8. **Lᵖ norms** are computed directly: ‖f‖ₚ = (Σ|f(ω)|ᵖ · μ({ω}))^{1/p}. Hölder and Minkowski inequalities are verified numerically.

## The Math

**Sigma-algebra axioms:** A collection 𝒜 of subsets of Ω is a σ-algebra if: (1) ∅ ∈ 𝒜, (2) A ∈ 𝒜 ⟹ Aᶜ ∈ 𝒜, (3) A₁,A₂,... ∈ 𝒜 ⟹ ⋃Aₙ ∈ 𝒜.

**Measure axioms:** A function μ: 𝒜 → [0,∞] satisfies: (1) μ(∅) = 0, (2) μ(⋃Aₙ) = Σμ(Aₙ) for disjoint Aₙ (countable additivity). On finite spaces, this reduces to finite additivity.

**Lebesgue integral:** For a simple function φ = Σaᵢ·𝟙_{Aᵢ}, ∫φ dμ = Σaᵢ·μ(Aᵢ). For general f on finite Ω, ∫f dμ = Σf(ω)·μ({ω}).

**Monotone Convergence Theorem:** If fₙ ≥ 0 and fₙ ↑ f pointwise, then lim_{n→∞} ∫fₙ dμ = ∫f dμ.

**Fatou's Lemma:** For fₙ ≥ 0, ∫liminf fₙ dμ ≤ liminf ∫fₙ dμ.

**Dominated Convergence Theorem:** If fₙ → f pointwise and |fₙ| ≤ g for some integrable g, then lim ∫fₙ dμ = ∫f dμ.

**Radon-Nikodym theorem:** If ν ≪ μ (ν absolutely continuous w.r.t. μ), then there exists a unique (μ-a.e.) measurable f ≥ 0 such that ν(A) = ∫_A f dμ for all A ∈ 𝒜. This f = dν/dμ is the density.

**Hölder's inequality:** ‖fg‖₁ ≤ ‖f‖ₚ · ‖g‖ₚ' where 1/p + 1/p' = 1.

**Minkowski's inequality:** ‖f + g‖ₚ ≤ ‖f‖ₚ + ‖g‖ₚ for p ≥ 1.

**Fubini's theorem:** ∫_{Ω₁×Ω₂} f d(μ₁⊗μ₂) = ∫_{Ω₁} [∫_{Ω₂} f(x,y) dμ₂(y)] dμ₁(x).

**Total variation distance:** δ(P₁, P₂) = sup_{A∈𝒜} |P₁(A) - P₂(A)| = ½ Σ|P₁({ω}) - P₂({ω})|.

**KL divergence:** D_KL(P₁ || P₂) = Σ P₁(ω) · ln(P₁(ω) / P₂(ω)).

## Tests

**43 unit tests** across 9 modules: sigma-algebra (trivial, power set, generated, Borel, complement, De Morgan), measure (uniform, counting, Dirac, axioms, subadditivity, monotonicity), measurable functions (basics, preimage, interval preimage, add, multiply), Lebesgue integral (simple function, uniform, expectation/variance, covariance, simple approximation, linearity), convergence (MCT increasing, MCT convergent, Fatou, Fatou strict inequality, DCT, DCT approximating sequence), product measures (product space, product measure uniform, Fubini), Radon-Nikodym (absolute continuity, derivative computation, density integrates to one), Lᵖ norms (L¹, L², Hölder, Minkowski), agent probability (creation, expectation, total variation, KL divergence).

## License

MIT
