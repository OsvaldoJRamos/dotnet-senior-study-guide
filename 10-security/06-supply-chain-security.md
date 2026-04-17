# Supply Chain Security

OWASP A06:2021 (Vulnerable and Outdated Components) and A08:2021 (Software and Data Integrity Failures) are both supply-chain categories. After `event-stream`, `ua-parser-js`, `SolarWinds`, and `xz-utils`, every senior is expected to talk concretely about **what we ship, where it came from, and whether we trust it**.

## The three questions

1. **What's in our build?** — dependency inventory, SBOM.
2. **Do any known vulnerabilities affect it?** — scanning.
3. **Can we trust what we're pulling?** — signed packages, signed builds, provenance.

## Dependency inventory — SBOM

A **Software Bill of Materials (SBOM)** is the list of every component that went into a build, with versions and relationships. Regulations (US Executive Order 14028, EU Cyber Resilience Act) are pushing SBOMs toward mandatory.

### Two standards you must recognize

| | SPDX | CycloneDX |
|---|---|---|
| Maintainer | Linux Foundation | OWASP Foundation + Ecma International (TC54) |
| Status | ISO/IEC 5962:2021 | ECMA-424 |
| Typical use | Legal / license compliance, broad enterprise adoption | Security-focused; covers SBOM, SaaSBOM, VEX, CBOM (cryptography), HBOM (hardware), AI/ML-BOM |
| Format | JSON, YAML, RDF, tag-value | JSON, XML |

Both are open standards; many tools emit both. Pick one per program and stick to it; don't generate inconsistent SBOMs across teams.

### Generating an SBOM for a .NET project

Community and Microsoft tooling both exist. Common options:

- **`dotnet CycloneDX`** (community) — `dotnet tool install --global CycloneDX`, then `dotnet CycloneDX MyProject.sln -o ./bom`. Produces a CycloneDX JSON/XML SBOM.
- **Microsoft `sbom-tool`** — cross-language SBOM generator from Microsoft; produces SPDX 2.2 JSON. Used by Azure DevOps pipelines and the `Microsoft.Sbom.Targets` MSBuild package.
- **GitHub dependency graph** — automatically inferred for public repos and populates the dependency-review / security tab. Exports SPDX.

Attach the SBOM to the release artifact. An SBOM that only lives on your machine has no value.

## Vulnerability scanning

### `dotnet package list --vulnerable`

Available starting in **.NET SDK 9.0.300**. In .NET 10 SDK the command is **`dotnet package list`** (noun-first); in .NET 9 and earlier it's **`dotnet list package`** (verb-first).

```bash
# .NET 10+ form
dotnet package list --vulnerable --include-transitive

# .NET 9 form
dotnet list package --vulnerable --include-transitive
```

What the docs say: the `--vulnerable` option *"lists packages that have known vulnerabilities. Cannot be combined with `--deprecated` or `--outdated`"*. The data source is the NuGet `VulnerabilityInfo` resource; you can point to custom `<AuditSources>` in your `NuGet.config`.

A companion feature — **NuGet audit** — runs during `dotnet restore` on modern SDKs and emits warnings (or errors, depending on `<NuGetAuditLevel>`). Turn it on; fail CI on high severity.

### Dependabot (GitHub)

Per GitHub docs, Dependabot supports the **`nuget`** ecosystem. Minimal config:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

Dependabot opens PRs for outdated and vulnerable packages. Pair it with required status checks so updates ship automatically once tests pass.

### Other scanners worth recognizing

- **GitHub Advanced Security / Dependency Review** — comments on PRs when a dependency introduces a known vulnerability.
- **Snyk**, **Sonatype Nexus**, **JFrog Xray**, **Mend** — commercial SCA with richer reachability analysis.
- **OWASP Dependency-Check**, **Trivy**, **Grype** — open-source scanners. Trivy and Grype also scan container images and IaC.

## Package signing and provenance

### Signed NuGet packages

Microsoft signs all packages on `nuget.org` with a repository signature; authors can additionally attach an **author signature** using `dotnet nuget sign`:

