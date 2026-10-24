XA-TRUST — Reproducible Evidence Validation

Overview

This project provides a GitHub Actions workflow pattern for generating and validating execution evidence.

The pipeline distinguishes YAML parsing, workflow structure, observed execution, test results, and claim-scoped evidence validation.

Important: A successful workflow run does not, by itself, establish ARM-TTK conformance, external service state, or reconciliation with the XA Banking Ledger GT7.

Repository structure

.github/
└── workflows/
    ├── homologation-runner.yml
    └── evidence-validation.yml
ci/
├── requirements.txt
└── validate_evidence.py
tests/
└── test_validate_evidence.py
README.md

The workflow files must be stored in ".github/workflows/". Code blocks in this README are documentation only and are not executed by GitHub Actions.

Workflow responsibilities

1. Homologation runner

File: ".github/workflows/homologation-runner.yml"

Responsibilities:

- Run on "workflow_dispatch" and the configured push branch.
- Install the declared Python dependencies.
- Execute the configured test suite.
- Generate an evidence manifest containing the run ID, run attempt, repository, commit SHA, reference, RUID, timestamp, and observed test outcomes.
- Upload the manifest as an execution-specific artifact.
- Fail the job if dependency installation or tests fail.

The manifest is generated evidence, not an independent cryptographic attestation.

2. Evidence validation

File: ".github/workflows/evidence-validation.yml"

Responsibilities:

- Run after the homologation workflow completes or through an explicitly configured manual invocation.
- Retrieve the artifact from the exact source run and attempt.
- Check the manifest against the expected source run, commit, attempt, RUID when supplied, and execution conclusion.
- Produce a validation result.
- Fail closed when required evidence is missing, inconsistent, or invalid.

The validation workflow and "ci/validate_evidence.py" must implement the same command-line contract. Manual validation must explicitly identify the source run and attempt.

Evidence states

State| Meaning
"PARSE_VALID"| YAML parsing succeeded.
"STRUCTURE_VALID"| The workflow structure satisfies the checks performed.
"EVIDENCE_LIMITED"| Some evidence exists, but required checks remain incomplete.
"NOT_VERIFIED"| Verification failed or could not be completed.
"VERIFIED"| All defined checks passed for the explicitly stated claim and evidence scope.

These labels are project-level states. They must not be interpreted as GitHub certifications or external regulatory approvals.

Run the workflow

Install and authenticate GitHub CLI:

gh auth login

Confirm that the workflow exists in the target repository:

gh workflow list \
  --repo cyrillofrancisco30-lgtm/actions-workflow-samples

Dispatch the workflow only after it has been committed to the target repository:

gh workflow run homologation-runner.yml \
  --repo cyrillofrancisco30-lgtm/actions-workflow-samples \
  --ref master \
  -f ruid="manual-$(date +%s)"

List recent runs:

gh run list \
  --repo cyrillofrancisco30-lgtm/actions-workflow-samples \
  --limit 10

Watch a specific run using its actual ID:

gh run watch RUN_ID \
  --repo cyrillofrancisco30-lgtm/actions-workflow-samples \
  --exit-status

Replace "RUN_ID" with the ID returned by GitHub CLI. Confirm the repository's actual default branch before using "master".

Verification requirements

Before treating an execution as verified, inspect:

1. The exact workflow run ID and attempt.
2. The source commit SHA and reference.
3. The workflow conclusion and individual step logs.
4. The artifact associated with that exact run and attempt.
5. The manifest fields and validator output.
6. The test results and any failed or skipped checks.

A commit or push does not prove that an API or workflow executed. A successful workflow does not prove semantic correctness. A matching hash does not establish the truth of the underlying claim.

Scope limitations

Unless separately implemented and evidenced, this pipeline does not establish:

- ARM-TTK execution or conformance.
- Azure deployment success or external service state.
- XA Banking Ledger GT7 reconciliation.
- Regulatory approval or certification.
- The semantic truth of external financial claims.

Each additional claim requires its own evidence and validation procedure.

Configuration references

