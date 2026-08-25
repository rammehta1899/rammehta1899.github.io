---
layout: post
title: Two-Party Envelope Decryption and Zero-SDK Secret Injection
date: 2026-08-25 14:04:20 -0400
description: How zero-trust envelope decryption and process boundary injection solve
  developer secret sync without runtime SDKs or key rotation cascades.
categories:
- Engineering
- Security
tags:
- secrets
- security
- platform engineering
- devops
- zero-trust
author: Ram Mehta
---

*Published August 25, 2026 at 2:04 PM ET*

Platform teams spent the last decade building elaborate mechanisms to solve a simple problem: getting environment variables into running applications. We built centralized key-value vaults, deployed background daemon sidecars, and required application teams to import language-specific SDKs. Every approach introduced trade-offs. Runtime SDKs tie your code to a vendor's API lifecycle. Centralized vaults turn security incidents into catastrophic breach events. Daemon sidecars consume memory overhead across hundreds of ephemeral containers. A CLI toolchain called [capy-cli on GitHub](https://github.com/capysc/capy-cli) takes a different path, combining two-party envelope decryption with process boundary injection.

Traditional secret management relies heavily on central server trust. When an application boot sequence requests database credentials, it authenticates against a vault service. The vault decrypts stored ciphertext using a central master key, then hands back raw secrets over TLS. If an attacker gains read access to the vault storage layer or executes a subpoena against the provider, all plaintext secrets stored at rest become vulnerable. Encrypting at rest protects disks, but it does not protect against infrastructure compromise at the service provider layer.

Capy shifts this cryptographic trust boundary directly to the developer machine. It encrypts raw `.env` files locally before transmission. The backend infrastructure stores membership records and wrapped ciphertext, but never holds master keys, project keys, or plaintext environment values. Decryption requires a two-party cryptographic sequence. When an engineer runs a build, the remote service strips the outer layer of `key.enc`. The local client machine then strips the inner layer. Neither the server nor the client can complete decryption alone.

This architecture fundamentally alters the blast radius of a security breach. If an enterprise backend or storage bucket leaks entirely, the attacker receives inert outer envelopes. They lack the local hardware keys required to unwrap the inner cryptographic layer. Conversely, if a laptop is stolen, the local key envelope remains useless without the remote server completing the outer unwrapping step.

```
+-----------------------------------------------------------------------+
|                         TWO-PARTY DECRYPTION                          |
|                                                                       |
|  +--------------------+         Outer Wrap         +---------------+  |
|  |   Remote Service   | -------------------------> |  Local Client |  |
|  |  (Strips Outer)    |                            | (Strips Inner)|  |
|  +--------------------+                            +---------------+  |
|            |                                               |          |
|            v                                               v          |
|    Ciphertext Only                                 Injected process.env |
+-----------------------------------------------------------------------+
```

Managing team access presents another historical pain point for engineering managers. When a developer leaves a team, standard operational procedures require revoking their access and rotating every shared credential they could reach. Rotating database passwords, API tokens, and cloud keys across dozens of microservices consumes hours of engineering effort. Teams often defer credential rotation because of deployment risks, leaving active credentials exposed long after an employee departs.

Cryptographic revocation fixes this operational gap through targeted key invalidation. Running `capy kick` against a departing teammate invalidates the remote outer unwrap authorization for that specific user. Their local `key.enc` becomes cryptographically inert instantly. Remaining team members continue operating with their existing local keys without interruption. No database passwords need to be rotated, and no active Kubernetes deployments need emergency patches.

Environment state management also struggles with version drift across branches. Developers routinely switch between feature branches that require distinct feature flags or local service URLs. Updating an `.env` file manually during branch switches leads to mysterious local build failures. Capy mirrors git workflows by pairing branch state with a committed manifest file named `keep.lock`. This manifest pins environment configurations directly to git commits, ensuring environment states travel alongside source code.

Process injection avoids embedding third-party SDK dependencies inside application codebases. Instead of importing vendor modules into Node, Python, Go, or Ruby runtimes, developers invoke their standard start commands through `capy run -- npm run dev`. The CLI wrapper decrypts environment variables in memory and injects them straight into the process boundary as native `process.env` values.

This zero-SDK philosophy mirrors broader architecture shifts toward isolated context management across engineering workflows. As platform teams attempt to synchronize knowledge across AI developer tools and engineering agents, tools like [OzBrain](https://ozbrain.com) explore shared state primitives so teams avoid manually re-sharing context across disconnected tools. Whether handling secrets or operational context, keeping environment logic out of core business code reduces technical debt.

```bash
# Standard workflow using Capy CLI
npm install -g @capysc/cli

# Edit environment variables in a local terminal TUI
capy edit

# Inject secrets into application execution without SDKs
capy run -- python main.py
```

Evaluating any platform security tool requires inspecting trade-offs and vendor constraints. Capy is released under the AGPL-3.0 license, which may trigger scrutiny from corporate legal departments cautious about copyleft licenses in development toolchains. Because two-party decryption requires the remote service to strip the outer encryption wrap, local execution requires network connectivity to the Capy service during initial unwrapping. Offline development environments will require explicit caching or offline session capabilities.

Security tools must prove their longevity through operational stability under network partitions. Platform engineers considering this architecture should test edge cases around automated CI/CD pipelines, containerized build runners, and air-gapped deployment environments. You can review the complete open-source implementation at [capy-cli on GitHub](https://github.com/capysc/capy-cli) to evaluate its cryptographic primitives against your organization's compliance requirements.

## References

- [Capy CLI Repository](https://github.com/capysc/capy-cli)
- [OzBrain Shared Agent Brain](https://ozbrain.com)

## Further reading

- [https://github.com/capysc/capy-cli](https://github.com/capysc/capy-cli)
- [https://ozbrain.com](https://ozbrain.com)

---
