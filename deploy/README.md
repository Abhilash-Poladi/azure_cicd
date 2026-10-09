# Azure Data Factory deployment

The workflow at `.github/workflows/deploy-adf.yml` responds automatically to
pushes to `adf_publish`, `uat`, and `prod`. It deploys the inline
`appogit/ARMTemplateForFactory.json` from an exact commit in `adf_publish`.
The generated template contains the pipeline, dataset, and linked service in a
single file, so this workflow does not need to upload or publish linked
templates.

GitHub loads a push-triggered workflow from the branch receiving the push. For
that reason, merge this workflow and the `deploy/parameters` files into each
deployment branch: `adf_publish`, `uat`, and `prod`. They are deployment
configuration only; no ARM templates need to be merged into `main`.

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

Edit `deploy/parameters/dev.json`, `uat.json`, and `prod.json` in the deployment
configuration change with each environment's factory name and ADLS Gen2 URL.
The DEV values reflect the currently published ADF parameters. UAT and PROD
values are intentionally marked `REPLACE_WITH_...`; the workflow rejects
those markers until replaced. Do not put keys, passwords, or tokens in these
files. The target Data Factory must already exist because the generated
template deploys factory child resources, not the factory itself.

Configure GitHub Environment required reviewers for `uat` and `prod` if you
want approvals before those automated deployments proceed. Restrict each
environment's allowed deployment branches as appropriate for your repository.

## Automatic deployment flow

1. Publishing/merging generated ARM templates to `adf_publish` automatically
   deploys that exact `adf_publish` commit to DEV.
2. Merge the published artifact branch into `uat`. The workflow finds the
   `adf_publish` commit in that merge's ancestry and deploys that exact commit
   to UAT.
3. Merge the promoted `uat` branch into `prod`. The workflow resolves the same
   `adf_publish` artifact commit from the merge ancestry and deploys it to PROD.

Each promotion refuses to proceed unless that exact artifact has a successful
deployment record in the previous environment. Merge the desired published
artifact into `uat` after its DEV deployment succeeds, then merge the promoted
`uat` branch into `prod` after the same artifact's UAT deployment succeeds.
The workflow resolves the `adf_publish` ancestor of the merge and checks out
that exact commit, rather than resolving the latest branch head at deployment
time.
