# Node.js + Express + PostgreSQL on Kind with GitHub Actions CI/CD

This project is a small Node.js API connected to PostgreSQL, containerized with Docker, and deployed to a local Kind Kubernetes cluster using a self-hosted GitHub Actions runner.

The goal is to demonstrate a complete CI/CD flow:

- developer pushes to main
- GitHub Actions runs tests
- a Docker image is built
- the image is pushed to GHCR
- a self-hosted Windows runner deploys the app to a local Kind cluster
- PostgreSQL and the Node API run inside Kubernetes

---

## What is included

- Dockerfile for the Node.js application
- Kubernetes manifests for the namespace, ConfigMap, Secret, app Deployment, app Service, PostgreSQL Deployment, and PostgreSQL Service
- GitHub Actions pipeline for CI and CD
- local Kind-based Kubernetes deployment model
- NodePort-based local exposure for the app

---

## Tools selected and why

### GitHub Actions
Used for CI/CD because it integrates directly with GitHub and supports both GitHub-hosted runners and self-hosted runners.

### Docker
Used to containerize the application so the same artifact can be built and deployed consistently across environments.

### GHCR (GitHub Container Registry)
Used to store the built image created by the GitHub Actions workflow. The image is tagged using the Git commit SHA for traceability and reproducibility.

### Kubernetes
Used for orchestration, declarative deployment, and service exposure.

### Kind
Chosen instead of cloud Kubernetes because the assessment requires a local, low-cost, real Kubernetes environment that can be demonstrated from a workstation.

### kubectl
Used as the standard Kubernetes CLI for applying manifests and validating deployment health.

### Self-hosted Windows GitHub Actions runner
Used because the target Kind cluster is running locally on the same Windows machine used for the assessment environment.

---

## Architecture

```text
Developer -> GitHub push -> GitHub Actions -> GHCR
                                       |
                                       v
                              self-hosted Windows runner
                                       |
                                       v
                                      Kind
                              +---------------------+
                              |                     |
                              v                     v
                        Node API Pod         PostgreSQL Pod
                              |                     |
                              v                     v
                           NodePort Service      ClusterIP Service
                              |
                              v
                        localhost:30080
```

---

## Requirements

- Git
- Node.js 20+
- npm
- Docker
- Kind
- kubectl
- GitHub repository with Actions enabled
- Self-hosted GitHub Actions runner configured for Windows x64

---

## Local setup

### Step 1: Install Kind on Windows

Kind is used to run a local Kubernetes cluster.

