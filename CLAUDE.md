# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for creating **SNS topics** with security and naming conventions built in. The template follows the nested stack pattern and is designed to be referenced by parent CloudFormation stacks.

**Key characteristics:**

- Nested CloudFormation template (referenced via `TemplateURL`)
- Creates either a Standard or FIFO SNS topic, selected by a parameter
- Parameterized topic naming with project, base name, environment, region, and optional CI suffix
- Optional KMS encryption of the topic via a key alias, key ARN, or key ID
- Optional cross-account access through an `AWS::SNS::TopicPolicy` driven by account IDs and actions
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                     # Nested template: SNS topic, optional topic policy
├── parameters.json                   # Parameter values used by CI (Environment, CiSuffix)
└── stack-config.json                 # Stack name, template file, and parameter file used by CI

.github/workflows/
├── ci.yaml                           # Reads stack-config.json and calls the reusable CI build workflow
├── release.yaml                      # Semantic release on push to main
├── create-branch.yaml                # Auto-create feature branches from issues
├── claude.yaml                       # Claude Code GitHub integration
├── claude-code-review.yaml           # Claude Code review on pull requests
├── notify.yaml                       # Notifications
└── setup-environments.yaml           # GitHub environment setup

.env/
└── environments.yaml                 # Environment-to-AWS mapping and regions (ci, devl; us-east-1)

scripts/plugins/
├── release.config.js                 # Semantic-release configuration
└── (other release plugins)           # Commit analysis, notes generation, publish, verify-conditions

.claude/
└── .skills/                          # Project skills (README, CI workflow, contribution, package.json)

.devcontainer/
└── devcontainer.json                 # Dev container setup (Node.js 20)

package.json                          # Dependencies: semantic-release, commitizen
README.md                             # Template documentation and usage examples
CHANGELOG.md                          # Release history
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat`, `fix`, and breaking changes trigger releases.

## Key Architecture Concepts

### Nested Stack Pattern

This repo provides a **nested stack template**, referenced from parent/root CloudFormation stacks via `TemplateURL`. The template is self-contained and exports outputs for cross-stack references.

- **Parent stack** calls: `AWS::CloudFormation::Stack` with `TemplateURL` pointing to S3
- **Nested template** returns values through its `Outputs` section, with `Export` names of the form `${AWS::StackName}-<OutputKey>`
- Parent retrieves outputs via `!GetAtt NestedStack.Outputs.OutputKey`

### Topic Naming Convention

Topic names are built deterministically from parameters:

```bash
# Without CiSuffix
{ProjectName}-{SnsTopicBaseName}-{Environment}-{Region}

# With CiSuffix
{ProjectName}-{SnsTopicBaseName}-{Environment}-{Region}-{CiSuffix}

