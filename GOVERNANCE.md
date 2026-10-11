# Governance

This Working Group, the **A3 Working Group** ([a3wg.org](https://a3wg.org)), follows the [Community Specification Governance Policy 1.0](https://github.com/CommunitySpecification/1.0/blob/main/05-governance.md). This document records the choices that policy leaves to the Working Group, and the additional rules these specifications keep.

## Maintainers and editors

The maintainers are the owners of the [a3wg](https://github.com/a3wg) organisation on GitHub, and the editors are the people who review pull requests. Both can be reached at [info@a3wg.org](mailto:info@a3wg.org). The governance policy requires that the process not be dominated by any single interest (§3.2), and the rules below say how the group widens.

## Implementations

The reference implementation is [Rashnu](https://rashnu.dev). A specification reaches 1.0 when its text is complete, its vectors cover every verification rule, and an implementation passes all of them. The rules below say what changes when there are more implementations.

## Rules in addition to the policy

1. **A change to the wire format or the verification rules** needs an issue, at least one week of public notice, and new or updated vectors. Where there is more than one maintainer, it also needs approval from a maintainer who did not write it.
2. **A breaking change** (a new major version, see [README.md](README.md#conventions)) to a specification with more than one conformant implementation needs agreement from those implementers, as well as rule 1.
3. **A change to the core is a last resort.** A new environment or protocol enters as a binding, and a new need enters as a profile. A proposal that seems to need a core change says why a binding or profile will not do.
4. **Every decision of consequence is made on a public issue,** and the reasoning stays on that issue.
5. **The reference implementation is not normative.** The text and the vectors are. A disagreement between them and the reference implementation is a bug in the implementation, unless it shows the text is wrong. Then the text is fixed through rule 1, never by pointing at what the implementation does.

## As the group grows

- **New maintainers** are invited from people who have contributed substantially to more than one specification, preferring those from organisations not yet represented.
- **More implementations.** When a second, independent implementation passes a specification's vectors, the two are compared on inputs beyond the vectors. Every disagreement either becomes a new vector or a correction to the text.
- **A neutral home.** The specifications move to a neutral foundation when either of these happens first: a second conformant implementation exists that this Working Group did not write, or maintainers come from more than one organisation. The move includes the names and the domains (see [NOTICES.md](NOTICES.md#names-and-marks)).
