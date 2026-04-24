Here is the complete integration for the GitHub Deploy Integration for your existing Node.js + Express web application into the Google ecosystem, ensuring operational stability and comprehensive monitoring.

---

### **Integration Overview**

This integration will establish a robust Continuous Deployment (CD) pipeline for your Node.js + Express application from a GitHub repository to Google Cloud Run. Google Cloud Run provides a fully managed environment for stateless containers, offering scalability and ease of deployment.

For operational stability and auditability, we will integrate Google Sheets for deployment logging and Google Cloud Monitoring for real-time application health and performance tracking. Google Apps Script will act as a bridge to log deployment events from GitHub Actions into Google Sheets.

**Key Components:**
*   **Google Cloud Run:** Serverless platform for hosting the Node.js application.
*   **GitHub Actions:** Automates the build and deployment process on every push to the `main` branch.
*   **Google Cloud Build:** Used by GitHub Actions to build the Docker image for Cloud Run.
*   **Google Apps Script:** Webhook endpoint to receive deployment notifications.
*   **Google Sheets:** Centralized log for all application deployments.
*   **Google Cloud Monitoring:** Comprehensive platform for tracking application metrics, logs, and setting up alerts.
*   **Google Cloud Logging:** Centralized log management for the Cloud Run service.

**Database Note (Crucial for `better-sqlite3`):**
Your application uses `better-sqlite3` with `relay.db`. Google Cloud Run instances have an ephemeral filesystem, meaning any data written to the local disk (like `relay.db`) will be lost when the instance scales down, restarts, or a new revision is deployed. This is *not* suitable for persistent data storage in a production environment.

**Recommendation:** For true operational stability, migrate your database to a persistent, managed solution like **Google Cloud SQL (PostgreSQL or MySQL)** or **Firestore/Datastore**. This integration focuses on deploying your *application code* and its CI/CD. While `better-sqlite3` will technically run in the container, it will lose all data. I will provide the `Dockerfile` for the current setup but strongly advise addressing the database persistence issue.

---

### **Prerequisites**

Before starting, ensure you have the following:

1.  **Google Cloud Project:** An active GCP project.
2.  **GCP Billing Account:** Enabled for your GCP project.
3.  **`gcloud` CLI:** Installed and configured on your local machine, authenticated to your GCP project.
4.  **GitHub Repository:** Your existing Node.js + Express application code pushed to a GitHub repository.
5.  **Node.js Application:** Ensure your Node.js application is configured to listen on the port specified by the `PORT` environment variable (e.g., `process.env.PORT || 5000`).

---

### **Step-by-Step Integration & Deployment**

#### 1. Google Cloud Project Setup

1.  **Select/Create Project:**
    ```bash
    gcloud projects create YOUR_GCP_PROJECT_ID --name="Three AI Relay App"
    gcloud config set project YOUR_GCP_PROJECT_ID
    ```
    (Replace `YOUR_GCP_PROJECT_ID` with your desired project ID). If you have an existing project, just set it.

2.  **Enable Required APIs:**
    ```bash
    gcloud services enable run.googleapis.com \
        cloudbuild.googleapis.com \
        logging.googleapis.com \
        monitoring.googleapis.com \
        oauth2.googleapis.com \
        script.googleapis.com # For Apps Script to work with Sheets
    ```

3.  **Create a Service Account for GitHub Actions:**
    This service account will be used by GitHub Actions to authenticate and deploy to Cloud Run.
    ```bash
    SERVICE_ACCOUNT_NAME="github-actions-deployer"
    SERVICE_ACCOUNT_EMAIL="${SERVICE_ACCOUNT_NAME}@${YOUR_GCP_PROJECT_ID}.iam.gserviceaccount.com"

    gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME} \
        --display-name="GitHub Actions Cloud Run Deployer"

    # Grant necessary roles for Cloud Build and Cloud Run deployment
    gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT_ID \
        --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
        --role="roles/run.admin" # For deploying and managing Cloud Run services
    gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT_ID \
        --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
        --role="roles/iam.serviceAccountUser" # To act as other service accounts if needed (e.g., Cloud Run runtime SA)
    gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT_ID \
        --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
        --role="roles/storage.objectViewer" # To pull images from Container Registry (Cloud Build default)
    gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT_ID \
        --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
        --role="roles/cloudbuild.builds.editor" # For Cloud Build to build images
    ```