```bash
dotnet nuget sign MyPackage.nupkg \
  --certificate-path mycert.pfx \
  --timestamper http://timestamp.digicert.com
```

Requires .NET 6.0.100 SDK or later. Always include a timestamp — otherwise the signature becomes invalid when the certificate expires.

`nuget.org` rejects packages signed with self-issued certificates. For enterprise internal feeds, you can enforce signature requirements in `NuGet.config` via `signatureValidationMode` and trusted signer lists.

### Sigstore

Sigstore (Linux Foundation project, now CNCF) is the modern answer to *"who signed this and can anyone verify it without PKI pain?"*. Three components you should know:

| Component | Role |
|---|---|
| **Cosign** | Client; signs container images, binaries, SBOMs — the thing developers run |
| **Fulcio** | Short-lived code-signing CA; binds an OIDC identity (Google, GitHub, Microsoft) to an ephemeral certificate |
| **Rekor** | Append-only transparency log of all signing events — public, tamper-evident audit trail |

The model is **keyless signing**: no long-lived private key to store or rotate. A GitHub Actions workflow authenticates as itself via OIDC, gets a one-minute cert from Fulcio, signs the artifact, records the event in Rekor.

For .NET, the current ecosystem integration is strongest around container images (via `cosign sign`) rather than NuGet packages directly. NuGet support for Sigstore is tracked in the NuGet team's backlog; as of the latest releases, the primary signing mechanism for `.nupkg` remains author and repository signatures with X.509 certificates.

### SLSA

SLSA ("Supply-chain Levels for Software Artifacts", pronounced "salsa") is a framework for measuring build-pipeline trustworthiness — useful vocabulary for designing provenance requirements, though not a tool you install.

## Base images and containers

If you ship containers, the supply chain extends to every layer.

- Prefer **distroless** or **chiseled .NET images** (`mcr.microsoft.com/dotnet/runtime-deps:*-chiseled`) for smaller attack surface.
- Rebuild regularly; a pinned image tag is a frozen-in-time vulnerability.
- Scan images in CI (Trivy, Grype, Snyk, Defender for Containers).
- **Sign the image** with Cosign; enforce signature verification at admission in Kubernetes (Sigstore policy-controller, Kyverno, OPA/Gatekeeper).

## NuGet hygiene checklist

- `packageSourceMapping` in `NuGet.config` — pin each package to a specific feed. Defends against dependency confusion.
- `NuGetAudit`: enabled by default in modern SDKs; ratchet `NuGetAuditLevel` up to `medium` or `low` when your team is ready.
- Lock files: commit `packages.lock.json` (`RestorePackagesWithLockFile=true`) so restore is reproducible.
- Prefer pinned versions over floating (`[1.2.3]` over `1.*`).
- Use an internal feed (Azure Artifacts, GitHub Packages, JFrog, ProGet) that proxies public NuGet and caches a **reviewed** copy.
- Remove unused dependencies aggressively; every one is a tax.

## Anti-patterns

- **Treating the SBOM as a checkbox.** If you never query it during an incident, you don't have one, you have a file.
- **Auto-merging every Dependabot PR.** Run the tests; a patch release can still break you (`Hyperium.xx`, `sweetalert2.xx`, countless examples).
- **Ignoring transitive dependencies.** Most CVEs are transitive. Always scan with `--include-transitive`.
- **Dependency confusion** — publishing an internal package name to public NuGet lets an attacker register a higher version and take over your build. Use `packageSourceMapping`.
- **"We pin versions, so we're safe."** Pinning prevents accidental upgrades; it does **not** protect against a compromised version you already trust. The defense is signatures + review.
- **Signing binaries with a key checked into the repo.** Stop. Use a hardware token, Key Vault, Secrets Manager, or Sigstore.

## Rule of thumb

Generate an SBOM on every build, scan on every PR, sign every release artifact, and rebuild base images weekly. None of those are expensive individually; together they catch 80% of supply-chain failures before they reach production.

---

[← Previous: Threat Modeling](05-threat-modeling.md) | [Next: Cryptography Basics →](07-cryptography-basics.md) | [Back to index](README.md)