- "GitHub Actions documentation" (https://docs.github.com/en/actions)
- "Workflow syntax" (https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- "Artifact actions" (https://github.com/actions/upload-artifact)
- "Artifact download action" (https://github.com/actions/download-artifact)
- "GitHub CLI manual" (https://cli.github.com/manual/)

Current verification status

Do not claim that the workflows have passed GitHub-hosted execution until an actual run has been inspected and its artifacts and logs have been reviewed.

Update this section only with reproducible run IDs, commit SHAs, test outcomes, and validation results.


 Homologation runner

on:
  workflow_dispatch:
    inputs:
      ruid:
        description: Run unique ID
        required: true
        type: string
  push:
    branches:
      - master

permissions:
  contents: read

jobs:
  homologation:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        id: install
        continue-on-error: true
        run: |
          python -m pip install -r ci/requirements.txt

      - name: Run tests
        id: tests
        if: steps.install.outcome == 'success'
        continue-on-error: true
        run: |
          python -m pytest -q

      - name: Generate evidence manifest
        if: always()
        env:
          RUID: ${{ inputs.ruid || github.run_id }}
          INSTALL_OUTCOME: ${{ steps.install.outcome }}
          TEST_OUTCOME: ${{ steps.tests.outcome }}
        run: |
          mkdir -p evidence

          python - <<'PY'
          import json
          import os
          from datetime import datetime, timezone

          install = os.environ.get("INSTALL_OUTCOME", "not_run")
          tests = os.environ.get("TEST_OUTCOME", "not_run")

          status = (
              "EXECUTION_PASSED"
              if install == "success" and tests == "success"
              else "EXECUTION_FAILED"
          )

          manifest = {
              "schema_version": "1.0.0",
              "ruid": os.environ["RUID"],
              "run_id": os.environ["GITHUB_RUN_ID"],
              "run_number": os.environ["GITHUB_RUN_NUMBER"],
              "run_attempt": os.environ["GITHUB_RUN_ATTEMPT"],
              "workflow": os.environ["GITHUB_WORKFLOW"],
              "repository": os.environ["GITHUB_REPOSITORY"],
              "sha": os.environ["GITHUB_SHA"],
              "ref": os.environ["GITHUB_REF"],
              "created_at_utc": datetime.now(timezone.utc).isoformat(),
              "install_outcome": install,
              "tests_outcome": tests,
              "status": status
          }

          with open(
              "evidence/manifest.json", "w", encoding="utf-8"
          ) as f:
              json.dump(manifest, f, indent=2)
          PY

      - name: Upload evidence artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: evidence-${{ github.run_id }}-${{ github.run_attempt }}
          path: evidence/
          if-no-files-found: error
          retention-days: 30

      - name: Enforce execution result
        if: always()
        env:
          INSTALL_OUTCOME: ${{ steps.install.outcome }}
          TEST_OUTCOME: ${{ steps.tests.outcome }}
        run: |
          if [ "$INSTALL_OUTCOME" != "success" ]; then
            echo "::error::Dependency installation did not succeed."
            exit 1
          fi

          if [ "$TEST_OUTCOME" != "success" ]; then
            echo "::error::Tests did not succeed."
            exit 1
          fi


Homologation runner

on:
  workflow_dispatch:
    inputs:
      ruid:
        description: Run unique ID
        required: true
  push:
    branches:
      - master

permissions:
  contents: read

jobs:
  homologation:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r ci/requirements.txt

      - name: Generate evidence manifest
        env:
          RUID: ${{ github.event_name == 'workflow_dispatch' && inputs.ruid || github.run_id }}
        run: |
          mkdir -p evidence
          python - <<'PY'
          import json, os
          from datetime import datetime, timezone

          manifest = {
              "schema_version": "1.0.0",
              "ruid": os.environ.get("RUID", ""),
              "run_id": os.environ.get("GITHUB_RUN_ID", ""),
              "run_number": os.environ.get("GITHUB_RUN_NUMBER", ""),
              "workflow": os.environ.get("GITHUB_WORKFLOW", ""),
              "repository": os.environ.get("GITHUB_REPOSITORY", ""),
              "sha": os.environ.get("GITHUB_SHA", ""),
              "ref": os.environ.get("GITHUB_REF", ""),
              "created_at_utc": datetime.now(timezone.utc).isoformat(),
              "status": "generated"
          }

          with open("evidence/manifest.json", "w", encoding="utf-8") as f:
              json.dump(manifest, f, indent=2)
          PY

      - name: Upload evidence artifact
        uses: actions/upload-artifact@v4
        with:
          name: evidence
          path: evidence/
          if-no-files-found: error
          retention-days: 30
          


          name: Evidence validation

on:
  workflow_dispatch:
    inputs:
      ruid:
        description: Run unique ID
        required: true
  workflow_run:
    workflows: ["Homologation runner"]
    types: [completed]

permissions:
  contents: read
  actions: read
  id-token: write

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r ci/requirements.txt

      - name: Download evidence artifact from triggering workflow
        if: ${{ github.event_name == 'workflow_run' }}
        uses: actions/download-artifact@v4
        with:
          name: evidence
          path: evidence
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Validate evidence
        env:
          RUID: ${{ github.event_name == 'workflow_dispatch' && inputs.ruid || github.event.workflow_run.id }}
          EXPECTED_COMMIT: ${{ github.event_name == 'workflow_dispatch' && github.sha || github.event.workflow_run.head_sha }}
        run: |
          echo "RUID=$RUID"
          echo "EXPECTED_COMMIT=$EXPECTED_COMMIT"

          if [ ! -f evidence/manifest.json ]; then
            echo "manifest.json not found in evidence/"
            ls -R evidence || true
            exit 1
          fi

          python -m ci.validate_evidence \
            --ruid "$RUID" \
            --expected-commit "$EXPECTED_COMMIT" \
            --manifest evidence/manifest.json \
            --output evidence/validated-graph.json


            
          
          ## Starter Action Workflows to deploy to Azure

Você precisa separar em dois arquivos distintos dentro de `.github/workflows/`:

- `.github/workflows/homologation-runner.yml`
- `.github/workflows/evidence-validation.yml`

O bloco que você colou tem:
- dois `name:`
- dois `on:`
- tudo em um único arquivo
- isso não funciona em GitHub Actions

Estrutura correta

Arquivo 1: `.github/workflows/homologation-runner.yml`

```yaml
name: Homologation runner

on:
  workflow_dispatch:
    inputs:
      ruid:
        description: Run unique ID
        required: true
  push:
    branches:
      - master

permissions:
  contents: read

jobs:
  homologation:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r ci/requirements.txt

      - name: Generate evidence manifest
        env:
          RUID: ${{ github.event_name == 'workflow_dispatch' && inputs.ruid || github.run_id }}
        run: |
          mkdir -p evidence
          python - <<'PY'
          import json, os
          from datetime import datetime, timezone

          manifest = {
              "schema_version": "1.0.0",
              "ruid": os.environ.get("RUID", ""),
              "run_id": os.environ.get("GITHUB_RUN_ID", ""),
              "run_number": os.environ.get("GITHUB_RUN_NUMBER", ""),
              "workflow": os.environ.get("GITHUB_WORKFLOW", ""),
              "repository": os.environ.get("GITHUB_REPOSITORY", ""),
              "sha": os.environ.get("GITHUB_SHA", ""),
              "ref": os.environ.get("GITHUB_REF", ""),
              "created_at_utc": datetime.now(timezone.utc).isoformat(),
              "status": "generated"
          }

          with open("evidence/manifest.json", "w", encoding="utf-8") as f:
              json.dump(manifest, f, indent=2)
          PY

      - name: Upload evidence artifact
        uses: actions/upload-artifact@v4
        with:
          name: evidence
          path: evidence/
          if-no-files-found: error
          retention-days: 30
```

Arquivo 2: `.github/workflows/evidence-validation.yml`

```yaml
name: Evidence validation

on:
  workflow_dispatch:
    inputs:
      ruid:
        description: Run unique ID
        required: true
  workflow_run:
    workflows: ["Homologation runner"]
    types: [completed]

permissions:
  contents: read
  actions: read
  id-token: write

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r ci/requirements.txt

      - name: Download evidence artifact from triggering workflow
        if: ${{ github.event_name == 'workflow_run' }}
        uses: actions/download-artifact@v4
        with:
          name: evidence
          path: evidence
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Validate evidence
        env:
          RUID: ${{ github.event_name == 'workflow_dispatch' && inputs.ruid || github.event.workflow_run.id }}
          EXPECTED_COMMIT: ${{ github.event_name == 'workflow_dispatch' && github.sha || github.event.workflow_run.head_sha }}
        run: |
          echo "RUID=$RUID"
          echo "EXPECTED_COMMIT=$EXPECTED_COMMIT"

          if [ ! -f evidence/manifest.json ]; then
            echo "manifest.json not found in evidence/"
            ls -R evidence || true
            exit 1
          fi

          python -m ci.validate_evidence \
            --ruid "$RUID" \
            --expected-commit "$EXPECTED_COMMIT" \
            --manifest evidence/manifest.json \
            --output evidence/validated-graph.json
```

Comandos de disparo corretos

Para arm-ttk:
```bash
gh auth login

gh workflow run homologation-runner.yml \
  --repo cyrillofrancisco30-lgtm/arm-ttk \
  --ref master \
  -f ruid="manual-$(date +%s)"

gh run list --repo cyrillofrancisco30-lgtm/arm-ttk --limit 10
gh run watch --repo cyrillofrancisco30-lgtm/arm-ttk
```

Para Xa-Finance-AI-:
```bash
gh auth login

gh workflow run homologation-runner.yml \
  --repo cyrillofrancisco30-lgtm/Xa-Finance-AI- \
  --ref main \
  -f ruid="manual-$(date +%s)"

gh run list --repo cyrillofrancisco30-lgtm/Xa-Finance-AI- --limit 10
gh run watch --repo cyrillofrancisco30-lgtm/Xa-Finance-AI-
```

Importante:
- O workflow de validação dispara automaticamente após o `Homologation runner` terminar.
- O `evidence` precisa ser o nome exato do artefato publicado pelo workflow upstream.
- Isso valida o pipeline e a geração de evidência operacional, não o estado real do XA Banking Ledger GT7.



[GitHub Actions for Azure](https://github.com/Azure/actions) help you easily create workflows to build, test, package, release and deploy to Azure, following a push or pull request.

You use Azure starter templates present in this repo to easily create GitHub CI/CD workflows targeting Azure, to deploy your apps created with popular languages and frameworks such as .NET, Node.js, Java, PHP, Ruby or Python, in containers or running on any operating system.

## Guidelines to select/author a new sample workflow

**Folder Structure:**
These workflow samples to automate your deployment workflows targeting various Azure services are organized under folders of same names. For example: `/AppService/asp.net-core-webapp-on-azure.yml`

- [**/AppService** ](https://github.com/Azure/actions-workflow-samples/tree/master/AppService) Samples to configure and deploy web applications that scale with your business, to [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/web/)

- [**/AzureCLI**](https://github.com/Azure/actions-workflow-samples/tree/master/AzureCLI) Samples to run Azure CLI scripts to provision and manage Azure resources from a GitHub Action workflow

- [**/ARM**](https://github.com/Azure/actions-workflow-samples/tree/master/ARM) Samples to deploy [Azure Resource Manager templates](https://docs.microsoft.com/bs-latn-ba/azure/azure-resource-manager/templates/)

- [**/AzurePipelines**](https://github.com/Azure/actions-workflow-samples/tree/master/AzurePipelines) Samples to trigger a CD run in Azure Pipelines from a GitHub Action workflow

- [**/Database**](https://github.com/Azure/actions-workflow-samples/tree/master/Database) Samples to deploy to a database on Azure, [Azure SQl database](https://azure.microsoft.com/en-us/services/sql-database/) or [Azure MySQL database](https://azure.microsoft.com/en-us/services/mysql/)

- [**/FunctionApp**](https://github.com/Azure/actions-workflow-samples/tree/master/FunctionApp) Samples to build and deploy serverless apps to [Azure Functions](https://azure.microsoft.com/en-us/services/functions/)

- [**/Kubernetes**](https://github.com/Azure/actions-workflow-samples/tree/master/Kubernetes) Samples to deploy to any Kubernetes cluster on-premise or any cloud including [Azure Kubernetes service](https://azure.microsoft.com/en-us/services/kubernetes-service/)

- [**/MachineLearning**](https://github.com/Azure/actions-workflow-samples/tree/master/MachineLearning) Samples to build and deploy machine learning models using [Azure Machine Learning](https://docs.microsoft.com/en-us/azure/machine-learning/)

- [**/Terraform**](https://github.com/Azure/actions-workflow-samples/tree/master/Terraform) Samples to deploy infrastructure to an Azure subscription using [Terraform Azure Provider](https://www.terraform.io/docs/providers/azurerm/index.html)

- [**/AzurePolicy**](https://github.com/Azure/actions-workflow-samples/tree/master/AzurePolicy) Samples to trigger on-demand Azure Policy compliance scans from a GitHub Action workflow

- [**/End-to-End/Serverless**](/End-to-End/Serverless-web-application) Sample to deploy and manage the lifecycle of a [serverless web application](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/serverless/web-app). The application serves static content from Azure Blob Storage, and implements an API using Azure Functions which is exposed through API Management. The API reads data from Cosmos DB and returns the results to the web app.

**Naming Notation:**
* `os-ecosystem-ServiceName-on-azure`: example, linux-container-functionapp-on-azure.yml
* OS in the name is optional if the action workflow sample is OS agnostic and doesnt significantly change between OS (Linux/Windows) 
* Ecosystem can be a language (.NET, Nodejs, java, Python, Ruby etc.) or Docker/Container Or Database flavours like SQL/MySQL etc.

**Workflow structure**
* Include 'name' for every workflow to indicate the purpose of the workflow
* Ensure that starter workflows run on: push by default.  
* For all secrets to be defined in the workflow, use UPPER_CASE with underscore delimiters instead of snake_case or camelCase.
* Include a commented **Configuration section** which includes hyperlinks to documentation for the Actions used and other pre-reqs.
* Define environment variables as part of configuration.  We think this will help provide visibility into the things that need to be configured as part of te workflow.
* Ensure all Azure actions referenced in the workflow are pointing to a released version of the action and not from the master. For list of all released GitHub actions for Azure, please refer to https://github.com/Azure/actions

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