4.  **Configure Workload Identity Federation for GitHub Actions:**
    This is the most secure way to authenticate GitHub Actions to GCP without long-lived service account keys.
    ```bash
    # Create the Workload Identity Pool
    gcloud iam workload-identity-pools create "github-pool" \
        --project="YOUR_GCP_PROJECT_ID" \
        --location="global" \
        --display-name="GitHub Actions Pool"

    # Create a provider for your GitHub repository
    gcloud iam workload-identity-pools providers create-oidc "github-provider" \
        --project="YOUR_GCP_PROJECT_ID" \
        --location="global" \
        --workload-identity-pool="github-pool" \
        --display-name="GitHub OIDC Provider" \
        --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
        --issuer-uri="https://token.actions.githubusercontent.com"

    # Grant the GitHub Actions service account permission to impersonate the deployer SA
    # Replace <YOUR_GITHUB_ORG> and <YOUR_REPO_NAME>
    # Example: attribute.repository_owner=my-org attribute.repository=my-repo
    # For a specific branch, add `and attribute.ref == "refs/heads/main"`
    gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT_EMAIL}" \
        --project="YOUR_GCP_PROJECT_ID" \
        --role="roles/iam.workloadIdentityUser" \
        --member="principalSet://iam.googleapis.com/projects/$(gcloud projects describe YOUR_GCP_PROJECT_ID --format='value(projectNumber)')/locations/global/workloadIdentityPools/github-pool/attribute.repository/YOUR_GITHUB_ORG/YOUR_REPO_NAME"
    ```
    **Note:** Make sure to replace `YOUR_GCP_PROJECT_ID`, `YOUR_GITHUB_ORG`, and `YOUR_REPO_NAME` with your actual values.

#### 2. GitHub Repository Preparation

Ensure your Node.js application is in a GitHub repository. You will need to add a `Dockerfile` and the GitHub Actions workflow file.

#### 3. Application Containerization (`Dockerfile`)

Create a `Dockerfile` in the root of your Node.js application.

```dockerfile
# Use a Node.js LTS image as the base
FROM node:20-slim

# Create and change to the app directory
WORKDIR /app

# Copy package.json and package-lock.json and install dependencies
# This step is cached and only re-runs if package.json or package-lock.json change
COPY package*.json ./
RUN npm install --production

# Copy the rest of your application source code
COPY . .

# Expose the port your app listens on. Cloud Run will inject PORT env var.
# Ensure your app listens on process.env.PORT
ENV PORT 8080
EXPOSE ${PORT}

# Command to run the application
CMD ["npm", "start"]
```
**Important:** Ensure your `package.json` has a `start` script, e.g.:
```json
{
  "name": "three-ai-relay-app",
  "version": "1.0.0",
  "description": "The Three-AI Relay Web Application",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "better-sqlite3": "^X.Y.Z",
    "express": "^X.Y.Z"
  }
}
```
And your `app.js` (or main entry file) uses `process.env.PORT`:
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 5000; // Use port 5000 as fallback if PORT is not set

// ... your application logic ...

