# Local GitHub Actions: Dev and Prod

This repository deploys the Java application to Docker Desktop on a Windows
self-hosted GitHub Actions runner. It does not deploy to Azure or the public
internet.

## Targets

| Environment | Host URL | Docker container | Docker image |
|---|---|---|---|
| dev | `http://localhost:4040` | `gs-serving-web-content-dev` | `gs-serving-web-content-dev:latest` |
| prod | `http://localhost:6060` | `gs-serving-web-content-prod` | `gs-serving-web-content-prod:latest` |

The existing `gs-serving-web-content` container on port `8080` is unrelated
to these deployments and is not removed by the workflow.

## Runner setup

1. Start Docker Desktop and confirm Linux containers are available:

   ```powershell
   docker info
   ```

2. Create a runner registration token:

   ```powershell
   $repo = "prashanth0509/gs-serving-web-content"
   $token = gh api --method POST "repos/$repo/actions/runners/registration-token" --jq .token
   Write-Output $token
   ```

   Treat the printed token as temporary secret material. Do not commit it.

3. On the Windows machine, download and configure a GitHub Actions runner from
   the repository's **Settings > Actions > Runners > New self-hosted runner**
   page. During configuration, use the label `windows-local`.

   You can verify the registered label by running:

   ```powershell
   gh api repos/prashanth0509/gs-serving-web-content/actions/runners --jq '.runners[] | {name,status,labels:[.labels[].name]}'
   ```

4. Install and start the runner as a Windows service. The runner process must
   remain running and must be able to run Docker commands.

5. Create GitHub environments named `dev` and `prod`:

   ```powershell
   gh api --method PUT repos/prashanth0509/gs-serving-web-content/environments/dev
   gh api --method PUT repos/prashanth0509/gs-serving-web-content/environments/prod
   ```

   In GitHub, open **Settings > Environments > prod**, add yourself under
   **Required reviewers**, and save. This creates the manual approval gate.
   GitHub must allow the same deployment actor to approve in your repository
   settings if you are the only person.

## Deploy with a manual workflow run

1. Open **Actions** in the GitHub fork.
2. Select **Local Dev and Prod Deployment**.
3. Choose **Run workflow**.
4. Select `dev` or `prod`.
5. For `prod`, approve the deployment when GitHub pauses for the environment.

The workflow builds `complete/Dockerfile`, replaces only the matching dev or
prod container, and checks the greeting endpoint until it becomes healthy.

## Deploy with branch pushes

- Push to the `dev` branch to deploy dev.
- Deploy prod by using **Run workflow** and selecting `prod`.
- Production always waits for the `prod` environment approval gate.

Create and push a dev branch if it does not already exist:

```powershell
git switch -c dev
git push -u origin dev
```

## Verify locally

```powershell
docker ps

Invoke-WebRequest -UseBasicParsing "http://localhost:4040/greeting?name=Dev"
Invoke-WebRequest -UseBasicParsing "http://localhost:6060/greeting?name=Prod"
```

The dev request should contain `Hello, Dev`; the prod request should contain
`Hello, Prod`.

## Stop deployments

```powershell
docker rm -f gs-serving-web-content-dev
docker rm -f gs-serving-web-content-prod
```

Do not stop the `gs-serving-web-content` container unless you intentionally
want to stop the original manual demo on port `8080`.
