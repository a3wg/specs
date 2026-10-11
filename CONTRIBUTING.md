---
---

# Contributing

This Working Group follows the [Community Specification Contribution Policy 1.0](https://github.com/CommunitySpecification/1.0/blob/main/06-contributing.md). By contributing you agree to the [Contributor License Agreement](CONTRIBUTOR-LICENSE-AGREEMENT.md).

The policy describes the process: issues labelled Discussion, Proposal or Spec Change, and pull requests reviewed by an editor. A few things specific to these specifications:

- **Say which layer a change touches.** Core, binding or profile. A core change has to explain why it cannot be a binding or a profile ([GOVERNANCE.md](GOVERNANCE.md), rule 4).
- **Vectors come with the change.** A change to the wire format or the verification rules includes the vectors that show it, and their expected results.
- **Editorial changes are welcome and cheap.** Clarifications, typos and better examples need no notice period. They do not change a version.
- **Specify only what can be checked from outside.** If a verifier cannot check a property from a record, a key set and the text, that property belongs in a runtime, not here.

Sign off each commit under the Developer Certificate of Origin (`git commit -s`).

## Previewing the site

[a3wg.org](https://a3wg.org) is this repository, built by GitHub Pages: every `.md` file is a page, and `README.md` is the home page. To see a change as it will appear there, install Ruby 3 with Bundler, then:

```sh
bundle install
bundle exec jekyll serve
```

and open <http://127.0.0.1:4000>. A change that reads correctly on GitHub almost always reads correctly on the site; the preview is for checking links and tables.

The sidebar is set in `_config.yml`, under `defaults`, so that the Markdown files carry no front matter. A new page needs an entry there to appear in it. The exceptions are `CONTRIBUTING.md` and `LICENSE.md`, which start with an empty front-matter block, because GitHub Pages skips files with those names unless they have one.
