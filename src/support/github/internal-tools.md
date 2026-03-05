# Handling Internal Tools in Mirrored Repositories

When open-source repositories hosted on GitHub are replicated to internal customer environments (e.g., Bitbucket), customers often need to add internal directories for commercial scanning tools (code coverage, security vulnerabilities, etc.).

This document outlines strategies to keep internal tooling isolated from the upstream public GitHub repository to avoid accidental leaks or merge conflicts.

## Strategies for Isolation

### 1. The "Wrapper Repository" (Git Submodules) — *Highly Recommended*

Instead of adding the internal commercial tooling directly into the mirrored GitHub repository, the customer can create a brand-new internal Bitbucket repository that acts as a "wrapper."

*   **How it works:** The customer creates a new Bitbucket repo. They commit their commercial tooling configurations, scan scripts, and extra directories to *this* repository. Then, they add your GitHub open-source repository as a **Git Submodule** inside it.
*   **The pipeline:** When their CI/CD pipeline runs, it checks out the wrapper repository, pulls down the submodule (your open-source code), and runs the scanning tools against the submodule directory.
*   **Why it's great:** Your GitHub repository remains a 100% exact mirror. The customer never has to worry about accidentally pushing internal files back to your GitHub repo, and you never have to worry about merge conflicts during updates.

### 2. The Upstream/Downstream Branching Strategy

If the customer insists on having the internal files and the open-source code tracked in the exact same repository, they must use a branch isolation strategy.

*   **How it works:**
    1.  The customer maintains a `main` branch that is a **strict mirror** of your GitHub repository. Nothing internal is ever committed here.
    2.  They create an `internal-main` branch branched off `main`.
    3.  They commit their internal scanning directories and files to `internal-main`.
*   **Syncing updates:** When you release new code on GitHub, the customer pulls those changes into their `main` branch, and then runs `git merge main` into their `internal-main` branch.
*   **Contributing back:** If the customer finds a bug and wants to push a fix back to your GitHub, they branch off `main` (not `internal-main`), commit the fix, and push that feature branch back to GitHub.
*   **Why it's great:** It utilizes standard Git workflows. The internal directories literally do not exist in the history of the `main` branch, so they can never be synced back to GitHub.

### 3. Externalize the Configuration in CI/CD

Often, security and code-coverage tools require a configuration file (like a `.sonar` folder, Fortify configs, or pipeline YAMLs). Instead of committing these directly to the source code repository, the customer can inject them at runtime.

*   **How it works:** The customer keeps their Bitbucket mirrored repository completely identical to your GitHub repo. They place the scanning tool directories into a *second*, completely separate Bitbucket repository.
*   **The pipeline:** During their CI/CD build process, the pipeline clones the mirrored open-source repo, then immediately clones the "tooling" repo, copies the required directories into the workspace, and runs the scan.
*   **Why it's great:** It entirely decouples the source code from the infrastructure/tooling, avoiding any Git history modifications.

### 4. Gitignore (Only if the directories are *generated*, not committed)

If the commercial tools simply *generate* directories (like an `out/`, `coverage/`, or `reports/` folder) and those folders do not actually need to be tracked in Bitbucket's version control:

*   **How it works:** Ensure the customer does not run `git add` on those directories.
*   They can add the directories to a global `.gitignore` on their CI/CD build servers, or add `.git/info/exclude` locally in their Bitbucket repo environment.
*   **Note:** You *could* add these commercial directory names to the `.gitignore` file hosted on your public GitHub. It doesn't hurt your open-source repo to ignore standard commercial tool outputs (e.g., adding `.sonar/` to your public `.gitignore`), and it prevents the customer from accidentally committing them.

## Recommendation for Customers

"To ensure seamless updates from our upstream GitHub repository without merge conflicts or accidentally leaking your internal configurations, we recommend **not** committing your tooling directories directly to the mirrored source code branch. Instead, either keep your tooling in a parent wrapper repository using Git Submodules, or maintain a distinct `internal` branch that merges updates from our `main` branch but is never pushed back upstream."
