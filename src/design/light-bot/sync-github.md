# Sync GitHub Repositories

## Introduction

When collaborating with external customers or partners to enhance and customize GitHub repositories, standard branching strategies can break down. Customers often have their own internal private Git servers (like GitHub Enterprise or Bitbucket). 

Previously, `light-bot` utilized a two-branch model (`master` and `sync`) that attempted to automatically merge changes between the two environments. This led to frequent and severe merge conflicts when multiple teams attempted to update the same repository simultaneously, as bots cannot intelligently resolve textual conflicts.

To solve this, `light-bot` has adopted a **Hub and Spoke** (Fork and Pull) model that strictly separates mirroring from contribution, entirely relying on Pull Requests for code integration.

## Architecture and Flow

The new workflow ensures that the customer's internal Git server acts as a "Spoke" while GitHub remains the "Hub" (Source of Truth). 

The `SyncGitRepoTask` executes hourly and follows this precise flow:

1. **Mirror Master**: The bot replicates the `master` branch from GitHub to the internal Git `master` branch. The internal `master` branch is treated as read-only for customer developers.
2. **Customer Development**: Customer teams create standard feature branches (e.g., `feature/custom-login`) on their internal Git server.
3. **Internal Approval**: Customers open a Pull Request on their internal Git system to go through their own internal approvals and security checks.
4. **Handoff (Rename to Sync)**: Once approved internally, the developer renames their feature branch from `feature/custom-login` to `sync`. This acts as a handoff signal for `light-bot`.
5. **Replicate to GitHub**: During the next hourly job, the bot detects the `sync` branch and pushes it to GitHub.
6. **GitHub PR Creation**: GitHub utilizes a workflow action to automatically create a Pull Request from `sync` to `master`.
7. **Merge and Cleanup**: The core internal team reviews and merges the PR on GitHub. 
8. **Automated Pruning**: On the subsequent bot run, `light-bot` checks if the `sync` branch exists internally and verifies if its commits are fully merged into `master`. If they are, the bot automatically drops the `sync` branch from the internal server, clearing the queue for the next feature.

## Edge Cases and Rules

To ensure this workflow operates smoothly, two strict rules must be observed:

### 1. Merge Commits Only (No Squash/Rebase)
To safely detect if a `sync` branch can be pruned from the customer's server, the bot executes `git merge-base --is-ancestor sync origin/master`. 

**Critical Rule:** The core team merging the PR on GitHub **MUST** use the **"Create a Merge Commit"** option. 
If the team uses "Squash and Merge" or "Rebase and Merge", GitHub generates entirely new commit hashes. As a result, the ancestor check will fail, the bot will think the branch is unmerged, and it will fail to clean up the `sync` branch on the customer's server.

### 2. Concurrency and Queuing
Because the bot only looks for a single `sync` branch as the handoff mechanism, customer teams cannot push multiple features simultaneously. 

If Team A renames their branch to `sync`, Team B must wait until Team A's PR is merged on GitHub and the bot deletes the `sync` branch before Team B can rename their feature to `sync`. This queuing mechanism is intentional; it serializes contributions and prevents massive, difficult-to-resolve merge conflicts across distributed systems.
