# NetworkNT documentation

Read the published Light-4j Platform documentation at **[www.networknt.com](https://www.networknt.com/)**.

This repository contains the mdBook source. Chapters live in [`src/`](src/), navigation is maintained in [`src/SUMMARY.md`](src/SUMMARY.md), and book settings are in [`book.toml`](book.toml).

## Publishing through GitHub

Yes—publication is automated by the existing [Build and Deploy mdBook workflow](.github/workflows/deploy.yml). View its [GitHub Actions runs](https://github.com/networknt/networknt-doc/actions/workflows/deploy.yml).

On every push to `master`, the workflow:

1. Installs mdBook `0.5.4` and mdbook-mermaid `0.17.1`.
2. Runs `mdbook build` to generate `book/`.
3. Uses `peaceiris/actions-gh-pages@v4` and the repository secret `DOCS_DEPLOY_TOKEN` to publish `book/` to the `master` branch of [`networknt/networknt.github.io`](https://github.com/networknt/networknt.github.io).
4. Sets the custom domain to `www.networknt.com`. GitHub Pages serves the destination repository's `master` branch from its root.

The public address is the custom domain above; this is an organization Pages site, rather than a project site under `/networknt-doc/`. Local edits appear online after they are committed and pushed to `master` and publication succeeds. The deployment token must retain permission to write to the destination repository.

## Local preview

Install the versions used by CI:

```sh
cargo install mdbook --version 0.5.4 --locked
cargo install mdbook-mermaid --version 0.17.1 --locked
```

From this repository's root, run:

```sh
mdbook serve --hostname 127.0.0.1 --port 3000
```

Open [http://127.0.0.1:3000](http://127.0.0.1:3000). Use another port if 3000 is already occupied.

To build without starting a server:

```sh
mdbook build
```

Generated HTML is written to `book/`.
