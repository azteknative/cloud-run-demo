# Cloud Run Demo Setup

This `README.md` documents the steps taken to set up a Google Cloud Run demonstration environment, including a sample application, a CI/CD pipeline with GitHub Actions, and traffic management capabilities.

## 1. GCP Project Setup

**Goal**: Create a new Google Cloud Project and enable necessary APIs.

### Create Project
```bash
gcloud projects create aztek-native-cloud-run-demo
```
_Explanation_: Creates a new GCP project named `aztek-native-cloud-run-demo`.

### Select Project and Set Region
```bash
gcloud config set project aztek-native-cloud-run-demo
gcloud config set run/region europe-west2
```
_Explanation_: Sets the newly created project as the active one and configures the default region for Cloud Run deployments to `europe-west2` (London).

### Enable Required APIs
```bash
gcloud services enable run.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com --project aztek-native-cloud-run-demo
```
_Explanation_: Enables the Cloud Run, Artifact Registry, and Cloud Build APIs, which are essential services for container deployment and management.

## 2. Sample Application Setup

**Goal**: Create a simple Python Flask application with Dockerization.

### `main.py`
_Explanation_: A basic Flask application that serves a "Hello, World!" message.

### `requirements.txt`
_Explanation_: Lists the Python dependencies (`Flask`) for the application.

### `Dockerfile`
_Explanation_: Defines the steps to build a Docker image for the Flask application.

## 3. Initial Cloud Run Deployment

**Goal**: Deploy the initial version of the application to Cloud Run.

### Create Artifact Registry Repository
```bash
gcloud artifacts repositories create cloud-run-repo --repository-format=docker --location=europe-west2 --project=aztek-native-cloud-run-demo --description="Docker repository for Cloud Run demo"
```
_Explanation_: Creates a Docker repository in Artifact Registry to store the container images.

### Build and Push Docker Image (v1)
```bash
gcloud builds submit --tag europe-west2-docker.pkg.dev/aztek-native-cloud-run-demo/cloud-run-repo/hello-world --project=aztek-native-cloud-run-demo
```
_Explanation_: Uses Cloud Build to build the Docker image from the current directory and pushes it to the Artifact Registry. This is the initial `v1` image.

### Deploy to Cloud Run
```bash
gcloud run deploy hello-world-service --image europe-west2-docker.pkg.dev/aztek-native-cloud-run-demo/cloud-run-repo/hello-world --platform managed --region europe-west2 --allow-unauthenticated --project=aztek-native-cloud-run-demo
```
_Explanation_: Deploys the `v1` Docker image to Cloud Run, creating a new service named `hello-world-service` and allowing unauthenticated access.

## 4. GitHub Actions CI/CD Integration

**Goal**: Set up a GitHub Actions workflow to automate builds and deployments.

### Create Service Account for GitHub Actions
```bash
gcloud iam service-accounts create github-actions-deployer --display-name "GitHub Actions Deployer" --project aztek-native-cloud-run-demo
```
_Explanation_: Creates a dedicated GCP service account for GitHub Actions.

### Grant IAM Roles to Service Account
```bash
gcloud projects add-iam-policy-binding aztek-native-cloud-run-demo --member="serviceAccount:github-actions-deployer@aztek-native-cloud-run-demo.iam.gserviceaccount.com" --role="roles/run.admin" --project aztek-native-cloud-run-demo
gcloud projects add-iam-policy-binding aztek-native-cloud-run-demo --member="serviceAccount:github-actions-deployer@aztek-native-cloud-run-demo.iam.gserviceaccount.com" --role="roles/artifactregistry.writer" --project aztek-native-cloud-run-demo
gcloud projects add-iam-policy-binding aztek-native-cloud-run-demo --member="serviceAccount:github-actions-deployer@aztek-native-cloud-run-demo.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser" --project aztek-native-cloud-run-demo
gcloud projects add-iam-policy-binding aztek-native-cloud-run-demo --member="serviceAccount:github-actions-deployer@aztek-native-cloud-run-demo.iam.gserviceaccount.com" --role="roles/cloudbuild.builds.editor" --project aztek-native-cloud-run-demo
gcloud projects add-iam-policy-binding aztek-native-cloud-run-demo --member="serviceAccount:github-actions-deployer@aztek-native-cloud-run-demo.iam.gserviceaccount.com" --role="roles/viewer" --project aztek-native-cloud-run-demo
```
_Explanation_: Assigns necessary roles (`Cloud Run Admin`, `Artifact Registry Writer`, `Service Account User`, `Cloud Build Editor`, `Viewer`) to the service account, granting it permissions to manage Cloud Run, push images, act on behalf of other service accounts, trigger Cloud Builds, and view logs.

