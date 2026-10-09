# Azure Data Factory deployment

The workflow at `.github/workflows/deploy-adf.yml` responds automatically to
pushes to `adf_publish`, `uat`, and `prod`. It deploys the inline
`appogit/ARMTemplateForFactory.json` from an exact commit in `adf_publish`.
The generated template contains the pipeline, dataset, and linked service in a
single file, so this workflow does not need to upload or publish linked
templates.

The workflow and its parameter files are checked out from `main`; that branch
contains deployment configuration only, not the ADF ARM template. The artifact
is always checked out separately from `adf_publish`.

## Configure environments

Create GitHub Environments named `dev`, `uat`, and `prod`. Configure each with
these variables:

| Variable | Purpose |
| --- | --- |
| `AZURE_CLIENT_ID` | Azure federated application/client ID |
| `AZURE_TENANT_ID` | Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription containing the target factory |
| `ADF_RESOURCE_GROUP` | Resource group containing the existing Data Factory |

Configure `ADF_STORAGE_ACCOUNT_KEY` as a secret in each environment. It is
passed only as the ARM template's secure `ls_dev_accountKey` parameter via a
temporary runner file that is removed when the deployment step exits; it is
not stored in the repository. Set up an Azure federated credential for each
environment with subject
`repo:Abhilash-Poladi/azure_cicd:environment:<environment-name>` and grant the
identity only the permissions needed to deploy the factory resources to that
environment's resource group.

Edit `deploy/parameters/dev.json`, `uat.json`, and `prod.json` with each
environment's factory name and ADLS Gen2 URL. The DEV values reflect the
currently published ADF parameters. UAT and PROD values are intentionally
marked `REPLACE_WITH_...`; the workflow rejects those markers until replaced.
Do not put keys, passwords, or tokens in these files. The target Data Factory
must already exist because the generated template deploys factory child
resources, not the factory itself.

Configure GitHub Environment required reviewers for `uat` and `prod` if you
want approvals before those automated deployments proceed. Restrict each
environment's allowed deployment branches as appropriate for your repository.

## Automatic deployment flow

1. Publishing/merging generated ARM templates to `adf_publish` automatically
   deploys that exact `adf_publish` commit to DEV.
2. Pushing/merging to `uat` promotes the latest successful DEV deployment's
   exact artifact SHA to UAT.
3. Pushing/merging to `prod` promotes the latest successful UAT deployment's
   exact artifact SHA to PROD.

Each promotion refuses to proceed unless the latest deployment in the previous
environment succeeded. It verifies the selected commit is still in
`adf_publish` history and checks out that exact commit, rather than resolving
the latest branch head at deployment time. Merge to `uat` after the desired
artifact has completed its DEV deployment; merge to `prod` after the UAT
deployment of that artifact succeeds.
