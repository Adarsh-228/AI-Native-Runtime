# Contributing to `ai-native-runtime`

We welcome contributions from researchers, systems architects, and software engineers. `ai-native-runtime` is an open-source, research-first project governed by transparency, peer review, and empirical validation.

---

## 1. Core Contribution Principles

1. **Research Before Implementation:** Substantial features must begin with documented research, an RFC, or an ADR before submitting implementation PRs.
2. **Prior Art & Alternatives:** Proposals must evaluate existing solutions (e.g., Ray, KubeEdge, LangGraph, Automerge) and justify novelty or optimization.
3. **Reproducible Evidence:** Performance or scalability claims must be supported by reproducible benchmarks.
4. **Modular Independence:** Contributions to specific crates or SDKs should maintain clear modular boundaries.

---

## 2. Developer Certificate of Origin (DCO)

All contributions must adhere to the Developer Certificate of Origin (DCO Version 1.1). By submitting a pull request, you certify that:
- The contribution was created in whole or in part by you, and you have the right to submit it under the project's open-source license; or
- The contribution is based upon previous work that is covered by an appropriate open-source license and you have the right to submit that work.

Include a `Signed-off-by` line in your git commit messages:
```bash
git commit -s -m "feat(scheduler): implement capability matching algorithm"
```

---

## 3. Contribution Workflow

1. **Discuss:** Open a Research Proposal or Feature Request issue to discuss the idea.
2. **Propose:** For architectural changes, submit an RFC in `rfcs/proposed/`.
3. **Implement:** Follow established formatting and test suites.
4. **Pull Request:** Submit a PR referencing the relevant issue/RFC. Ensure all CI checks and lints pass.