# FIFO topics (TopicType=FIFO) additionally end in .fifo, which AWS requires for FIFO topic names
```

Example: `proj-ztc-sns-topic-devl-us-east-1` (Standard), `proj-ztc-sns-topic-devl-us-east-1-build42.fifo` (FIFO with CI suffix)

Naming gives:

- Environment isolation
- Consistent naming for infrastructure automation

### Parameter-Driven Configuration

The template has no standalone/integrated modes. All behavior is selected by parameters:

- **`TopicType`**: `Standard` (default) or `FIFO`. `ContentBasedDeduplication` applies only to FIFO topics.
- **`KmsKeyId`**: empty for no SSE; otherwise an alias (`alias/my-key`), key ARN, key ID, or multi-Region key ID (`mrk-...`). The KMS key policy must also grant any cross-account principals `kms:Decrypt`/`kms:GenerateDataKey`; the topic policy alone is not enough.
- **`CrossAccountIds`**: empty skips the `SnsTopicPolicy` resource entirely. A comma-separated list of 12-digit account IDs creates a policy granting `CrossAccountActions` (`sns:Publish`, `sns:Subscribe`, or both) to those accounts on this topic. Changing this value and updating the stack updates the policy.

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates an SNS topic and, optionally, a cross-account topic policy.

**Key inputs:**

- `ProjectName` (default `proj-ztc`): Project prefix, lowercase letters, numbers, and hyphens
- `SnsTopicBaseName` (default `sns-topic`): Base name component
- `Environment` (default `devl`): Environment label (e.g. devl, stag, prod)
- `CiSuffix` (default empty): Optional suffix for CI/CD unique deployments
- `TopicType` (`Standard` | `FIFO`)
- `ContentBasedDeduplication` (`true` | `false`)
- `KmsKeyId` (optional)
- `CrossAccountIds` and `CrossAccountActions` (optional)

**Key outputs (exported as `${AWS::StackName}-<name>`):**

- `TopicArn`: Topic ARN
- `TopicName`: Topic name
- `CrossAccountPolicyApplied`: `true` when a cross-account topic policy was created

**Resources:**

- `SnsTopic` (`AWS::SNS::Topic`): always created. Tagged with `Environment` and `ManagedBy=CloudFormation`.
- `SnsTopicPolicy` (`AWS::SNS::TopicPolicy`): created only when `CrossAccountIds` is set.

### `.github/workflows/ci.yaml`

**Triggered on:**

- Manual `workflow_dispatch`
- Pushes to `feature/**` and `bug/**` branches
- Pull requests targeting `main`

Each trigger runs only when `cloudformation/**` or `.github/workflows/ci.yaml` changes.

**Phases:**

1. **Load config:** reads `cloudformation/stack-config.json` for the stack name, template file, and parameter file, and writes them to the job summary.
2. **CI build:** calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with environment `ci`, passing the stack name, template file, and parameter file. Validation, deployment, and cleanup are handled there.
3. **Changelog and release:** on `main` only, generates `CHANGELOG.md` and creates a GitHub release.

### `cloudformation/stack-config.json` and `cloudformation/parameters.json`

- `stack-config.json` names the stack (`cfn-nested-aws-sns-stack`), the template, and the parameter file used by CI.
- `parameters.json` is a flat JSON object of parameter values (currently `Environment` and `CiSuffix`). Other parameters use template defaults unless added here.

### `.github/workflows/release.yaml`

**Triggered:** On push to main

**Process:**

1. Analyze commits (conventional format: `feat:`, `fix:`, `BREAKING CHANGE:`)
2. Generate release notes
3. Update CHANGELOG.md
4. Create GitHub release and tag
5. Commit version bump

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` → MAJOR bump (0.1.0 → 1.0.0)
- Other commits → no release

## Testing & Validation

**Template validation:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Lint (used during development):**

```bash
cfn-lint cloudformation/template.yaml
```

**Manual stack deployment:**

`aws cloudformation deploy` takes parameter overrides as `KEY=VALUE` pairs, not as the flat JSON in `parameters.json`:

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-aws-sns-stack-devl \
  --parameter-overrides Environment=devl TopicType=Standard \
  --region us-east-1
```

The CI workflow runs the validate and deploy cycle automatically on the triggers above.

## AWS Credentials & Environment Variables

**GitHub environment `ci` requires:**

- `AWS_REGION`: CloudFormation deployment region
- `AWS_ACCOUNT_ID`: AWS account to deploy into
- `OIDC_ROLE_NAME`: IAM role name for OIDC trust (uses `arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}`)
- `CFN_TEMPLATES_S3_BUCKET`: S3 bucket where templates are stored

**OIDC setup:** The CI workflow uses AWS OIDC for keyless auth. The GitHub OIDC provider must trust the specified role. The workflow requests `id-token: write`.

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying the Template

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** if the CI parameter values need to change
3. **Validate locally** with `aws cloudformation validate-template` (and `cfn-lint` if available)
4. **Create a PR** with a conventional commit message (e.g., `feat: add KMS key parameter`)
5. **CI validates and deploys** automatically on the triggers above
6. **Merge to main** → release workflow creates the version tag and GitHub release

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