app.listen(port, () => {
  console.log(`Three-AI Relay App listening on port ${port}`);
});
```

#### 4. Google Apps Script for Deployment Logging

This script will act as a simple webhook to receive deployment notifications from GitHub Actions and log them into a Google Sheet.

1.  **Create a New Google Sheet:** Go to `sheets.google.com` and create a new blank spreadsheet. Name it `Three-AI Relay App Deployment Log`.
2.  **Set up Headers:** In the first row, add the following headers:
    `Timestamp`, `Git Commit Hash`, `Deployed By`, `Deployment Status`, `Cloud Run Service URL`, `Cloud Run Revision ID`, `GitHub Workflow Run URL`
3.  **Open Apps Script Editor:** Go to `Extensions > Apps Script`. This will open a new Apps Script project.
4.  **Paste the Code:** Replace any existing code (`Code.gs`) with the following:

    ```javascript
    /**
     * @fileoverview Google Apps Script for logging GitHub Actions deployments to Google Sheets.
     * @author Gem Bot
     */

    // --- Configuration ---
    const SHEET_NAME = "Three-AI Relay App Deployment Log"; // Name of your Google Sheet tab
    // ---------------------

    /**
     * Handles POST requests, acting as a webhook receiver for deployment events.
     * Expected JSON payload from GitHub Actions:
     * {
     *   "commitHash": "a1b2c3d4e5f6...",
     *   "deployedBy": "github-actions[bot]",
     *   "status": "SUCCESS", // or "FAILURE"
     *   "cloudRunServiceUrl": "https://your-service-url.run.app",
     *   "cloudRunRevisionId": "my-service-00001-abc",
     *   "githubWorkflowRunUrl": "https://github.com/org/repo/actions/runs/12345"
     * }
     * @param {Object} e The event object containing request parameters.
     * @return {GoogleAppsScript.Content.TextOutput} A JSON response.
     */
    function doPost(e) {
      const lock = LockService.getScriptLock();
      // Wait for up to 30 seconds for the lock.
      try {
        lock.waitLock(30000);
      } catch (e) {
        return ContentService.createTextOutput(JSON.stringify({ status: "ERROR", message: "Could not acquire lock." }))
          .setMimeType(ContentService.MimeType.JSON);
      }

      try {
        const requestBody = JSON.parse(e.postData.contents);
        const {
          commitHash,
          deployedBy,
          status,
          cloudRunServiceUrl,
          cloudRunRevisionId,
          githubWorkflowRunUrl
        } = requestBody;

        const timestamp = new Date().toLocaleString();
        const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);

        if (!sheet) {
          throw new Error(`Sheet with name "${SHEET_NAME}" not found.`);
        }

        sheet.appendRow([
          timestamp,
          commitHash,
          deployedBy,
          status,
          cloudRunServiceUrl,
          cloudRunRevisionId,
          githubWorkflowRunUrl
        ]);

        lock.releaseLock(); // Release the lock before returning
        return ContentService.createTextOutput(JSON.stringify({ status: "SUCCESS", message: "Deployment logged successfully." }))
          .setMimeType(ContentService.MimeType.JSON);

      } catch (error) {
        Logger.log("Error processing webhook: " + error.message);
        if (lock.hasLock()) {
          lock.releaseLock();
        }
        return ContentService.createTextOutput(JSON.stringify({ status: "ERROR", message: "Failed to log deployment.", error: error.message }))
          .setMimeType(ContentService.MimeType.JSON);
      }
    }

    /**
     * (Optional) Function to run manually for initial setup or debugging.
     * No-op for this specific integration, but good practice for other Apps Script projects.
     */
    function onOpen() {
      // You can add a custom menu here if needed
    }
    ```

5.  **Save the Script:** Click the floppy disk icon or `File > Save project`.
6.  **Deploy as Web App:**
    *   Click `Deploy > New deployment`.
    *   For "Select type," choose `Web app`.
    *   **Description:** `Deployment Logger Webhook`
    *   **Execute as:** `Me (your Google account email)`
    *   **Who has access:** `Anyone` (This is crucial for GitHub Actions to be able to send requests. The endpoint is obscure enough, but consider IP restrictions if possible in a highly sensitive scenario.)
    *   Click `Deploy`.
    *   You will be asked to authorize the script. Follow the prompts. It will request access to your Google Sheet.
    *   Once deployed, copy the **Web app URL**. This is your `APPS_SCRIPT_WEBHOOK_URL` and will be used in your GitHub Actions workflow.

#### 5. Google Sheets Setup

As described in step 4, ensure your Google Sheet `Three-AI Relay App Deployment Log` has the exact headers in the first row:
`Timestamp`, `Git Commit Hash`, `Deployed By`, `Deployment Status`, `Cloud Run Service URL`, `Cloud Run Revision ID`, `GitHub Workflow Run URL`

#### 6. GitHub Actions Workflow for CI/CD

Create a file named `.github/workflows/deploy.yml` in your GitHub repository.

```yaml
name: Deploy Three-AI Relay App to Google Cloud Run

on:
  push:
    branches:
      - main # Trigger on pushes to the main branch

