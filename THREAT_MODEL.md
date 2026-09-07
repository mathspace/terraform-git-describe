# terraform-git-describe threat model

## Overview

Terraform external data-source helper emits git describe output for use as metadata; no AWS or Cloudflare resources are created (main.tf:5, README.md:3).

| Component | Source |
| --- | --- |
| Terraform external data source/output | main.tf:5 |
| Git description shell helper | git-describe.sh:1 |
| Standalone example with local state | examples/as_output/main.tf:1 |

| Deployment or workflow | Resource or capability | Configuration and precedence | Safe effective value or location | Readers, writers, or recipients | Enforcing control | Evidence or unknowns |
| --- | --- | --- | --- | --- | --- | --- |
| Terraform external query | Version description | inherited cwd/environment → git describe --always --tags --long --dirty=* → result.description | Description of Git repository selected by runner context, not necessarily module checkout | Terraform output and any caller-selected metadata consumer | Local process permissions; set -euo pipefail | git-describe.sh:3; main.tf:5 |

## Threat Model, Trust Boundaries, and Assumptions

Protected assets: Integrity of version metadata and Terraform runner process context (git-describe.sh:5).

This library has no cloud deployment identity or runtime service. Terraform invokes the module-relative executable, which invokes git from the inherited environment. Because the script does not change directory, Git describes the repository selected by the runner’s working directory and Git environment, not necessarily the helper’s own checkout (main.tf:6, git-describe.sh:5).

A source/tag editor can influence a description only where that checkout/ref state is used. A Terraform caller already controls its runner and may select the checkout deliberately; this is not a new privilege boundary. The meaningful separate actor is a downstream consumer that mistakes descriptive text for authorization or artifact provenance (README.md:7).

The script uses strict shell options and escapes quotation marks before generating external-provider JSON. The result is a string, not a signed attestation or digest bound to a built artifact. Caller obligations are to choose the intended checkout, trust the script and executable environment, handle failure, and treat the output as display metadata unless stronger independent checks exist (git-describe.sh:3, git-describe.sh:6).

The example uses a local terraform.tfstate backend and emits the module output; it does not deploy Lambda or modify a resource tag. Those uses are documentation suggestions. There is no network interface, credential broker, cloud provider or release workflow in the inventory. External-provider resolution, shell/Git availability and downstream consumers remain caller-owned. Review is offline architecture mapping, not execution-based validation (examples/as_output/main.tf:2, examples/as_output/main.tf:8, README.md:7).

## Attack Surface, Mitigations, and Attacker Stories

These are prioritized hypotheses for review, not confirmed vulnerabilities. Each depends on the stated actor and deployment prerequisites.

| Priority | Scenario and capability gain | Prerequisites | Impact | Existing controls | Mitigation | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| P1 | A downstream release gate could mistake attacker-influenced Git metadata for proof that an artifact is authorized. | A caller uses description for a security decision; attacker controls relevant tags/worktree but not release authority. | Promotion of an artifact that should not be trusted. | Module only exports text and README describes tracking metadata, not authorization. | Bind release decisions to reviewed revisions/artifact digests; keep description informational. | README.md:7; main.tf:9; git-describe.sh:5 |
| P2 | An inherited checkout or Git environment could produce plausible metadata for the wrong repository. | Runner context differs from the caller’s intended checkout. | Misattributed deployment version and impaired rollback/incident diagnosis. | Helper location is module-relative; no claim of explicit cwd binding in code. | Set and verify the checkout context at the call site. | main.tf:6; git-describe.sh:5 |
| P3 | Unexpected description contents or absent Git context could fail external JSON evaluation. | Unusual ref-derived text or runner lacking usable repository/Git executable. | Terraform metadata evaluation fails; downstream plan blocked. | Strict shell failure and explicit quote escaping. | Handle evaluation failure and validate output at real consumers without interpreting it as code. | git-describe.sh:3; git-describe.sh:6 |

## Severity Calibration (Critical, High, Medium, Low)

**Critical.** Critical requires an additional consumer that uses attacker-controlled metadata to grant consequential broad authority; no such consumer is present. A misleading tag alone cannot justify this level.

**High.** High could apply if a verified deployment workflow trusts this text to authorize execution or publication across a privilege boundary. The module’s output operation alone provides no such capability.

**Medium.** Medium fits a demonstrable wrong-version report that materially disrupts recovery, or a reachable metadata failure that blocks an important deployment. The operational dependency and attacker’s control must be shown.

**Low.** Low fits cosmetic description changes, local example failures and harmless malformed metadata rejected by Terraform. Authorized users describing their own checkout are exercising intended functionality, not attacking a separate principal.

This is an offline, source-backed architecture review. No application execution, cloud state inspection or vulnerability validation was performed.

Repository: https://github.com/mathspace/terraform-git-describe
Version: 560178fba1aa91fc046288b51d8c7b156f170747
