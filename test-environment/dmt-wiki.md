# CI/CD Pipeline Strategy — RAPID/DMT #

1. ### Purpose & Scope<BR> 
Build, test, scan, and (for dev) of CI/CD pipeline with consistent versioning and quality gates. Applies to MRs targeting dev and commits to dev. <BR>
Pipeline references: dmt-pipeline-core, dmt-service-core, & dmt-web-client <BR>

2. ### Triggers & Branch Policy<BR> 
Runs on :<BR> 
- Merge Requests where target branch is dev.<BR> 
- Pushes to dev.<BR> 
Why: keeps pipelines focused on integration branches, MRs get full validation, and dev pushes also produce a versioned image which deploys to dmt-dev and creates a Git tag. <BR> 

3. ### Stages & Key Jobs<BR>
PREPARE — the set_package_version job exports PACKAGE_VERSION via dotenv.<BR>
VALIDATE — the check_version job blocks the pipeline if the image tag already exists, the k8s_comparison requires the kustomize tag to match package.json, and the validate:kustomize job builds the overlays.<BR>
COMPILE — build_application job runs.<BR>
TEST — lint (runs "npm run lint" and maps to ESLint), test (runs "npm run test" and executes unit/integration tests), and test-coverage (runs "npm run coverage" but does not enforce a threshold or generate viewable artifacts; it passes/fails based on the script's exit code).<BR>
PUBLISH —<BR>
MR to dev: build_image publishes<BR>
Commit to dev: build_version_image publishes<BR>
SCAN — GitLab Dependency-Scanning, SAST, Secret Detection templates, plus sonarqube and trivy (artifacts JSON/CSV/HTML).<BR>
DEPLOY — deploy_dev (AKS login, create/patch secrets, kubectl apply -k overlay).<BR>
TAG — create_git_tag using release-cli <BR>

4. ### Versioning Requirements<BR>
PACKAGE_VERSION is read from package.json.<BR>
version confirmation: check_version will not run if ${PACKAGE_VERSION} already exists in the registry.<BR>
K8s consistency: k8s_comparison enforces k8s/base/kustomization.yaml image tag must match the package.json version.<BR>
Publishing policy: MR builds a temporary slug tag; dev pushes and unchangeable publish artifact ${PACKAGE_VERSION}. <BR>

5. ### Container Building Method<BR>
Builder: Kaniko (no Docker-in-Docker).<BR>
Base: private IronBank Node image; build-time variables (args) for private npm scope; and the cache repo defined.<BR>
Skopeo is utilized to list existing tags (auth via project token). <BR>

6. ### Security & Quality Gates<BR>
GitLab templates: Dependency-Scanning (SBOM), SAST, Secret Detection.<BR>
SonarQube analysis (non-blocking).<BR>
Trivy container scan; exports GitLab-compatible report + CSV + HTML with all artifacts retained by the pipeline. <BR>

7. ### Deployments<BR>
Environment: dmt-dev on AKS (AzureUSGovernment cloud).<BR>
Process: SPN login -> kubelogin -> set namespace -> create/update secrets -> kubectl apply -k k8s/overlays/....<BR>
Ingress: Application Gateway ingress with host/path routing (dmt.pti-n.com). <BR>

8. ### Releases & Tags<BR>
Only on commit-to-dev will the pipelines tag the repo v${PACKAGE_VERSION} via release-cli.  This prevents unwanted images from being put in the container registry.<BR>

9. ### Operational Settings<BR>
Runner tag: pti-group-n.<BR>
Timeout: 30m default.<BR>
Retry: up to 2x on runner/system/api failure.<BR>
Interruptible: true (saves runner minutes on superseded runs). <BR>

10. ### Config & Secrets<BR>
Kustomize overlays manage env config and ingress.<BR>
Secrets are created/applied at deploy time (OAuth, DB connection string, etc.).<BR>
Local and cloud overlays are kept separate; base manifests define Deployment/Service. <BR>

11. ### Pipeline Differences<BR>
Trivy job name:  in both pipeline-core and web-client, the job name is "trivy-scan", but in service-core the job name is "trivy".<BR>
Linting failure permitted:  in pipeline-core, the lint job under the Test Stage has the allow-failure is set to true.<BR>
Kustomize overlay secrets: web-client creates dummy tls.crt and tls.key; others create .env only.<BR>

12. ### How to Extend (pattern)<BR>
New service: copy this pipeline, point IMAGE_URI, keep stages, add service-specific tests.<BR>
The AZURE_DEVOPS_REPO and IMAGE_URI (both found in variables) will need to have an updated URL inserted to point to the applicable repos.  In addition, new vars will have to be defined in Gitlab so that SonarQube will function correctly.<BR>
If you need main to publish/deploy, add rules for *rules-commit-main to the publish/deploy/tag jobs and create a prod overlay with required approvals.<BR>