env:
  PROJECT_ID: YOUR_GCP_PROJECT_ID # Replace with your GCP project ID
  SERVICE_NAME: three-ai-relay-app # Desired name for your Cloud Run service
  REGION: us-central1 # Desired GCP region (e.g., us-central1, europe-west1)
  APPS_SCRIPT_WEBHOOK_URL: YOUR_APPS_SCRIPT_WEBHOOK_URL # The URL obtained from Apps Script deployment

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write' # Required for Workload Identity Federation

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        id: 'auth'
        uses: 'google-github-actions/auth@v2'
        with:
          workload_identity_provider: 'projects/$(gcloud projects describe YOUR_GCP_PROJECT_ID --format="value(projectNumber)")/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'github-actions-deployer@YOUR_GCP_PROJECT_ID.iam.gserviceaccount.com'
          # Replace YOUR_GCP_PROJECT_ID, github-pool, github-provider with your actual values

      - name: Set up Cloud SDK (gcloud)
        uses: google-github-actions/setup-gcloud@v2

      - name: Build and Push Docker Image to Google Container Registry
        run: |
          gcloud builds submit --tag gcr.io/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:${{ github.sha }} .
        # The tag uses the commit SHA for unique versioning

      - name: Deploy to Google Cloud Run
        id: deploy
        uses: 'google-github-actions/deploy-cloudrun@v2'
        with:
          service: ${{ env.SERVICE_NAME }}
          region: ${{ env.REGION }}
          image: gcr.io/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:${{ github.sha }}
          # Configure resources and environment variables as needed
          # For better-sqlite3, you *must* specify a CPU minimum to prevent instances from shutting down
          # and losing the database state, but this is still NOT persistent storage.
          # For production, integrate with Cloud SQL.
          flags: --cpu=1 --memory=512Mi --min-instances=1 --max-instances=5 --allow-unauthenticated # Example flags
          env_vars: |
            NODE_ENV=production
            # Add other env vars required by your app, e.g., DATABASE_URL if using Cloud SQL
            # For sqlite, no specific env vars are needed for the db itself.
        # The output of this step will contain the service URL and revision ID

      - name: Get Cloud Run Service URL and Revision ID
        id: get_url_revision
        run: |
          SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --platform managed --region ${{ env.REGION }} --project ${{ env.PROJECT_ID }} --format="value(status.url)")
          REVISION_ID=$(gcloud run services describe ${{ env.SERVICE_NAME }} --platform managed --region ${{ env.REGION }} --project ${{ env.PROJECT_ID }} --format="value(status.latestReadyRevisionName)")
          echo "CLOUD_RUN_SERVICE_URL=${SERVICE_URL}" >> "$GITHUB_OUTPUT"
          echo "CLOUD_RUN_REVISION_ID=${REVISION_ID}" >> "$GITHUB_OUTPUT"
        # Using "value(status.url)" and "value(status.latestReadyRevisionName)" to get current info

      - name: Log Deployment to Google Sheet (SUCCESS)
        if: success()
        run: |
          curl -X POST -H "Content-Type: application/json" \
            -d '{
              "commitHash": "${{ github.sha }}",
              "deployedBy": "${{ github.actor }}",
              "status": "SUCCESS",
              "cloudRunServiceUrl": "${{ steps.get_url_revision.outputs.CLOUD_RUN_SERVICE_URL }}",
              "cloudRunRevisionId": "${{ steps.get_url_revision.outputs.CLOUD_RUN_REVISION_ID }}",
              "githubWorkflowRunUrl": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }' \
            ${{ env.APPS_SCRIPT_WEBHOOK_URL }}

      - name: Log Deployment to Google Sheet (FAILURE)
        if: failure()
        run: |
          curl -X POST -H "Content-Type: application/json" \
            -d '{
              "commitHash": "${{ github.sha }}",
              "deployedBy": "${{ github.actor }}",
              "status": "FAILURE",
              "cloudRunServiceUrl": "", # No URL if deployment failed
              "cloudRunRevisionId": "", # No revision ID if deployment failed
              "githubWorkflowRunUrl": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }' \
            ${{ env.APPS_SCRIPT_WEBHOOK_URL }}
