# Azure Data Factory deployment

The workflow at `.github/workflows/deploy-adf.yml` deploys the inline
`appogit/ARMTemplateForFactory.json` from an exact commit in `adf_publish`.
The generated template contains the pipeline, dataset, and linked service in a
single file, so this workflow does not need to upload or publish linked
templates.

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

For UAT and PROD, configure GitHub Environment required reviewers so the
environment approval gates deployment. Restrict each environment's allowed
deployment branches as appropriate for your repository.

## Promote a published version

1. Dispatch **Deploy Azure Data Factory** with target `dev` and the full
   40-character SHA of a commit in `adf_publish`.
2. Validate the deployment in DEV.
3. Dispatch the workflow again with the **same SHA** and target `uat` or `prod`.
   Set **confirm_dev_validation** only after completing those DEV checks. The
   workflow requires both that confirmation and a latest successful DEV
   deployment record for the exact SHA; UAT/PROD Environment reviewers provide
   the approval gate.

The workflow verifies the selected commit is still in `adf_publish` history
and checks out that commit, rather than resolving the latest branch head at
deployment time. It is intentionally manual: merging the collaboration,
`uat`, or `main` branches does not select or deploy an artifact automatically.
Run the workflow from the protected default branch; the target GitHub
Environment, not the workflow-source branch, selects the Azure destination.
