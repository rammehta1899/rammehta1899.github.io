---
layout: post
title: 'Deterministic Secret State: Using keep.lock Manifests to Stop CI/CD Drift'
date: 2026-09-02 13:23:41 -0400
description: Aligning secret state with Git commits using committed keep.lock manifests
  prevents build failures and runtime secret drift across engineering teams.
categories:
- Platform Engineering
- DevOps
tags:
- secrets management
- cicd
- capy
- git
- platform engineering
author: Ram Mehta
issue_number: 13
issue_url: https://github.com/rammehta1899/blog-gen/issues/13
---

*Published September 2, 2026 at 1:23 PM ET*

Every platform engineer knows the specific frustration of a CI/CD pipeline breaking because a developer added a feature requiring a new environment variable, but forgot to update the staging environment secrets. Application code moved forward in Git, but secret configuration remained trapped in a completely separate web dashboard or unversioned cloud store. This gap between code state and secret state causes persistent build drift in modern software delivery. Tools like [Capy CLI](https://github.com/capysc/capy-cli) aim to solve this issue by introducing a committed `keep.lock` manifest that pins versioned secret revisions directly to your Git repository commits.

Decoupling application code from secret configuration breaks the guarantee of atomic deployments. Git gives teams exact point-in-time code snapshotting. Traditional secret managers, however, operate entirely out of band. When a pull request merges, your CI runner pulls the latest commit from Git, but fetches secrets asynchronously from an external API or remote key-value store. If someone altered a key name or updated a database URL out of sync with that commit, the build crashes during startup or fails midway through execution.

Capy addresses this mismatch by pairing Git repository state directly with a committed lockfile named `keep.lock`. Just as a `package-lock.json` or `Cargo.lock` pins dependency versions to a commit, `keep.lock` pins ciphertext revisions and branch references to the exact commit tree. Because secrets get encrypted locally on the developer machine prior to syncing to remote storage, plaintext values never hit the network. When a team member edits a secret or switches branches, Capy updates the lockfile. You commit `keep.lock` alongside your application code. When your CI/CD pipeline checks out the repository, it receives both the code and the precise manifest describing which secret revisions belong to that specific commit.

Avoiding runtime SDK imports is a key architectural decision in this model. Managing secrets inside application code often meant importing language-specific SDKs, handling SDK initialization failures, or configuring background sidecar daemons. Capy bypasses this runtime overhead by injecting decrypted secrets directly into process memory at execution time via `capy run`. Whether your software runs on Node, Go, Python, or Ruby, your application reads standard `process.env` or system environment variables without requiring custom code imports. In a CI runner or Docker build step, executing `capy run -- npm test` or `capy run -- go test ./...` populates environment variables in memory right before giving control to the test runner.

The cryptographic architecture backing this workflow centers on source-side encryption rather than standard server-side encryption at rest. Most cloud secret managers encrypt data on their own servers using project keys. If a provider's cloud infrastructure gets compromised or served a subpoena, plaintext values can theoretically be accessed. Capy forces encryption at the source on developer laptops. The storage server retains only membership records and wrapped ciphertext. Decryption requires a two-party operation. The central service strips an outer wrapper from a user's `key.enc` file, and the local machine strips the inner layer. Neither the remote service nor an attacker gaining full access to remote storage holds both keys.

For platform teams evaluating toolchain integration, details on installation options and architecture are documented on the [capy-cli project on GitHub](https://github.com/capysc/capy-cli). The project is distributed under the AGPL-3.0 license. The CLI installs quickly across build runners using standard package managers, including npm (`npm install -g @capysc/cli`), Bun (`bun add -g @capysc/cli`), or Homebrew (`brew install capysc/tap/capy`). Because the tool installs as a lightweight binary executable, adding it to your automated build workflows adds minimal startup overhead compared to pulling heavy container images or setting up complex secret rotators.

While lockfile-driven secret management solves environment drift conceptually, engineering leaders must weigh real operational tradeoffs before enforcing it across enterprise teams.

Lockfiles introduce merge conflict friction. When two developers independently modify secrets on separate feature branches, Git flags a conflict inside `keep.lock`. Resolving conflicts in an encrypted lockfile is difficult because human developers cannot inspect or merge raw ciphertext hashes manually. Teams need clear operational hygiene around branch syncing to minimize lockfile collisions.

Licensing presents another consideration. Enterprise legal teams sometimes raise flags regarding AGPL-3.0 software in automated toolchains. You need to clarify that the CLI runs as an external build-time executable rather than a linked library compiled into your distributed application binaries.

Cryptographic revocation changes offboarding dynamics. Executing `capy kick` renders the target user's local `key.enc` file cryptographically inert without forcing a mandatory, immediate key rotation cascade across the entire team. That design is clean. Still, offboarding protocols must account for third-party service credentials that human engineers may have viewed in plaintext while active on the project.

Comparing this approach to existing options highlights clear positioning. Mozilla SOPS provides file-based encryption using KMS or PGP, but lacks built-in team branch coordination and simplified developer TUIs. Centralized systems like HashiCorp Vault offer deep policy engine capabilities, but operating Vault clusters consumes significant platform engineering resources. SaaS platforms like Doppler or Infisical deliver streamlined interfaces, yet historically lean on server-side key management. Capy targets a distinct niche: local zero-trust encryption combined with Git-native lockfile workflows.

If you want to test deterministic secret state in your engineering organization, start small. Identify a single microservice suffering from high pipeline churn or frequent configuration errors. Integrate `keep.lock` into that pipeline, run it alongside your pull request workflows, and monitor how developers handle lockfile updates during concurrent feature work. You can explore the full CLI specification directly on [Capy's GitHub repository](https://github.com/capysc/capy-cli) to assess fit for your build infrastructure. The broader platform engineering question over the coming years is whether lockfile-backed manifests will become standard practice across infrastructure pipelines, or if legacy audit requirements will keep teams tied to central web dashboards.

## Further reading

- [Capy CLI Source Repository & Documentation](https://github.com/capysc/capy-cli)

## Further reading

- [https://github.com/capysc/capy-cli](https://github.com/capysc/capy-cli)

---
