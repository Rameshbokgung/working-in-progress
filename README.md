# working-in-progress


We'll structure this into two jobs:

build-and-publish-artifact: This job will build your Function App, create the versioned zip, calculate its checksum, and publish it to a GitHub Release. It will also upload the published-app folder as a workflow artifact, making it available to subsequent jobs in the same workflow run.
deploy-to-azure: This job will download the published-app workflow artifact, log into Azure, deploy the Function App, and perform the verification steps, including the optional SCM_COMMIT_ID check.
Update Your Workflow File (.github/workflows/ci-cd-function-app.yml)



Key Changes and How it Works:

Two Separate Jobs (build-and-publish-artifact and deploy-to-azure):

This provides clear separation of concerns: one job builds and formally publishes the artifact, the other consumes it and deploys.
needs: build-and-publish-artifact in the deploy-to-azure job ensures it only runs after the first job successfully completes.
Upload Deployment Artifact for Azure Step:

uses: actions/upload-artifact@v4: This action takes the published-app directory (which is ready for Azure deployment) and uploads it as a temporary "workflow artifact." This is different from GitHub Releases.
It gives the artifact a name including the project name and version, making it unique.
retention-days: Defines how long GitHub keeps this workflow artifact.
Download Deployment Artifact for Azure Step:

uses: actions/download-artifact@v4: In the deploy-to-azure job, this action downloads the artifact previously uploaded.
name: Crucially, the name here must exactly match the name used in the upload-artifact step. We use the outputs from the previous job (needs.build-and-publish-artifact.outputs.artifact_version) to dynamically get the correct artifact name.
path: ./downloaded-app: It downloads the contents into a new directory named downloaded-app.
Deploy to Azure Function App Step:

package: './downloaded-app': Now, the azure/functions-action points to the downloaded-app directory, ensuring it deploys the artifact that was prepared and verified in the previous job.
Post-Deployment Verification: Check SCM_COMMIT_ID:

This new step uses the Azure CLI (az functionapp config appsettings list) to retrieve the SCM_COMMIT_ID environment variable from your deployed Function App. This ID often reflects the Git commit hash of the deployed code.
It then compares this DEPLOYED_COMMIT_ID with ${{ github.sha }}, which is the Git commit hash of the code that triggered this specific workflow run.
This gives you that final sanity check to ensure the deployed code matches what you expect from your source control.
Before You Run:

Review env variables: Double-check FUNCTION_APP_PROJECT_PATH, FUNCTION_APP_PROJECT_NAME, and DOTNET_VERSION.
Update Function Endpoint: In the Verify Deployment (HTTP) step, change /api/YourFunctionEndpoint to your actual HTTP-triggered function's route.
GitHub Secrets: Ensure all AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID, AZURE_FUNCTIONAPP_NAME, and AZURE_RESOURCES_GROUP secrets are correctly configured in your GitHub repository.
GitHub Environments (Recommended): Consider setting up a "Production" environment in your GitHub repository settings (under Settings > Environments). You can then add protection rules (like manual approval) to the deploy-to-azure job's environment, giving you more control over production deployments.