```
**Important Replacements in `deploy.yml`:**
*   `YOUR_GCP_PROJECT_ID`: Your Google Cloud Project ID.
*   `YOUR_GITHUB_ORG/YOUR_REPO_NAME`: Your GitHub organization/username and repository name (e.g., `my-org/three-ai-relay-app`). This is critical for the Workload Identity Federation binding.
*   `YOUR_APPS_SCRIPT_WEBHOOK_URL`: The URL you obtained after deploying your Apps Script as a web app.
*   `REGION`: Choose a GCP region close to your users (e.g., `us-central1`, `europe-west1`).

#### 7. Initial Deployment

1.  **Commit and Push:** Commit the `Dockerfile` and `.github/workflows/deploy.yml` files to the `main` branch of your GitHub repository.
2.  **Trigger Workflow:** Pushing these files will automatically trigger the GitHub Actions workflow.
3.  **Monitor Deployment:**
    *   **GitHub Actions:** Navigate to the "Actions" tab in your GitHub repository to see the workflow progress.
    *   **Google Cloud Console:**
        *   Go to `Cloud Build > History` to see the Docker image build.
        *   Go to `Cloud Run` to see your service being deployed and its URL.
        *   Go to your `Three-AI Relay App Deployment Log` Google Sheet to verify the entry appears.

#### 8. Google Cloud Monitoring Setup

Cloud Run services automatically integrate with Cloud Monitoring and Cloud Logging. Here's how to configure them for optimal stability.

1.  **Access Cloud Monitoring:** Go to `Monitoring` in the Google Cloud Console.

2.  **Explore Metrics & Dashboards:**
    *   Navigate to `Metrics explorer` to query various metrics for your Cloud Run service (e.g., `run.googleapis.com/request_count`, `run.googleapis.com/request_latencies`, `run.googleapis.com/container/cpu/utilizations`, `run.googleapis.com/container/memory/utilizations`).
    *   Go to `Dashboards > + Create Dashboard`. Add charts for key metrics:
        *   **Request Count (Total):** `run.googleapis.com/request_count` (sum)
        *   **Request Latency (P99):** `run.googleapis.com/request_latencies` (99th percentile)
        *   **Error Count (4xx/5xx):** Filter `run.googleapis.com/request_count` by `response_code_class = "4xx"` or `"5xx"`.
        *   **CPU Utilization:** `run.googleapis.com/container/cpu/utilizations` (average)
        *   **Memory Utilization:** `run.googleapis.com/container/memory/utilizations` (average)
        *   **Instance Count:** `run.googleapis.com/container/instance_count`

3.  **Set Up Alerting Policies:**
    *   Navigate to `Alerting > + Create Policy`.
    *   **Condition:**
        *   **Example 1: High Error Rate**
            *   **Metric:** `Cloud Run Revision > Request count`
            *   **Filter:** `response_code_class = "5xx"` (or "4xx" for client errors)
            *   **Aggregator:** `sum`
            *   **Transform:** `delta`
            *   **Aligner:** `sum`
            *   **Period:** `5 minutes`
            *   **Threshold:** `is above` `5` (e.g., more than 5 errors in 5 minutes)
            *   **Trigger:** `Any time series violates`
        *   **Example 2: High Latency**
            *   **Metric:** `Cloud Run Revision > Request latencies`
            *   **Aggregator:** `99th percentile`
            *   **Aligner:** `mean`
            *   **Period:** `5 minutes`
            *   **Threshold:** `is above` `1000 ms` (1 second)
        *   **Example 3: High CPU/Memory Utilization**
            *   **Metric:** `Cloud Run Revision > CPU Utilization` / `Memory Utilization`
            *   **Threshold:** `is above` `80 %`
    *   **Notification Channels:** Configure notification channels (e.g., Email, Slack, PagerDuty).
    *   **Name the Policy:** Give it a descriptive name (e.g., `Cloud Run 5xx Error Rate`).

4.  **Cloud Logging:**
    *   Go to `Logging > Logs Explorer`.
    *   Filter by `Resource type: Cloud Run Revision` and your `Service Name`.
    *   You will see standard request logs (from Cloud Run) and any `console.log`, `console.error`, etc., statements from your Node.js application. This is essential for debugging.
    *   **Log-based Metrics & Alerts:** You can also create custom log-based metrics and alerts for specific application-level errors or events found in your logs (e.g., a specific error message `SQLITE_ERROR` if your SQLite database encounters an issue).

---

### **Complete Integration Code**

#### 1. `Dockerfile` (located in your app's root directory)

```dockerfile
# Use a Node.js LTS image as the base
FROM node:20-slim

# Create and change to the app directory
WORKDIR /app

# Copy package.json and package-lock.json and install dependencies
# This step is cached and only re-runs if package