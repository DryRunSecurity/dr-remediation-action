# DryRun Remediation

Two reusable GitHub Actions workflows, powered by the existing Deep Agents Code runtime and DryRun remediation skills. Generation, validation, and publishing are centrally maintained here; consumers keep only a small caller defining events, permissions, inputs, and secrets.

| Workflow | Result | Caller example |
|---|---|---|
| [PR comment remediation](.github/workflows/dryrun-comment-remediation.yml) | Explained inline suggestions and a timeline summary/patch on the existing PR; no commits or pushes | [Comment caller](examples/github-actions/dryrun-comment-remediation.yml) |
| [Finding remediation](.github/workflows/dryrun-findings-remediation.yml) | One combined remediation PR for finding UUIDs supplied directly or through an issue | [Finding caller](examples/github-actions/dryrun-findings-remediation.yml) |

Both publish agent-written explanations of the original issue, exact changes, why they address it, and remaining prerequisites. Existing matching results are reused; an existing remediation PR can be inspected read-only to refresh its explanation without another commit.

## Install

**First release pending:** `v1` has not been published. The examples show the intended release interface. Before the first release, replace `@v1` with the full feature-branch commit SHA containing these workflows. Creating this PR does not publish a tag or release.

1. Copy the desired [caller example](examples/github-actions/) into your repository's `.github/workflows/` directory on the **default branch**. GitHub requires this for automatic `issue_comment` events and manual dispatch availability. You do not copy the runtime or central multi-job workflows.
2. Create repository secret `OPENAI_API_KEY`. Examples explicitly pass it as `MODEL_API_KEY`; secrets are not inherited implicitly. GitHub Free private repositories need repository secrets because organization secrets are unavailable to them.
3. For finding remediation, also create secret `DRYRUN_API_KEY` and variable `DRYRUN_ACCOUNT_ID`, or supply the account UUID when dispatching.
4. Keep the example's permissions and allow this public reusable workflow and its referenced actions in your Actions policy. Finding remediation requires repository **and organization** policy to allow GitHub Actions to create pull requests; it does not bypass that policy.

The core comment caller is:

```yaml
name: DryRun remediation
on:
  issue_comment:
    types: [created, edited]
permissions:
  contents: read
  pull-requests: write
  issues: read
jobs:
  remediate:
    uses: DryRunSecurity/dr-remediation-action/.github/workflows/dryrun-comment-remediation.yml@v1
    secrets:
      MODEL_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

Automatic runs accept only created/edited PR **timeline conversation comments** from `dryrunsecurity[bot]` (ID `142451713`, type `Bot`), not inline review comments. The full example also supports manual dispatch. Only open same-repository PRs are supported; forks are excluded. Manual comment runs and all finding runs require a repository writer.

A reusable workflow is called at the **job level**, as above. It supplies separate generation and publishing jobs. A root `uses: DryRunSecurity/dr-remediation-action@v1` step would require an `action.yml` and cannot encapsulate this multi-job separation; this repository deliberately exposes workflows instead.

## Finding IDs and issues

Install the [finding caller](examples/github-actions/dryrun-findings-remediation.yml), then select **Actions → Remediate DryRun findings → Run workflow**. Supply exactly one of:

- `finding_id`: one finding UUID.
- `finding_ids`: comma- or whitespace-separated UUIDs, up to 20 unique IDs. Duplicates are normalized and removed.
- `issue_number`: an issue in the caller repository with the format below.

All selected findings are resolved through the account-scoped API before editing. The workflow produces one combined remediation PR and, for an issue-based run, adds or updates a link on the source issue.

### Copy-paste issue format

Replace these example UUIDs with real DryRun finding IDs:

```markdown
## DryRun finding IDs
- 11111111-1111-4111-8111-111111111111
- 22222222-2222-4222-8222-222222222222

## Context
Describe relevant repository or deployment context here.
```

Keep the ID section limited to UUIDs, separated by whitespace/commas or listed as bullets. An unlabelled fenced list also works. The section ends at the next Markdown heading; put prose under `## Context`, not among the IDs. An alternative is to include DryRun risk-register links such as `https://app.dryrun.security/risk-register?finding=11111111-1111-4111-8111-111111111111` in the issue body.

**Current hookup is manual:** create the issue, then dispatch with its `issue_number`. Creating, editing, or labelling an issue does not automatically start remediation. No automatic issue-label trigger is included.

## Inputs and credentials

Shared `with` inputs:

| Input | Default | Meaning |
|---|---|---|
| `provider` | `openai` | `openai` or `anthropic` |
| `model` | Empty | `gpt-5.5` for OpenAI; `claude-sonnet-4-5` for Anthropic |
| `base_url` | Empty | Official provider API, or an explicitly configured HTTPS-compatible gateway |
| `use_responses_api` | `true` | OpenAI Responses API; set `false` for Chat Completions-only gateways; ignored for Anthropic |

Without an override, OpenAI uses `https://api.openai.com/v1` and Anthropic uses `https://api.anthropic.com`. There is no internal endpoint or credential fallback.

**Comment inputs:** `pr_number` is required for manual calls; optional `comment_id` defaults to the most recently updated verified DryRun comment on that PR. Automatic calls use the event's PR and comment.

