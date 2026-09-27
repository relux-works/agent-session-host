# Diagrams

Architecture diagrams as code for the session-host specification ([`../spec/session-host.md`](../spec/session-host.md)).

## Structure

```
diagrams/
├── README.md                    this file: tools, commands, conventions
├── plantuml/
│   ├── _includes/theme.puml     shared look, the same palette as the relux-works wiki
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

## Tools

| Tool | Used for | Install |
| --- | --- | --- |
| PlantUML (the renders above: 1.2026.6) | turns each `.puml` source into SVG and PNG | `brew install plantuml`, or the JAR: `curl -L -o plantuml.jar https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar` |
| Graphviz `dot` (the renders above: 14.0.4) | the layout engine PlantUML uses for the deployment diagram | `brew install graphviz` |

## Render

From the repository root:

```bash
plantuml -tsvg -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/deployment/*.puml diagrams/plantuml/sequence/*.puml
plantuml -tpng -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/deployment/*.puml diagrams/plantuml/sequence/*.puml
```

The output directory is absolute because PlantUML resolves a relative `-o` against the folder of each source file.

## Conventions

- One purpose per diagram, with a title and a caption naming its sources.
- Technical terms only, as in the specification.
- Colour says what a thing is: blue for node services, purple for agents and harness processes, amber for the board and committed files, grey for private state, green for people.
- A change to a source is committed together with its re-rendered SVG and PNG.
