# LLM2SAT — Verifiable Constraint Reasoning with SAT / MaxSAT

**Natural-language requirements → LLM constraint extraction → CNF/WCNF encoding → SAT/MaxSAT optimisation → independent verification → explanation**

A neuro-symbolic system in which a language model is used **only** to translate prose into a typed
constraint specification, and is never trusted to solve anything. Solving is done by a sound
SAT/MaxSAT engine; every solution is re-checked by a verifier that reads the specification rather
than the encoding; infeasible requests are answered with a *minimal* set of conflicting requirements
instead of a shrug.

Everything runs in a single notebook — [`LLM2SAT.ipynb`](LLM2SAT.ipynb) — on a free Colab runtime,
end to end, with no credentials and no external solver binaries. Total runtime is well under a
minute after the `pip install`.

<!-- > **Open in Colab:** upload the notebook, or use
> `https://colab.research.google.com/github/<user>/<repo>/blob/main/LLM2SAT.ipynb`
> once the repository is pushed (replace `<user>/<repo>`). -->

---

## The research question

> Can formal SAT/MaxSAT reasoning compensate for unreliable LLM reasoning, by forcing every
> generated solution to satisfy explicit constraints?

The answer this project measures, stated honestly: **soundness is fully recovered, fidelity is not.**

Once a specification exists, the pipeline *cannot* return an invalid solution — it returns a provably
optimal one or a proof of infeasibility. What remains at risk is **translation**: if the model drops
or invents a requirement, the answer is optimal for the wrong problem. So the project reports two
metrics rather than one:

| | what it measures | who can check it |
|---|---|---|
| **validity w.r.t. the extracted spec** | the runtime guarantee | the system itself, at answer time |
| **validity w.r.t. the ground-truth spec** | actual correctness | only a benchmark with labels |

The gap between them *is* the residual risk of putting a language model in front of a sound solver,
and it is the reason the extracted specification should be shown to a human before the solution is
published. Measuring that gap — rather than reporting a single flattering accuracy number — is the
point of the project.

---

## What is in the notebook

