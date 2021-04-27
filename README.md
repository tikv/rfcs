# TiKV RFCs

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the TiKV community.

The "RFC" (request for comments) process is intended to provide a consistent
and controlled path for new features to enter the project, so that all
stakeholders can be confident about the direction the project is evolving in.

## How to submit an RFC

1. Copy `template.md` into `text/PRID-my-feature.md`.
2. Write the document and fill in the blanks.
3. Submit a pull request.

## Timeline of an RFC

1. An RFC is submitted as a PR.
2. Discussion takes place, and the text is revised in response.
3. The PR is accepted or rejected when at least two project maintainers reach consensus.
4. If accepted, create a tracking issue, refer it in the RFC, and finalize by merging.

## Style of an RFC

We follow lint rules listed in
[markdownlint](https://github.com/DavidAnson/markdownlint/blob/master/doc/Rules.md).

Run lints (you must have [Node.js](https://nodejs.org) installed):

```bash
# Install linters: npm install
npm run lint
```

## License

This content is licensed under Apachie License, Version 2.0,
([LICENSE](LICENSE) or http://www.apache.org/licenses/LICENSE-2.0)

## Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
