# Repository instructions

- This repository links three independent Git repositories as submodules. The platform calls the backend; the backend owns PostgreSQL and orchestration; agents executes VM workflows.
- For creating, running, and debugging workflows through the local backend, read [docs/agent-workflows.md](docs/agent-workflows.md). For implementing VM providers, read the agents repository's [provider guide](https://github.com/openlybuilt/agents/blob/main/docs/vm-providers.md).
- Preserve the submodule pins intentionally. Update them only to reviewed commits and keep the parent setup instructions consistent with the pinned code.
- Do not expose credentials, placeholder values, customer data, recordings, or live-view URLs. Follow the user's instructions before starting real workflows or incurring provider charges.
- Do not create or modify README.md or other documentation files unless the user explicitly requests it.