| § | Component | Substance |
|---|---|---|
| 1 | **Constraint schema** | closed, typed specification language; structural + referential validation |
| 2 | **Paired benchmark** | specifications sampled with a *planted feasible solution*, then rendered to prose — so prose ↔ label pairs come with ground truth and guaranteed satisfiability |
| 3 | **LLM extraction** | Claude with forced JSON-schema output, plus a validator-guided repair loop; an offline surrogate injects six realistic failure modes |
| 4 | **CNF/WCNF encoding** | sequential-counter and pairwise cardinality encodings, shared `IDPool`, hard clauses tagged by group, and a **soft-cardinality ladder** that turns a relaxation literal into a piecewise-linear objective inside pure weighted MaxSAT |
| 5 | **Solver layer** | interruptible portfolio wrapper (CaDiCaL 1.5.3/1.9.5, Glucose 4.2, MiniSat 2.2, MapleChrono, MergeSat 3), RC2, LSU |
| 5b | **MaxSAT solver from scratch** | weighted core-guided **Fu-Malik / WPM1** on an incremental SAT oracle: assumption-based solving, unsat-core extraction, core relaxation with AtMost(1) blockers, weighted core splitting, monotone lower bounds, honest in-call time budget |
| 6 | **Independent verification** | re-checks every hard rule from the specification in a different style from the encoder, recomputes the objective, and **differentially tests the encoding**: solver optimum ≠ recomputed penalty means the encoding is wrong |
| 7 | **Explanations** | satisfied/violated preferences with prices; for infeasible inputs a **group-MUS** (indicator assumptions + deletion-based minimisation) naming an irreducible set of conflicting requirements |
| 8 | **Pipeline** | the whole loop, plus a worked over-constrained example |
| 9 | **Trustworthiness experiment** | pipeline vs. unaided direct generation, four metrics, extraction-fidelity precision/recall/F1 |
| 10 | **MaxSAT engines head to head** | RC2 vs. RC2 with its refinements switched off vs. the from-scratch WPM1 vs. LSU, cross-validated on structured *and* random weighted instances |
| 11 | **Solver benchmarking** | four instance families (random 3-SAT at the phase transition, pigeonhole, graph colouring, the application's own encodings), wall-clock timeouts, PAR2 scoring |
| 12 | **ML algorithm selection** | 20 syntactic instance features → random forest → cross-validated selector, scored against every single solver, the single-best solver and the virtual-best oracle |
| 13 | **DIMACS / WDIMACS I/O** | round-trip export so instances can be run under Kissat, Open-WBO, EvalMaxSAT, or submitted to a MaxSAT Evaluation track |
| 14 | **Test suite** | 21 tests, including encoding soundness *and* completeness, objective agreement, MUS minimality, and "the pipeline never returns an invalid solution" under a deliberately unreliable extractor |
| 15 | **Artifacts, repo mapping, limitations** | CSVs, figures and DIMACS files zipped for download |

### The domain

Workforce shift scheduling. Hard requirements: coverage per shift and skill, availability, skill
gating, shift caps, at most one shift per day, mandatory rest after a late shift. Soft preferences:
preferred and avoided shifts, workload balance, staffing cost — each with a weight. Hard constraints
become hard clauses; preferences become weighted soft clauses. That split is exactly the MaxSAT
model, which is why MaxSAT and not plain SAT is the right formalism for "requirements + preferences".

---

## Running it

### Colab (recommended)

Open the notebook and *Runtime → Run all*. The first cell installs `python-sat` and (optionally)
`anthropic`; nothing else is needed.

### Locally

```bash
pip install python-sat numpy pandas matplotlib scikit-learn
jupyter notebook LLM2SAT.ipynb
```

`python-sat` is installed **without** the `[pblib,aiger]` extras: the notebook uses only the
cardinality encoders, which are in the base package, and the extras need a C++ toolchain (they fail
to build on stock Windows).

### With a real model instead of the surrogate

```python
import os
os.environ["ANTHROPIC_API_KEY"] = "sk-ant-..."
CONFIG["LLM_MODE"] = "api"           # then re-run from section 3 onwards
```

The `api` backend calls `claude-opus-5` with adaptive thinking and a strict JSON schema
(`output_config.format`), and feeds validator errors back as a repair prompt. The rest of the
pipeline is unchanged — that is the whole design.

### Configuration

Everything tunable is in one `CONFIG` dict in section 0: `LLM_MODE`, `MODEL`, `SEED`,
`SURROGATE_ERROR_RATE`, `N_EXPERIMENT_INSTANCES`, `BENCH_TIMEOUT`, `BENCH_SOLVERS`,
`MAXSAT_TIMEOUT`, `OUT_DIR`. Raise the timeouts and instance sizes for a serious benchmark run.

---

## Repository layout

The notebook is the executable paper; the package below is the maintained artefact. Section numbers
map one-to-one, so the notebook can be split into modules mechanically.

```
llm2sat/
├── llm2sat/
│   ├── parsers/
│   │   ├── constraint_schema.py     # §1   schema, validate_spec, fidelity metrics
│   │   └── requirement_parser.py    # §3   LLM backends, repair loop, normalisation
│   ├── encodings/
│   │   ├── cnf.py                   # §4   variables, cardinality helpers
│   │   └── wcnf.py                  # §4   soft clauses, soft-cardinality ladder
│   ├── solvers/
│   │   ├── pysat_solver.py          # §5   interruptible portfolio wrapper
│   │   ├── rc2_solver.py            # §5   RC2 / LSU adapters
│   │   ├── core_guided.py           # §5b  CoreGuidedMaxSAT (WPM1) from scratch
│   │   └── external_solver.py       # §13  DIMACS hand-off to kissat / open-wbo
│   ├── verification/
│   │   ├── constraint_checker.py    # §6   verify, cross_check
│   │   └── explanation.py           # §7   group_mus, explain
│   └── pipeline.py                  # §8
├── benchmarks/                      # §2, §11  instance generators
├── experiments/
│   ├── trustworthiness.py           # §9
│   ├── maxsat_comparison.py         # §10
│   ├── benchmark_runner.py          # §11
│   └── portfolio_selection.py       # §12
├── tests/                           # §14
├── notebooks/LLM2SAT.ipynb          # this notebook
├── requirements.txt
└── README.md
```

Running the notebook writes `llm2sat_artifacts/` (and a zip): experiment CSVs, benchmark runs and
features, figures, DIMACS/WDIMACS instances, and the demo specification, roster and explanation as
JSON.

---

## What this project does *not* claim

Being explicit about the limits is what separates a research artefact from a demo.

* **Offline numbers are a simulation.** The surrogate's error distribution was chosen by hand. Only
  `api` mode measures a model.
* **Template-rendered prose is easy prose.** Real requirement documents are ambiguous, contradictory
  and incomplete; the fidelity figures here are an upper bound.
* **One domain, one closed schema.** The soundness argument depends on every constraint type having a
  hand-written encoder *and* a hand-written checker. Extending the schema is manual work — that is
  the honest price of verifiability.
* **The algorithm-selection study is demo-scale.** A few dozen generated instances, no probing
  features, high cross-validation variance. The harness scales to the SAT Competition archives
  unchanged; the numbers do not transfer.
* **The from-scratch WPM1 is a teaching-grade solver.** It agrees with RC2 on every optimum it
  completes, which is the claim being made; it is slower, and on weighted instances it is fragile
  enough that some runs exhaust their budget. That is reported rather than hidden.
* **`balance_workload` and `minimize_cost` are scalarisations.** Genuinely multi-objective
  requirements want a Pareto front, not a weighted sum.

## Natural extensions, roughly by effort

1. **Confirm the extracted specification with the user before solving.** The measured fidelity gap
   says this is not optional; it collapses the residual risk.
2. **Self-repair from the MUS.** Feed the minimal conflicting set back to the model, ask which
   requirement to relax, re-solve — symbolic diagnosis driving neural revision.
3. **Ambiguity detection.** Sample *k* extractions at nonzero temperature; disagreement localises
   which sentence was ambiguous, turning extraction variance into a useful signal.
4. **Real benchmarks.** SAT Competition and MaxSAT Evaluation archives, probing features, and a
   runtime-regression selector instead of a winner classifier.
5. **A second domain** — course timetabling, test-suite minimisation, package dependency resolution —
   to show the architecture is not schema-specific. Weighted-MaxSAT test-suite minimisation reuses
   almost all of this machinery: coverage becomes hard clauses, execution cost and redundancy become
   weighted soft clauses.
6. **Encoding comparison.** Totalizer vs. sequential counter vs. pseudo-Boolean, measured — encoding
   choice usually matters more than solver choice.

---

## References

* PySAT — RC2 MaxSAT solver: <https://pysathq.github.io/docs/api/examples/rc2.html>
* CaDiCaL SAT solver: <https://github.com/arminbiere/cadical>
* Open-WBO MaxSAT solver: <https://github.com/sat-group/open-wbo>
* SAT Competition 2026 benchmark specification: <https://github.com/satcompetition/2026>
* Fu & Malik, *On solving the partial MAX-SAT problem* (SAT 2006) — the core-guided loop of §5b;
  Ansótegui, Bonet & Levy, *Solving (weighted) partial MaxSAT through satisfiability testing*
  (SAT 2009) — the weighted extension (WPM1).
* Ignatiev, Morgado & Marques-Silva, *PySAT: A Python toolkit for prototyping with SAT oracles*
  (SAT 2018).
* Xu, Hutter, Hoos & Leyton-Brown, *SATzilla: Portfolio-based algorithm selection for SAT* (JAIR
  2008) — the feature/selection methodology of §11–12.
