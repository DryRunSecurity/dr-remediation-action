# DryRun Remediation

GitHub Actions workflows that use an AI agent to propose fixes for security findings from [DryRun Security](https://www.dryrun.security).

| Workflow | Trigger | Result |
|---|---|---|
| **PR comment remediation** | DryRun Security comments on a pull request with findings | Inline code suggestions on that PR, each explaining why it fixes the finding, plus a summary comment with the full patch. Nothing is committed. |
| **Finding remediation** | You run it with one or more DryRun finding IDs, or an issue listing them | One pull request containing the fixes, with a detailed explanation of each finding and change. |

Every fix comes with the agent's explanation: the original issue and its impact, what changed and where, why the change addresses it, and anything still needed before merging (such as configuration or deployment steps).

You add a small workflow file to your repository that calls these workflows. The remediation logic stays in this repository.

## Quick start

### PR comment remediation

1. Add [`examples/github-actions/dryrun-comment-remediation.yml`](examples/github-actions/dryrun-comment-remediation.yml) to `.github/workflows/` on your **default branch**.
2. Add a repository secret `OPENAI_API_KEY` (see [Model providers](#model-providers) for Anthropic, Azure OpenAI, or other gateways).

When DryRun Security posts or updates a comment with findings on a pull request, the workflow runs automatically. You can also run it manually from the Actions tab with a PR number.

The core of the caller is:

```yaml
on:
  issue_comment:
    types: [created, edited]

jobs:
  remediate:
    permissions:
      contents: read
      pull-requests: write
      issues: read
    uses: DryRunSecurity/dr-remediation-action/.github/workflows/dryrun-comment-remediation.yml@v1
    secrets:
      MODEL_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

### Finding remediation

1. Add [`examples/github-actions/dryrun-findings-remediation.yml`](examples/github-actions/dryrun-findings-remediation.yml) to `.github/workflows/` on your **default branch**.
2. Add repository secrets:
   - `OPENAI_API_KEY`
   - `DRYRUN_API_KEY`: a DryRun API key that can read your account's findings.
   - `DRYRUN_ACCOUNT_ID`: your DryRun account ID.
3. Allow GitHub Actions to create pull requests: **Settings → Actions → General → Allow GitHub Actions to create and approve pull requests**. If the setting is disabled at the organization level, an organization owner must enable it there first.
4. Go to **Actions → Remediate DryRun findings → Run workflow** and supply exactly one of:
   - `finding_id`: a single finding ID.
   - `finding_ids`: up to 20 finding IDs, separated by commas or whitespace.
   - `issue_number`: an issue that lists finding IDs (format below).

All selected findings are fixed together in one pull request. When you start from an issue, the workflow also comments on the issue with a link to the pull request.

#### Issue format

List finding IDs under a `## DryRun finding IDs` heading. Put any other text under a different heading.

```markdown
## DryRun finding IDs
- 11111111-1111-4111-8111-111111111111
- 22222222-2222-4222-8222-222222222222

## Context
Anything reviewers should know.
```

Instead of the heading, you can paste DryRun dashboard links such as `https://app.dryrun.security/risk-register?finding=<finding-id>`.

Creating or labeling an issue does not start remediation on its own. Run the workflow with the issue's number.

## Configuration

### Inputs

Both workflows:

| Input | Default | Description |
|---|---|---|
| `provider` | `openai` | `openai` or `anthropic` |
| `model` | `gpt-5.5` (OpenAI) / `claude-sonnet-4-5` (Anthropic) | Model name, or deployment name for Azure OpenAI |
| `base_url` | Provider's official API | HTTPS endpoint of a compatible API or gateway |
| `use_responses_api` | `true` | Use the OpenAI Responses API. Set `false` for gateways that only support Chat Completions. Ignored for Anthropic. |

PR comment remediation:

| Input | Description |
|---|---|
| `pr_number` | PR to remediate. Required for manual runs. |
| `comment_id` | DryRun comment to use. Defaults to the most recently updated DryRun comment on the PR. |

Finding remediation:

| Input | Description |
|---|---|
| `finding_id` / `finding_ids` / `issue_number` | Findings to fix. Supply exactly one. |
| `account_id` | DryRun account ID. Overrides the `DRYRUN_ACCOUNT_ID` secret. |
| `base_branch` | Branch to fix. Defaults to the repository's default branch. |
| `finding_type` | Optional: `pullrequest`, `deepscan`, or `sca`. Needed only when an ID matches more than one type. |
| `dryrun_api_base_url` | DryRun API endpoint. Defaults to `https://simple-api.dryrun.security`. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `MODEL_API_KEY` | Yes | API key for your model provider or gateway |
| `MODEL_BASE_URL` | No | Private endpoint URL. Overrides `base_url` and is kept out of logs and published output. |
| `DRYRUN_API_KEY` | Finding remediation | DryRun API key |
| `DRYRUN_ACCOUNT_ID` | Finding remediation, unless `account_id` is set | DryRun account ID |

Pass secrets explicitly in the caller's `secrets:` block, as the examples do. GitHub doesn't pass them to reusable workflows automatically.

On GitHub Free, organization secrets aren't available to private repositories. Use repository secrets instead.

### Model providers

**OpenAI** is the default and needs only `MODEL_API_KEY`.

**Anthropic:**

```yaml
with:
  provider: anthropic
secrets:
  MODEL_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Azure OpenAI.** Set `model` to your deployment name. Store the endpoint, ending in `/openai/v1`, in a `MODEL_BASE_URL` secret so it stays private:

```yaml
with:
  provider: openai
  model: your-deployment-name
secrets:
  MODEL_API_KEY: ${{ secrets.AZURE_OPENAI_API_KEY }}
  MODEL_BASE_URL: ${{ secrets.AZURE_OPENAI_BASE_URL }}
```

**Other OpenAI- or Anthropic-compatible gateways.** Set `provider`, `model`, and either `base_url` or the `MODEL_BASE_URL` secret, plus the gateway's key. For Chat Completions-only OpenAI-compatible gateways, also set `use_responses_api: false`.

AWS Bedrock's native authentication is not supported.

The model provider receives your repository source and finding details. Only use a provider or gateway you trust with that data.

## How it works

Each workflow runs in two jobs.

1. **Propose** (read-only GitHub access):
   - Fetches the DryRun comment or findings and a snapshot of your source.
   - Runs the agent in an isolated container. The container receives only the model API key: no GitHub token or DryRun key.
   - The agent can only read and edit source files. It can't run shell commands, tests, or package managers, and it ignores repository agent-instruction files such as `AGENTS.md` and `CLAUDE.md`.
2. **Publish** (write access, no agent):
   - Checks that the PR, comment, or base branch hasn't changed since the proposal was made.
   - Checks that the patch applies exactly to the current source.
   - Posts the suggestions or opens the pull request.

The agent never edits `.github/` or environment files (`.env*`).

For **npm dependency fixes**, the agent edits `package.json` only. A separate container with no credentials then updates the root `package-lock.json` using `npm install --package-lock-only --ignore-scripts`.

For **PR comments**:
- Runs only for comments posted or edited by the DryRun Security bot, not for comments from people.
- Runs one at a time per PR. A newer DryRun update cancels an in-progress run, and a result that is out of date by the time it would be posted is skipped instead of failing.
- If the agent concludes no change is needed (for example, after DryRun reports the PR clean), the workflow removes its earlier suggestions and posts the explanation.

Manual runs require write access to the repository.

## Limitations

- **Review before merging.** Fixes are proposed for human review. Nothing is merged or deployed automatically, and the agent doesn't run your tests or build.
- **Run CI yourself on generated PRs.** Pull requests opened with the workflow's built-in token don't trigger your other workflows.
- **Same-repository PRs only.** Pull requests from forks aren't supported.
- **File limits:**
  - Only UTF-8 text files up to 1 MiB each are visible to the agent.
  - A proposal can change up to 30 files.
- **npm lockfiles:** automatic lockfile updates require a root npm `package-lock.json` without workspaces. Yarn, pnpm, Bun and authenticated private registries aren't supported.
- **Network:** the agent container needs network access to reach the model API. Its restrictions come from the tools it's allowed, not from blocking network access.
- **Artifacts:** proposals and agent output are kept as workflow artifacts for 7 days and contain source excerpts. On public repositories, anyone who can view the run can download them.
- **Public repositories:** finding remediation explains the vulnerability in the pull request description, which is public on a public repository. Consider this before remediating unfixed findings there.

## Versioning

Reference `@v1` to receive compatible updates, or pin a full commit SHA to control exactly which version runs.

## Development

The runtime lives in [`.github/dcode-remediation`](.github/dcode-remediation). The agent skills are pinned from [DryRunSecurity/external-plugin-marketplace](https://github.com/DryRunSecurity/external-plugin-marketplace).

- The workflows check out the runtime and skills at exact commits: four runtime checkouts (`95fc5a9b65e6055714fcf9ee443e5c3af66cffba`) and two skill checkouts (`918ad791da12f5b7bfa95d3e30a064a18bc9170a`).
- After changing the runtime or skills, commit the change, then update the corresponding pins.

```bash
PYTHONDONTWRITEBYTECODE=1 python3 -m unittest discover -s .github/dcode-remediation -p 'test_*.py' -v
docker build --tag dcode-remediation:0.1.66 .github/dcode-remediation
```

CI runs the tests and the Docker build, and lints the workflows and examples with actionlint.
