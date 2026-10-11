# Scope

This Working Group develops specifications, schemas and normative test vectors that let a party other than the one that ran a software agent verify:

1. what authority the agent acted under, and how that authority was derived and narrowed across delegation between agents;
2. that actions requiring approval were approved before they took place, by whom, and under which conditions, and that the action taken matched what was approved;
3. a tamper-evident record of what the agent was asked, decided and did, including verifiable redaction of that record;
4. what an agent declares it can do and may do, and how a task was placed on an agent;
5. how an arrangement of agents, their permissions and their approval requirements is described and checked for coherence; and
6. how the public keys needed for the above are published, discovered and changed over time.

The scope covers the core specifications for these subjects. It also covers **bindings**, which say how the core is carried in a specific environment or protocol (for example Kubernetes, edge devices, A2A, OAuth, MCP or OpenTelemetry), and **profiles**, which add optional constraints for a specific need (for example trace anchoring, compact encodings, plan-class authorisation, or mappings to regulatory frameworks).

The following are out of scope: policy languages; orchestration, scheduling and routing algorithms, except for the observable records they produce; the training or evaluation of models; and the functional-safety certification of physical systems.

Any changes of Scope are not retroactive.
