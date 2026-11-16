# rust-refs

Personal Rust reference notes, built with [mdBook](https://rust-lang.github.io/mdBook/).

Published at <https://deependujha.github.io/rust-refs/>.

## Local use

```sh
cargo install mdbook        # or: brew install mdbook
mdbook serve --open         # live reload at http://localhost:3000
mdbook build                # output in ./book
```

## Adding a chapter

1. Create `src/<topic>/<chapter>.md`.
2. Add it to `src/SUMMARY.md`.

Conventions: code first, short notes after. Every Rust block should compile on
its own (with its `use` lines) so it can be copied straight into a project.

## Deployment

`.github/workflows/deploy.yml` builds on every push and pull request, and
deploys to GitHub Pages on pushes to `main`. In the repository settings, set
**Pages -> Build and deployment -> Source** to **GitHub Actions**.