### Generate Service Account Key
```bash
gcloud iam service-accounts keys create gcp-key.json --iam-account github-actions-deployer@aztek-native-cloud-run-demo.iam.gserviceaccount.com --project aztek-native-cloud-run-demo
```
_Explanation_: Generates a JSON key for the service account. **This key's content must be added as a GitHub Secret (e.g., `GCP_SA_KEY`) for authentication.** Remember to delete the local file (`gcp-key.json`) after adding the secret.

### GitHub Actions Workflow File (`.github/workflows/deploy.yaml`)
_Explanation_: This workflow automates the build and deployment of the application to Cloud Run on pushes to the `main` branch. It uses the `GCP_SA_KEY` for authentication.

## 5. Traffic Management (Canary Deployments)

**Goal**: Demonstrate gradual traffic shifting between Cloud Run revisions.

### Update Application and Build New Image (v2)
_Explanation_: The `main.py` file was updated to represent `v2` of the application. The image was then built and tagged as `v2` in Artifact Registry.

```bash
gcloud builds submit --tag europe-west2-docker.pkg.dev/aztek-native-cloud-run-demo/cloud-run-repo/hello-world:v2 --project=aztek-native-cloud-run-demo
```
_Explanation_: Builds the `v2` Docker image and pushes it to Artifact Registry with the `v2` tag.

### Deploy v2 without Traffic
```bash
gcloud run deploy hello-world-service --image europe-west2-docker.pkg.dev/aztek-native-cloud-run-demo/cloud-run-repo/hello-world:v2 --no-traffic --platform managed --region europe-west2 --project=aztek-native-cloud-run-demo
```
_Explanation_: Deploys the `v2` image as a new revision but ensures it receives 0% of traffic initially.

### List Revisions (for reference)
```bash
gcloud run revisions list --service hello-world-service --region europe-west2 --project aztek-native-cloud-run-demo --format="json"
```
_Explanation_: Shows all available revisions for the service, including their names (e.g., `hello-world-service-00002-l6g` for v1, `hello-world-service-00003-ndh` for v2).

### Shift Traffic (Canary 10% to v2)
```bash
gcloud run services update-traffic hello-world-service --to-revisions hello-world-service-00003-ndh=10,hello-world-service-00002-l6g=90 --region europe-west2 --project aztek-native-cloud-run-demo
```
_Explanation_: Directs 10% of incoming traffic to the `v2` revision and keeps 90% on the `v1` revision.

### Shift Traffic (50% to v2)
```bash
gcloud run services update-traffic hello-world-service --to-revisions hello-world-service-00003-ndh=50,hello-world-service-00002-l6g=50 --region europe-west2 --project aztek-native-cloud-run-demo
```
_Explanation_: Increases the traffic split to 50% for `v2` and 50% for `v1`.

### Shift Traffic (100% to v2)
```bash
gcloud run services update-traffic hello-world-service --to-revisions hello-world-service-00003-ndh=100 --region europe-west2 --project aztek-native-cloud-run-demo
```
_Explanation_: Completes the rollout by directing 100% of traffic to the `v2` revision.

---

_Note_: Replace `aztek-native-cloud-run-demo` with your actual project ID if different. 