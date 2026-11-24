# CheeseBread Analysis

- UFMG Compiler's Lab

## Cangjie and range analysis

[Cangjie](https://cangjie-lang.cn/) is a general-purpose, statically typed language designed for application development across several computing environments. CheeseBread extends the Cangjie compiler with an integer range analysis pass that runs over CHIR, the compiler's intermediate representation.

A range analysis over-approximates the possible values of each integer variable at every program point. That information lets the compiler answer questions that constant or conventional interval analysis may miss, such as whether a loop has reached a stable bound or whether a branch is unreachable. It can also support downstream optimizations, including dead-branch elimination and more precise integer operations.

<!-- Small example with identified ranges -->

```cj
func foo() {
    var i = 0           // i ∈ {0}
    while (i < 10) {
        i++             // i ∈ [0,9:1]
    }
    if (i != 10) {      // i ∈ {10}
        println("dead") // i ∈ ∅ (⊥)
    }
}
```

## Abstract domain

The integer domain combines finite sets with strided intervals. The exact set $\{v_1, \ldots, v_k\}$ records constants while $k \leq N$, where `INT_VALUE_SET_SIZE` controls $N$. Once the set limit is exceeded, the value collapses to an interval

$$
[l, u : s] = \{l, l+s, l+2s, \ldots, u\},
$$

with lower bound $l$, upper bound $u$, and stride $s \geq 1$. Bounds may be constants or $\pm\infty$. Booleans use a separate abstract value.

The lattice is ordered by precision: $\bot$ is the empty set, finite sets and strided intervals describe successively less precise states, and $\top = (-\infty,+\infty:1)$ accepts every integer. Joining values enlarges the approximation. A set can grow until $N$ is reached, after which the implementation derives a strided interval from the GCD of its values.

```mermaid
graph BT
    %% Edges go from more precise to less precise (a ⊑ b). Example with N = 3.
    BOT["⊥ = ∅"] --> S0["{0}"] & S2["{2}"]
    S0 & S2 --> S02["{0, 2}"]
    S02 --> S024["{0, 2, 4}"]
    S024 -->|"set overflows N,<br/>collapse to stride"| I082["(0, 8, 2)"]
    I082 -->|"stride refinement"| I081["(0, 8, 1)"]
    I082 -->|"bound widening"| I0P2["(0, +∞, 2)"]
    I081 & I0P2 --> I0P1["(0, +∞, 1)"]
    I0P1 --> TOP["⊤ = (−∞, +∞, 1)"]

    classDef set fill:#c8e6c9,stroke:#2e7d32,color:#000
    classDef si fill:#bbdefb,stroke:#1565c0,color:#000
    classDef note fill:#fff,stroke:#999,stroke-dasharray:4 3,color:#000
    class S0,S2,S02,S024,L1 set
    class I082,I081,I0P2,I0P1,L2 si
    class L3,L4,L5 note
```

## Constraint extraction

Range analysis is formulated as a system of dataflow constraints over SSA variables. `CompetitionRangeAnalysis` traverses CHIR values and adds equations to a `ConstraintGraph`; helper constraints for constants such as `0` and `true` are inserted explicitly.

The solver currently models assignments and constants, SSA $\phi$ nodes, arithmetic and comparisons, intersections with bounds, and related integer operations. The principal constraint kinds include:

- `InitializationConstraint`: $x = c$, for an integer constant $c$.
- `InitializationIntegerTop`: initializes a variable to the Cangjie `Int64` top range.
- `PhiConstraint`: $x = \phi(x_1, \ldots, x_k)$ joins values from multiple control-flow predecessors.
- `IntersectionConstraint`: $x = y \cap [l,u]$, with bounds that may reference another variable symbolically.
- Arithmetic and comparison constraints: equations that evaluate expressions using abstract integer or Boolean values.

## Solver pipeline

<details>
<summary><strong>Range Analysis pipeline</strong> (click to expand)</summary>

```mermaid
flowchart TD
    %% ── Pipeline (what the outside world sees) ──
    CHIR([CHIR program]):::input
    CG[(Constraint graph)]:::structure
    SCC[[Tarjan SCC decomposition]]:::analysis
    ORDER[/Topological SCC order/]:::order
    OUT[(Abstract state<br/>variable → range)]:::result

    CHIR -->|generate constraints| CG
    CG --> SCC
    SCC --> ORDER
    ORDER --> SOLVER
    SOLVER --> OUT

    subgraph SOLVER["Range solver · fixpoint per SCC"]
        direction TB
        NEXT{More SCCs?}:::decision
        CYC{SCC cyclic?}:::decision
        EVAL[Single-pass evaluation]:::eval
        W[Widening<br/>grow bounds upward<br/>to force termination]:::widen
        F[Symbolic future resolution<br/>bind deferred bounds]:::future
        N[Narrowing<br/>tighten monotonically]:::narrow
        STABLE{Stable?}:::decision
        COMMIT[Commit SCC ranges]:::commit

        NEXT -->|yes| CYC
        CYC -->|no: acyclic| EVAL --> COMMIT
        CYC -->|yes: cyclic| W --> F --> N --> STABLE
        STABLE -->|no: rerun| W
        STABLE -->|yes| COMMIT
        COMMIT --> NEXT
    end

    classDef input fill:#bbdefb,stroke:#1565c0,color:#000
    classDef structure fill:#eceff1,stroke:#455a64,color:#000
    classDef analysis fill:#ffe0b2,stroke:#ef6c00,color:#000
    classDef order fill:#fff9c4,stroke:#f9a825,color:#000
    classDef decision fill:#e1bee7,stroke:#7b1fa2,color:#000
    classDef eval fill:#e0f2f1,stroke:#00897b,color:#000
    classDef widen fill:#ffcdd2,stroke:#c62828,color:#000
    classDef future fill:#fff59d,stroke:#f9a825,color:#000
    classDef narrow fill:#c8e6c9,stroke:#2e7d32,color:#000
    classDef commit fill:#b2dfdb,stroke:#00695c,color:#000
    classDef result fill:#80cbc4,stroke:#00695c,color:#000

    style SOLVER fill:#fafafa,stroke:#455a64,stroke-width:2px,stroke-dasharray:6 4,color:#000
```

</details>

The constraint graph is decomposed with Tarjan's algorithm and reversed into topological order, so producers are evaluated before consumers. Each strongly connected component is solved independently:

1. **Growth analysis** repeatedly evaluates every constraint and joins newly discovered values. Set limits and widening delays control how precision is reduced.
2. **Future resolution** replaces symbolic intersection bounds, such as bounds derived from another variable, with the values currently known for that variable.
3. **Narrowing analysis** reevaluates constraints from a bottom state and intersects the result with the previously widened state. This recovers tighter bounds while the domain is moving toward a fixed point.

The result is a sound over-approximation of the values reaching each analyzed SSA variable.

## Benchmarks and validation

The test harness under `test/` exercises the analysis with static assertions (`sassert`). These assertions have no runtime meaning; they describe properties that range analysis should prove after compilation.

- `test/handmade/`: small programs designed to expose individual behaviors such as loops, arithmetic, arrays, recursion, and control-flow merges.
- `test/flavors/`: focused widening and narrowing cases. For example, `fudge.cj` verifies that `i` is exactly `10` after a bounded loop.
- `test/wcet/`: real-time and worst-case execution benchmark programs adapted from the WCET benchmark collection. They cover nested loops, sorting, digital filters, matrix operations, bit operations, recursion, and generated control flow.
- `test/CangjieBench/`: 164 Cangjie programs used to measure analysis scale and code generation success across larger source inputs.

Run a suite after building the compiler:

```bash
./test/run_all.sh all
./test/run_all.sh handmade
./test/run_all.sh wcet
./test/run_all.sh flavors
./test/run_all.sh cangjiebench
```

The harness compiles each case with `-O2 -j1`, reports static assertion failures, and compares expected output for handmade cases that provide `expected.txt`. WCET cases receive a generated input file and CangjieBench programs are compiled as static libraries.

Set-size sensitivity can be measured with `evaluate_set_size.sh`. It rebuilds the compiler for exact-set capacities $1,2,4,8,16,32,64$, runs `test/CangjieBench/compile.sh` for each capacity, and writes `results-N.csv` files containing variable counts, constraint counts, bitwidth reduction, and analysis time.

## Setup

- Prerequisites
  - Python3
  - Clang-{16,18,20}
- [reference](https://gitcode.com/Cangjie/cangjie_build/blob/main/doc_en/linux.md)
- Initialize submodules

```bash
git submodule update --init --recursive
cd cangjie_runtime
git checkout v1.2.0-beta.02
```

- Run the setup script. `./setup.sh`

## Documentation

A full account of the range analysis algorithm implemented in this repository is available in this [technical report](https://lac-dcc.github.io/pubs/TechReports/LaC_TechReport032026.pdf). How to cite:

```bib
@techreport{Borges26,
  title        = {Cangjie Range Analysis},
  author       = {Bernardo Borges and Elisa Frohlich and Vinicius Silva and Rafael Paniago and Pereira, Fernando Magno Quint{\~a}o},
  year         = {2026},
  institution  = {The Federal University of Minas Gerais},
  number       = {03/2026},
  url          = {https://lac-dcc.github.io/pubs/TechReports/LaC_TechReport032026.pdf}
}
```
