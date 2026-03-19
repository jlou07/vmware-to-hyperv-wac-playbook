# VMware to Hyper-V with Windows Admin Center

This repository contains a practical documentation set for migrating virtual
machines from VMware to Hyper-V by using the VM Conversion extension in
Windows Admin Center (WAC).

## Why This Repository Exists

Following recent VMware licensing and packaging changes, many organizations are
reassessing their hypervisor strategy. This playbook provides a structured,
risk-aware migration path that can be used for both technical delivery and
partner/customer conversations.

## Documentation Map

- Main overview: [docs/index.md](docs/index.md)
- Prerequisites: [docs/prerequisites.md](docs/prerequisites.md)
- Migration workflow: [docs/workflow.md](docs/workflow.md)
- Limitations and risks: [docs/limitations.md](docs/limitations.md)
- Hybrid positioning: [docs/hybrid.md](docs/hybrid.md)
- Full step-by-step article: [docs/article.md](docs/article.md)

## Recommended Reading Order

1. Start with [docs/index.md](docs/index.md)
2. Validate environment requirements in [docs/prerequisites.md](docs/prerequisites.md)
3. Follow execution guidance in [docs/workflow.md](docs/workflow.md)
4. Review caveats in [docs/limitations.md](docs/limitations.md)
5. Use [docs/hybrid.md](docs/hybrid.md) for strategic positioning
6. Use [docs/article.md](docs/article.md) for the full technical walkthrough

## Website Version

This documentation is also published with MkDocs at:

- https://jlou07.github.io/vmware-to-hyperv-wac-playbook/

## Build Locally

```bash
pip install mkdocs mkdocs-material
mkdocs serve
```

Then open the local URL shown in the terminal (typically
`http://127.0.0.1:8000`).

## Scope

The repository focuses on:

- Migration prerequisites across WAC, VMware, and Hyper-V
- A phased migration workflow for controlled cutovers
- Delivery limitations and risk communication
- A practical bridge toward hybrid and Azure-aligned strategy