**Finding inputs:** exactly one of `finding_id`, `finding_ids`, or `issue_number`, plus required `account_id`. Optional `base_branch` defaults to the caller's default branch; `finding_type` accepts `pullrequest`, `deepscan`, or `sca`. `dryrun_api_base_url` defaults to `https://simple-api.dryrun.security`.

**Secrets:** both workflows require `MODEL_API_KEY`, paired with the chosen provider and endpoint. Optional `MODEL_BASE_URL` supplies a private HTTPS endpoint and takes precedence over the public `base_url` input when nonempty. An empty or omitted secret preserves the public input or native default. Findings additionally require `DRYRUN_API_KEY`. The caller's built-in GitHub token is used automatically; no separate GitHub credential is accepted.

For a private Azure OpenAI-compatible endpoint, store the full resource URL ending in `/openai/v1` in repository secret `MODEL_BASE_URL` and the Azure API key in `OPENAI_API_KEY`. Add these settings to the calling job, retaining its existing target inputs and any `DRYRUN_API_KEY` mapping:

```yaml
with:
  provider: openai
  model: YOUR_AZURE_DEPLOYMENT_NAME
  use_responses_api: true
secrets:
  MODEL_API_KEY: ${{ secrets.OPENAI_API_KEY }}
  MODEL_BASE_URL: ${{ secrets.MODEL_BASE_URL }}
```

Pass the endpoint under `secrets`, not `with`: GitHub does not allow secret expressions in reusable-job inputs. The secret is used only during preparation and agent execution; it is not stored in the published context or prompt. The private runtime configuration is not bundled, and endpoint/hostname diagnostics and the API key are redacted from retained agent output and errors. This uses the compatible v1 API with a deployment name; no Azure-specific SDK or `api-version` parameter is needed.

For Anthropic, merge this into the caller's job, retaining any finding inputs and `DRYRUN_API_KEY` mapping:

```yaml
with:
  provider: anthropic
  model: claude-sonnet-4-5
secrets:
  MODEL_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

For an OpenAI-compatible gateway:

```yaml
with:
  provider: openai
  model: your-gateway-model
  base_url: https://gateway.example.com/v1
  use_responses_api: false
secrets:
  MODEL_API_KEY: ${{ secrets.MODEL_GATEWAY_API_KEY }}
```

Anthropic-compatible gateways use `provider: anthropic` with their `base_url`, model, and matching key. Only configure endpoints you trust with repository content and that key. **Native AWS Bedrock authentication is not supported**; a URL override does not add AWS authentication or translate its protocol.

## Execution boundaries

- Proposal jobs have read-only GitHub permissions. The DryRun key is passed only to finding preparation; the model container receives only the selected model credential, not GitHub or DryRun credentials.
- Publication runs on a separate runner with the necessary write permissions. It rechecks the target, current source, canonical before-images, and report before posting or committing; it runs no agent and receives no model or DryRun credentials in its steps.
- The agent has restricted filesystem tools, no shell execution, and no project instructions, hooks, or MCP. Protected/unsupported source paths are excluded. Model API networking remains enabled: this is **not OS-level network isolation**.
- An external credential-free npm container can regenerate a changed root manifest's lockfile, using only the manifest and original `package-lock.json` with lifecycle scripts disabled. Workspaces, alternative package managers, and authenticated private packages are unsupported.
- Application tests, builds, and deployment checks are **not run**. npm completion is not application validation. Review the explanations, operational prerequisites, and complete patch; individual suggestions can depend on other changes.
- Artifacts may contain sensitive source before-images and agent output. They are retained for seven days; agent stdout is not echoed in workflow logs.
- No automatic merges or deployments. Changes made with `GITHUB_TOKEN` do not trigger ordinary downstream Actions workflows, so arrange **explicit CI validation** before merging a generated PR.

## Maintainers and validation

The runtime lives here; the skills remain in [external-plugin-marketplace](https://github.com/DryRunSecurity/external-plugin-marketplace). Nothing is fetched from a mutable skill branch.

- Four runtime checkouts (both jobs in both workflows) pin `09cd3c78de765a0070a5ac7e2b257c5ba474c3de` in this repository.
- Two skill checkouts (proposal jobs only) pin `be4bc1c1ee311d52fc0c4f3cebbcd7fc4907214a` in the skill repository. That dependency is the skill-only [PR #13](https://github.com/DryRunSecurity/external-plugin-marketplace/pull/13).
- When changing runtime or skills, commit the implementation first, then update the corresponding four or two checkout pins together and validate before releasing. Merely moving a workflow release reference does not update these implementation pins.
- After review and merge, publish the first versioned release and `v1` reference. Compatible future releases may advance `v1`; consumers requiring immutable dependencies should pin a full workflow commit SHA. No release is published by the initial implementation PR.

Local checks, from the repository root:

```bash
PYTHONDONTWRITEBYTECODE=1 python3 -m unittest discover -s .github/dcode-remediation -p 'test_*.py' -v
docker build --tag dcode-remediation:0.1.66 .github/dcode-remediation
```

CI also lints the central workflows and caller examples with actionlint 1.7.12. The Docker build runs the pinned-runtime tool-policy check that is skipped when its dependencies are absent on the host. These are runtime/packaging checks, not application validation or proof of live model/API delivery.
