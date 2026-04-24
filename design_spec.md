## BUILD READY SPECIFICATION: GitHub Deploy Integration for Three-AI Relay Platform

### Overview:
This specification outlines the addition of a GitHub deployment pipeline to an existing web application that utilizes a Node.js + Express backend. The system currently integrates with Google Sheets and Google Drive for project management and artifact handling. The newly requested feature involves creating GitHub repositories for project build artifacts and deploying these via Railway, enhancing the current CI/CD processes.

### Existing System Architecture:
- **Backend**: Node.js + Express
- **Database**: better-sqlite3, with tables for `users`, `projects`, and `artifacts`.
- **Authentication**: JWT, with tokens stored in cookies.
- **Existing Endpoints**:
  - Authentication routes for user registration and login.
  - Build management routes for handling CRUD operations on projects.
- **Integration Services**:
  - Google Sheets for new build requests and status tracking.
  - Google Drive for fetching and storing build artifacts.
  
### Proposed System Enhancements:
1. **GitHub Deployment Module (`services/githubDeploy.js`)**
2. **Railway Deployment Module (`services/railwayDeploy.js`)**
3. **Database Schema Modifications**
4. **API Endpoint Addition**
5. **User Interface Updates**

### Detailed Design:

#### 1. GitHub Deployment Module (`services/githubDeploy.js`):
- **Functionality**:
  - Parse the JSON output from `_build.json` artifacts, extracting file content and names.
  - Authenticate and create a new GitHub repository named `relay-{cross_ref_id}` using a provided GITHUB_TOKEN.
  - Push all files to the repository's main branch using the GitHub Contents API.
  - Return the new repository URL.

- **Methodology**:
  - Use the GitHub REST API to manage repositories and file contents.
  - Assume authentication via OAuth tokens (stored in .env as `GITHUB_TOKEN`).

#### 2. Railway Deployment Module (`services/railwayDeploy.js`):
- **Functionality**:
  - Accepts a GitHub repository URL and initiates a deployment using the Railway API.
  - Links the Railway project to the GitHub repository.
  - Returns the deployment URL after initiating the deployment.

- **Methodology**:
  - Utilize Railway's GraphQL API for integration.
  - Manage API authentication through `RAILWAY_TOKEN` stored in .env.

#### 3. Database Schema Modifications:
- **Required Changes**:
  - Add `deploy_url TEXT` and `github_url TEXT` to the `projects` table to store URLs post-deployment.
  - Use the same error handling and migration strategy as existing modifications (`try/catch` with `ALTER TABLE`).

#### 4. API Endpoint Addition:
- **Endpoint**: `POST /api/builds/:id/deploy`
- **Authentication**: Required (use existing JWT strategy).
- **Functionality**:
  - Retrieve project artifacts from the database.
  - Invoke the `githubDeploy` and `railwayDeploy` modules sequentially.
  - Store the resulting GitHub and deployment URLs in the database.
  - Return `{ success: true, deploy_url }`.

#### 5. User Interface Updates:
- **Page**: `public/build.html`
- **Enhancements**:
  - Add a "Deploy to Railway" button for projects with a status of 'complete' and no existing `deploy_url`.
  - Incorporate a loading spinner during the deployment process.
  - Post-deployment, replace the deploy button with a direct link to the `deploy_url`.
  - If a `deploy_url` is already present, display the link immediately upon page load.

### Required Configurations:
- `.env` updates:
  - `GITHUB_TOKEN`: For GitHub API interactions.
  - `RAILWAY_TOKEN`: For Railway API interactions.

### Conclusion:
This specification details all necessary additions and modifications to integrate GitHub and Railway deployment into the existing web application structure. By following this blueprint, Claude Creation can implement these features, ensuring seamless integration and functionality.

### HANDOFF_TO_CLAUDE
Claude should now proceed with the implementation of `services/githubDeploy.js`, `services/railwayDeploy.js`, modifications to the SQLite database schema, the new API endpoint `POST /api/builds/:id/deploy`, and the necessary UI adjustments on `public/build.html`. All configurations and environmental variables must be correctly initialized to ensure functionality.