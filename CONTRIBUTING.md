
# Cross-Repo Governance Rules

**Canonical Ownership (No Drift):**

* **Global FRs + Traceability:** `cortex-os` (Integration repo)
* **Frontend Architecture + UI Registry:** `cortex-os-frontend`
* **Backend Architecture + Schema:** `cortex-os-backend`
* **API/IPC Contracts + Wiring Matrix:** `cortex-os-contracts`

**Cross-Repo Change Protocol:**

* If a change impacts another repo’s contract or document surface, you **must** open paired PRs and link them in the PR descriptions.
* **Integration Pinning:** After merging FE/BE/Contracts PRs, update the `cortex-os` submodule SHAs in a follow-up PR.
