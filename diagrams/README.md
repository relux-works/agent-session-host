# Diagrams

Architecture diagrams as code for the session-host specification ([`../spec/session-host.md`](../spec/session-host.md)).

## Structure

```
diagrams/
├── README.md                    this file: tools, commands, conventions
├── plantuml/
│   ├── _includes/theme.puml     shared look, the same palette as the relux-works wiki
│   ├── component/               module structure: what owns what
│   ├── deployment/              structure: what runs where
│   └── sequence/                runtime flows
└── artefacts/plantuml/          rendered SVG and PNG, committed
```

## Diagrams

| Diagram | Kind | Source | Renders |
| --- | --- | --- | --- |
| Starting a hosted primary session today | PlantUML sequence | [session-start-today.puml](plantuml/sequence/session-start-today.puml) | [SVG](artefacts/plantuml/session-start-today.svg), [PNG](artefacts/plantuml/session-start-today.png) |
| Doorbell delivery modes: today and the three target modes | PlantUML sequence | [doorbell-delivery.puml](plantuml/sequence/doorbell-delivery.puml) | [SVG](artefacts/plantuml/doorbell-delivery.svg), [PNG](artefacts/plantuml/doorbell-delivery.png) |
| One host per node with per-session holders | PlantUML deployment | [one-host-per-node.puml](plantuml/deployment/one-host-per-node.puml) | [SVG](artefacts/plantuml/one-host-per-node.svg), [PNG](artefacts/plantuml/one-host-per-node.png) |
| Park and resume through the board goal | PlantUML sequence | [park-and-resume.puml](plantuml/sequence/park-and-resume.puml) | [SVG](artefacts/plantuml/park-and-resume.svg), [PNG](artefacts/plantuml/park-and-resume.png) |
| Emergency stop (`keeper estop` over `curator session estop`) | PlantUML sequence | [emergency-stop.puml](plantuml/sequence/emergency-stop.puml) | [SVG](artefacts/plantuml/emergency-stop.svg), [PNG](artefacts/plantuml/emergency-stop.png) |
| Surviving a host restart: today and target | PlantUML deployment | [restart-survival.puml](plantuml/deployment/restart-survival.puml) | [SVG](artefacts/plantuml/restart-survival.svg), [PNG](artefacts/plantuml/restart-survival.png) |
| Module map: host, bridge, trust, launch, supervision, harnesses (Architecture v1) | PlantUML component | [module-map.puml](plantuml/component/module-map.puml) | [SVG](artefacts/plantuml/module-map.svg), [PNG](artefacts/plantuml/module-map.png) |
| Signed ingress: bridge, trust gate, injection, journal, projection (Architecture v1) | PlantUML sequence | [signed-ingress.puml](plantuml/sequence/signed-ingress.puml) | [SVG](artefacts/plantuml/signed-ingress.svg), [PNG](artefacts/plantuml/signed-ingress.png) |
| Controlled restart: park, forced stop, exit, resume bridge, report (Architecture v1) | PlantUML sequence | [controlled-restart.puml](plantuml/sequence/controlled-restart.puml) | [SVG](artefacts/plantuml/controlled-restart.svg), [PNG](artefacts/plantuml/controlled-restart.png) |
| Preserve versus rebuild: where each mode runs and where Remote Control sits (Architecture v1) | PlantUML deployment | [preserve-rebuild.puml](plantuml/deployment/preserve-rebuild.puml) | [SVG](artefacts/plantuml/preserve-rebuild.svg), [PNG](artefacts/plantuml/preserve-rebuild.png) |

## Tools

| Tool | Used for | Install |
| --- | --- | --- |
| PlantUML (re-rendered diagrams: 1.2026.8; untouched legacy renders: 1.2026.6) | turns each `.puml` source into SVG and PNG | `brew install plantuml`, or the JAR: `curl -L -o plantuml.jar https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar` |
| No Graphviz needed | all non-sequence diagrams use the built-in Smetana layout (`!pragma layout smetana`), including the migrated `one-host-per-node` and `restart-survival` | nothing to install |

## Render

From the repository root:

```bash
plantuml -tsvg -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/deployment/*.puml diagrams/plantuml/sequence/*.puml
plantuml -tpng -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/component/*.puml diagrams/plantuml/deployment/*.puml diagrams/plantuml/sequence/*.puml
```

The output directory is absolute because PlantUML resolves a relative `-o` against the folder of each source file. The same commands run through the JAR: `java -Djava.awt.headless=true -jar <tools>/plantuml.jar` in place of `plantuml`.

## Conventions

- One purpose per diagram, with a title and a caption naming its sources.
- Technical terms only, as in the specification.
- Colour says what a thing is: blue for node services, purple for agents and harness processes, amber for the board and committed files, grey for private state, green for people, teal for carriers and bridge endpoints.
- All non-sequence diagrams carry `!pragma layout smetana` right after `@startuml`, so they render without Graphviz.
- A change to a source is committed together with its re-rendered SVG and PNG.
