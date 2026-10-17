# SUTRA documentation

Everything about how SUTRA is built, run and evaluated. Diagrams are Mermaid and render on GitHub.

## Start here

| Document | Read it for |
|---|---|
| [Simple guide](architecture/SIMPLE_GUIDE.md) | A plain-language tour of the whole system, with a diagram for every idea |
| [Technical & architecture guide](architecture/TECHNICAL_ARCHITECTURE_GUIDE.md) | The complete reference: data model, pipeline, detectors, models, API, frontend, platform |
| [Architecture views](architecture/README.md) | Context, containers, AWS deployment and release flow |
| [Tech decisions](architecture/TECH_DECISIONS.md) | Why each library and service, and what was rejected |

## Reference

| Folder | Contents |
|---|---|
| [`adr/`](adr/README.md) | 23 Architecture Decision Records |
| [`architecture/`](architecture/) | Guides above, plus the [Investigation Council](architecture/INVESTIGATION_COUNCIL.md) design |
| [`ml/`](ml/) | [Model lifecycle](ml/MODEL_LIFECYCLE.md) · [M7 features](ml/features.md) · [labeling](ml/labeling.md) · [rule baseline](ml/rule_baseline.md) |
| [`integrations/`](integrations/) | [Account Aggregator](integrations/ACCOUNT_AGGREGATOR.md) · [Kestrel Bank, the mock bank](integrations/MOCK_BANK.md) |
| [`operations/`](operations/) | [Runbook](operations/runbook.md) · [disaster recovery](operations/disaster-recovery.md) · [SLOs](operations/slo.md) · [cost](operations/cost.md) |
| [`security/`](security/) | [Threat model (STRIDE)](security/threat-model.md) |
| [`assets/`](assets/) | Screenshots used in the README |

## HackMatrix 5.0 submission

SUTRA was built by Team Admiral for HackMatrix 5.0 (problem statement FIN-04). The submission
material lives in [`hackathon/`](hackathon/):

| Document | What it is |
|---|---|
| [Demo script](hackathon/DEMO_SCRIPT.md) | The 8-minute on-site walkthrough, with timings and fallbacks |
| [FIN-04 evidence package](hackathon/FIN04_EVIDENCE_PACKAGE.md) | What the final-round brief asked for, mapped to files and commands |
| [Project audit](hackathon/PROJECT_AUDIT.md) · [Implementation status](hackathon/IMPLEMENTATION_STATUS.md) | The final-round starting point and what changed |
| [Presentation](hackathon/presentation/ppt-content.md) | Deck content, [diagrams](hackathon/presentation/diagrams/) and the [video script](hackathon/presentation/video/script.md) |
| [Stitch prompts](hackathon/STITCH_PROMPT.md) | The design prompts behind the investigator UI |