1. Download Kind from the [Kind releases page](https://github.com/kubernetes-sigs/kind/releases)
2. Add Kind to your system PATH so it can be run from any PowerShell location
3. Verify the installation:

```powershell
kind --version
```

### Step 2: Create the Kind cluster configuration

Kind needs a configuration file to expose the NodePort 30080 from the container to the host machine.

1. On your Windows machine, create a file named `kind-config.yaml` (this file is local to your machine, not part of the repository):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
```

2. Create the Kind cluster using this configuration:

```powershell
kind create cluster --name kind --config kind-config.yaml
```

3. Verify the cluster was created:

```powershell
kubectl cluster-info --context kind-kind
kubectl get nodes --context kind-kind
```

### Step 3: Install and configure the self-hosted GitHub Actions runner

The self-hosted Windows runner is used by GitHub Actions to deploy the application to your local Kind cluster.

1. On your Windows x64 machine, create a folder for the runner:

```powershell
mkdir actions-runner
cd actions-runner
```

2. Download the latest self-hosted runner package:

```powershell
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.336.0/actions-runner-win-x64-2.336.0.zip -OutFile actions-runner-win-x64-2.336.0.zip
```

3. Optionally validate the hash (recommended for security):

```powershell
if((Get-FileHash -Path actions-runner-win-x64-2.336.0.zip -Algorithm SHA256).Hash.ToUpper() -ne 'd59123a43003e357b0805b5d0f611d0bd2f65ab67d51bd070dd4e7a0f685c162'.ToUpper()){ throw 'Computed checksum did not match' }
```

4. Extract the runner package:

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem ; [System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD/actions-runner-win-x64-2.336.0.zip", "$PWD")
```

5. Navigate to your GitHub repository, go to **Settings > Actions > Runners > New self-hosted runner** to get a registration token.

6. Configure the runner (replace the URL and token with your values from step 5):

```powershell
./config.cmd --url https://github.com/YOUR_USERNAME/node-express-postgresql-server --token YOUR_REGISTRATION_TOKEN
```

7. Start the runner:

```powershell
./run.cmd
```

The runner will now listen for jobs from GitHub Actions. Keep this terminal open.

> Tip: To run the runner as a service in the background, you can use `./config.cmd` with the `--runAsService` flag on a new PowerShell terminal with administrator privileges.

### Step 4: Clone the project

1. Clone the repository:

```powershell
git clone https://github.com/YOUR_USERNAME/node-express-postgresql-server
cd node-express-postgresql-server
```

2. Install dependencies locally:

```powershell
npm install
```

3. Verify the app runs locally (optional):

```powershell
npm start
```

The app will run on `http://localhost:3000` by default. Press `Ctrl+C` to stop it.

### Step 5: Add GitHub secrets

The project requires GitHub Actions secrets for the database credentials.

1. Navigate to your GitHub repository
2. Go to **Settings > Secrets and variables > Actions > New repository secret**
3. Create the following secrets:
   - `DATABASE_USER`: `postgres`
   - `DATABASE_PASSWORD`: `your-secure-password` (use a strong password for your local cluster)

### Step 6: Deploy via CI/CD

1. Make a change to the code (or just create an empty commit):

```powershell
git add .
git commit --allow-empty -m "Trigger CI/CD pipeline"
git push origin main
```

2. GitHub Actions will automatically:
   - Run the test suite
   - Build the Docker image
   - Push the image to GHCR
   - Deploy to your local Kind cluster via the self-hosted runner

3. Monitor the workflow in GitHub: **Actions > CI/CD workflow**

4. Once the workflow completes successfully, run the following command on your Windows machine to access the application:

```powershell
kubectl port-forward -n node-postgres-app service/node-api 30080:80
```

5. Access the app at:

```text
http://localhost:30080/users
http://localhost:30080/messages
```

Keep the port-forward terminal open to maintain access to the application.

## API routes

The app exposes the following REST endpoints:

- `/users`
- `/users/:id`
- `/messages`
- `/messages/:id`
- `/session`

Examples:

```bash
curl http://localhost:30080/users
curl http://localhost:30080/messages
```

---

## CI/CD flow

The workflow is designed to trigger on a direct push to the `main` branch.

### Pipeline steps

1. `test` job
   - checks out the repository
   - sets up Node.js
   - runs `npm ci`
   - runs `npm test`

2. `build` job
   - logs into GHCR
   - builds the Docker image
   - pushes it to GHCR using the GitHub commit SHA as the tag

3. `deploy` job
   - runs on a self-hosted Windows runner
   - verifies Docker and kubectl
   - selects the `kind-kind` cluster context
   - applies the namespace and ConfigMap
   - creates the PostgreSQL Secret from GitHub secrets
   - deploys PostgreSQL
   - waits for the PostgreSQL rollout
   - deploys the application
   - updates the application image to the current GHCR SHA
   - waits for the app rollout
   - verifies the resources in the cluster

---

## GitHub secrets and Kubernetes Secret handling

The project uses GitHub Actions secrets for sensitive values.

The application and PostgreSQL credentials are not hardcoded into the repo. They are injected at runtime through Kubernetes.

The relevant pattern is:

- ConfigMap for non-sensitive settings such as database name, host, port, and app port
- Secret for sensitive values such as username and password
- GitHub Actions secrets are translated into Kubernetes Secret values during the deploy job

This keeps credentials out of source control while still allowing the app and PostgreSQL containers to receive the values they require at runtime.

---

## Assumptions and trade-offs

### Kind instead of cloud Kubernetes
This project intentionally uses Kind because it is a local, real Kubernetes environment suitable for the assessment and easy to demonstrate without cloud infrastructure costs.

### NodePort instead of Ingress
A NodePort is sufficient for local validation and meets the requirement for a local assessment environment. Ingress would add unnecessary complexity for this scope.

### PostgreSQL deployed as a Deployment instead of StatefulSet
The project uses a Deployment because the assessment allows either a Deployment or a StatefulSet, and this approach is simpler for a local demonstration cluster. A StatefulSet would be a stronger choice for a production-grade database workload, but it is not required for the current assessment.

### GHCR image tags use Git SHA
Using a Git SHA is intentional. It ensures immutability, traceability, and easier auditing compared with a floating `latest` tag.

### Local cluster is the target environment
The solution is designed for a local Kind cluster running on the same machine as the self-hosted Windows runner. This is appropriate for the assessment and keeps the architecture simple.

---

## What I would improve with more time

If more time were available, I would improve the solution by:

- adding proper readiness and liveness probes for the Node API
- adding a persistent volume strategy for PostgreSQL beyond a local laboratory setup
- using a StatefulSet for PostgreSQL in a more production-like configuration
- adding image scanning with Trivy
- adding Ingress and TLS for a more production-like service exposure layer
- separating development, staging, and production environments
- improving secret management with a dedicated secret provider or environment-specific configuration
- documenting rollback strategy and deployment validation in more detail

---

## Summary

This project demonstrates an end-to-end DevOps workflow for a Node.js application using:

- GitHub Actions
- Docker
- GHCR
- Kind
- Kubernetes
- kubectl
- a self-hosted Windows runner

The final result is a working local Kubernetes deployment that is suitable for assessment purposes and follows the architecture described in the project requirements.

