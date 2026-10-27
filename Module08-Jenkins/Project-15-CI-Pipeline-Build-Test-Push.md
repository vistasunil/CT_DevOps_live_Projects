# Project 15 — CI Pipeline: Build → Test → Image → Registry

| | |
|---|---|
| **Module** | 8 — Jenkins |
| **Level** | L2 — Engineer |
| **Role you play** | DevOps Engineer building the delivery pipeline for a product team |
| **Scope** | `Jenkins-hands-on.md` — install, jobs, pipelines, agents, credentials, plugins |

---

## 1. Business problem

Today a developer builds the application on their laptop, runs some tests, builds a Docker image
with whatever tag they feel like, and pushes it. Builds are not reproducible, test results are
not recorded, and "it works on my machine" is the standard explanation for failures.

Deliver a **Jenkins CI pipeline as code** that, on every commit: checks out the source, builds,
runs the test suite, publishes the results, builds a Docker image tagged with the Git SHA, pushes
it to the registry, and fails loudly when anything breaks.

## 2. Objectives

1. Install Jenkins LTS on Ubuntu 26.04 with OpenJDK 21.
2. Configure a build agent (so the controller does not run builds).
3. Store registry/SCM credentials in the Jenkins credential store — never in the Jenkinsfile.
4. Write a **declarative `Jenkinsfile`** with stages, parallelism, artefact archiving, test
   reporting and post actions.
5. Trigger builds from a Git push and prove a failing test breaks the build.

## 3. Scope

**In scope:** Jenkins install, plugins, nodes/agents, credentials, freestyle vs pipeline,
declarative pipeline syntax (`agent`, `stages`, `steps`, `environment`, `options`, `parameters`,
`when`, `parallel`, `post`), `junit`, `archiveArtifacts`, `withCredentials`, Docker build/push
from Jenkins, webhook/poll triggers.

**Out of scope:** CD/deployment and approvals (Project 16), shared libraries, Kubernetes agents.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** CI-SOP-015 · **Title:** Continuous integration pipeline standard

### 4.1 Pipeline principles

| # | Principle | Implementation |
|---|---|---|
| 1 | Pipeline is **code in the app repo** | `Jenkinsfile` at the repo root |
| 2 | Builds run on **agents**, not the controller | Controller is for scheduling/UI only |
| 3 | **Fail fast** | Cheap stages first (lint → unit → build → image) |
| 4 | **No secrets in the pipeline file** | `credentials()` / `withCredentials` |
| 5 | **Every build is traceable** | Image tagged `git-<shortSha>` and `build-<number>` |
| 6 | **Test results are published** | `junit` step, not just console output |
| 7 | **Clean workspace** per build | `cleanWs()` / `skipDefaultCheckout` + explicit checkout |
| 8 | **Timeouts on everything** | `options { timeout(...) }` — a hung build blocks the queue |
| 9 | Build **once**, reuse the artefact downstream | No rebuilding per environment |

### 4.2 Build gates

| Gate | Fails the build? |
|---|---|
| Compile error | Yes |
| Unit test failure | Yes |
| Smoke/UI test failure | Yes |
| Image build failure | Yes |
| Registry push failure | Yes |
| Lint warnings | No (reported, tracked) |

### 4.3 Incident procedure for a red pipeline
1. Do not re-run hoping it passes. Read the console log and the published test report.
2. Classify: code defect / test defect / infrastructure (agent, network, registry).
3. Infrastructure issues → fix the agent, record it; recurring issues become a ticket.
4. Code/test defects → the author fixes forward; `main` stays green.
5. If `main` is red for more than 30 minutes, revert the breaking commit.

---

## 5. Hands-on implementation

### Step 1 — Install Jenkins LTS on Ubuntu 26.04

```bash
sudo apt-get update
sudo apt-get install -y openjdk-21-jre fontconfig

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install -y jenkins

sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

```bash
# Open port 8080 to YOUR IP only
gcloud compute firewall-rules create allow-jenkins \
  --allow tcp:8080 --target-tags jenkins --source-ranges <your-ip>/32

sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Browse to `http://<vm-ip>:8080`, unlock, install the suggested plugins, and create the admin user.

Add these plugins (**Manage Jenkins → Plugins**): *Pipeline*, *Git*, *Docker Pipeline*,
*Credentials Binding*, *Pipeline Utility Steps*, *Timestamper*, *Workspace Cleanup*, *JUnit*.

### Step 2 — Prepare the build agent

On a second Ubuntu 26.04 VM (`jenkins-agent-1`):

```bash
sudo apt-get update
sudo apt-get install -y openjdk-21-jdk git maven

# Docker CE (same block as Module 3)
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Jenkins agent account, allowed to use Docker
sudo useradd -m -s /bin/bash jenkins
sudo usermod -aG docker jenkins
sudo -u jenkins docker ps      # must work WITHOUT sudo
```

In Jenkins: **Manage Jenkins → Nodes → New Node** → `agent-1`, permanent agent,
remote root `/home/jenkins`, labels `linux docker maven`, launch via SSH with the `jenkins`
user's SSH key credential. Confirm the node shows **Connected**.

### Step 3 — Credentials (never in the Jenkinsfile)

**Manage Jenkins → Credentials → System → Global**:

| ID | Type | Contents |
|---|---|---|
| `dockerhub-creds` | Username with password | Docker Hub user + **access token** |
| `github-creds` | Username with password / SSH key | GitHub user + PAT |
| `agent-ssh-key` | SSH username with private key | `jenkins` agent login |

> Use a registry **access token**, not your account password, and give it push scope only.

### Step 4 — The application repo

```
orders-api/
├── Jenkinsfile
├── Dockerfile
├── pom.xml            (or package.json / requirements.txt)
└── src/
```

`Dockerfile` (multi-stage; build inside the image so the agent needs no language toolchain):

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B -q clean package -DskipTests

FROM eclipse-temurin:21-jre
LABEL org.opencontainers.image.title="orders-api"
ENV APP_HOME=/opt/app
RUN useradd -u 10001 -m appuser && mkdir -p ${APP_HOME} && chown appuser ${APP_HOME}
WORKDIR ${APP_HOME}
COPY --from=build --chown=appuser /src/target/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD ["sh","-c","curl -fsS http://localhost:8080/health || exit 1"]
ENTRYPOINT ["java","-jar","/opt/app/app.jar"]
```

### Step 5 — The `Jenkinsfile`

```groovy
pipeline {
    agent { label 'linux && docker' }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    parameters {
        booleanParam(name: 'PUSH_IMAGE', defaultValue: true,
                     description: 'Push the image to the registry')
        booleanParam(name: 'RUN_UI_TESTS', defaultValue: true,
                     description: 'Run the Selenium smoke pack')
    }

    environment {
        REGISTRY   = 'docker.io'
        IMAGE_NAME = 'acmedevops/orders-api'
        DOCKER_CREDS = credentials('dockerhub-creds')   // exposes _USR and _PSW
    }

    stages {

        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
                script {
                    env.GIT_SHA     = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_BRANCH_ = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG   = "git-${env.GIT_SHA}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.IMAGE_TAG}"
                    currentBuild.description = "branch ${env.GIT_BRANCH_}"
                }
                sh 'git log -1 --oneline'
            }
        }

        stage('Static checks') {
            parallel {
                stage('Lint Dockerfile') {
                    steps {
                        sh '''
                          docker run --rm -i hadolint/hadolint:latest-debian \
                            hadolint --no-fail - < Dockerfile | tee hadolint.txt
                        '''
                    }
                }
                stage('Secret scan') {
                    steps {
                        sh '''
                          # Fail the build if an obvious credential pattern is committed
                          if grep -rInE "(AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY-----|ghp_[A-Za-z0-9]{20,})" \
                               --exclude-dir=.git . ; then
                            echo "Potential secret committed - failing the build"; exit 1
                          fi
                          echo "No obvious secrets found"
                        '''
                    }
                }
            }
        }

        stage('Build & unit test') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    reuseNode true
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn -B clean verify'
            }
            post {
                always {
                    junit allowEmptyResults: false, testResults: 'target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
                }
            }
        }

        stage('UI smoke tests') {
            when { expression { return params.RUN_UI_TESTS } }
            steps {
                sh '''
                  # Headless Selenium pack from Module 7 / Project 14
                  mvn -B -f ui-tests/pom.xml clean test -Pci -Dheadless=true || EXIT=$?
                  exit ${EXIT:-0}
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'ui-tests/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'ui-tests/target/screenshots/**', allowEmptyArchive: true
                }
            }
        }

        stage('Build image') {
            steps {
                sh '''
                  docker build \
                    --label "org.opencontainers.image.revision=${GIT_SHA}" \
                    --label "ci.build=${BUILD_NUMBER}" \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${IMAGE_NAME}:build-${BUILD_NUMBER} \
                    .
                  docker images ${IMAGE_NAME}
                '''
            }
        }

        stage('Image smoke test') {
            steps {
                sh '''
                  docker rm -f smoke-${BUILD_NUMBER} 2>/dev/null || true
                  docker run -d --name smoke-${BUILD_NUMBER} -p 18080:8080 ${IMAGE_NAME}:${IMAGE_TAG}

                  for i in $(seq 1 20); do
                    if curl -fsS http://localhost:18080/health > /dev/null; then
                      echo "image smoke test PASSED"; exit 0
                    fi
                    sleep 3
                  done
                  echo "image smoke test FAILED"
                  docker logs smoke-${BUILD_NUMBER} || true
                  exit 1
                '''
            }
            post {
                always {
                    sh 'docker rm -f smoke-${BUILD_NUMBER} 2>/dev/null || true'
                }
            }
        }

        stage('Push image') {
            when {
                allOf {
                    expression { return params.PUSH_IMAGE }
                    anyOf { branch 'main'; branch 'release/*' }
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                      echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                      docker push ${IMAGE_NAME}:${IMAGE_TAG}
                      docker push ${IMAGE_NAME}:build-${BUILD_NUMBER}
                      docker logout
                    '''
                }
                script {
                    env.PUSHED_IMAGE = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                    echo "Published ${env.PUSHED_IMAGE}"
                }
            }
        }
    }

    post {
        success {
            echo "BUILD OK - image ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "BUILD FAILED - see the published test report and console log"
        }
        unstable {
            echo "BUILD UNSTABLE - tests failed"
        }
        always {
            sh 'docker image prune -f || true'
            cleanWs(notFailBuild: true)
        }
    }
}
```

### Step 6 — Create the Jenkins job

**New Item → Multibranch Pipeline** (or *Pipeline* → *Pipeline script from SCM*):

| Setting | Value |
|---|---|
| Branch Sources | Git → repo URL, credential `github-creds` |
| Build Configuration | by Jenkinsfile, script path `Jenkinsfile` |
| Scan triggers | Periodically if not otherwise run — 5 minutes |

Then **Scan Repository Now** and watch the first build.

### Step 7 — Trigger on push (webhook)

In GitHub: **Settings → Webhooks → Add webhook**
`Payload URL: http://<jenkins-ip>:8080/github-webhook/`, content type `application/json`,
event: *Just the push event*.

If Jenkins is not reachable from the internet, use polling instead:

```groovy
triggers { pollSCM('H/5 * * * *') }
```

### Step 8 — Prove the gates work

```bash
# 1. Break a unit test on a branch and push
git switch -c test/break-build
# edit a test so it fails
git commit -am "test: deliberately failing assertion"
git push -u origin test/break-build
```

Expected: build **FAILS** at *Build & unit test*, the test report shows the failing test, and no
image is pushed.

```bash
# 2. Commit a fake secret and push
echo 'AKIAIOSFODNN7EXAMPLE' > creds.txt
git add creds.txt && git commit -m "oops" && git push
```

Expected: build **FAILS** at *Secret scan*.

```bash
# 3. Fix both and push to main
git revert --no-edit HEAD
```

Expected: full green pipeline, image pushed as `git-<sha>` and `build-<n>`.

### Step 9 — Verify traceability

```bash
docker pull acmedevops/orders-api:git-<sha>
docker inspect acmedevops/orders-api:git-<sha> \
  --format '{{json .Config.Labels}}' | jq .
# revision label must match the Git commit that produced it
```

---

## 6. Acceptance criteria

- [ ] Jenkins LTS running on Java 21, agent connected with the `docker` label
- [ ] Builds execute on the **agent**, not the controller (check the console "Running on")
- [ ] No credentials anywhere in the `Jenkinsfile`
- [ ] Pipeline fails on: compile error, unit test failure, committed secret, failed image smoke test
- [ ] Test results published via `junit`; artefacts archived
- [ ] Image tagged `git-<shortSha>` **and** `build-<number>`, with a `revision` label
- [ ] Push only happens on `main`/`release/*`
- [ ] Build has a timeout and old builds are rotated
- [ ] A push to the repo triggers a build automatically

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| Jenkins won't start | Wrong Java version | `java -version` → install `openjdk-21-jre`; check `journalctl -u jenkins` |
| Agent stays offline | SSH key/host key/Java missing on the agent | Check node log; install JDK on the agent; verify credentials |
| `docker: permission denied` in a build | Agent user not in the `docker` group | `sudo usermod -aG docker jenkins`, reconnect the agent |
| `junit` step: "No test report files were found" | Wrong path or tests never ran | Fix `testResults` glob; `allowEmptyResults: false` makes it visible |
| Credentials appear in the log | Used string interpolation with a secret | Use `withCredentials` and single-quoted shell; Jenkins masks `****` |
| Build hangs forever | No timeout | `options { timeout(...) }` |
| Workspace fills the disk | No build discarder / no cleanup | `buildDiscarder`, `cleanWs()`, `docker image prune` |
| Webhook does nothing | Jenkins unreachable from GitHub | Check the webhook delivery log; fall back to `pollSCM` |
| Image pushed from a feature branch | Missing `when` condition | Gate the push stage on branch |

---

## 8. Cleanup

```bash
sudo systemctl stop jenkins
gcloud compute instances delete jenkins-controller jenkins-agent-1 --zone <zone>
gcloud compute firewall-rules delete allow-jenkins
```

---

## 9. Deliverables

1. `Jenkinsfile` and `Dockerfile` in the application repo.
2. `CI-SOP-015.md` — principles, gate table, red-pipeline incident procedure.
3. Screenshots: pipeline stage view (green), a failed build with the test report, the agent node page.
4. Registry screenshot showing `git-<sha>` and `build-<n>` tags.
5. `docker inspect` output proving the revision label maps to the commit.

---

## 10. Interview talking points

* *"Walk me through your CI pipeline."* — Checkout → static checks (parallel) → build+unit tests →
  UI smoke → image build → image smoke → conditional push, with timeouts, published results and
  cleanup in `post`.
* *"Where do secrets live?"* — Jenkins credential store, injected with `withCredentials`, masked
  in logs; registry access tokens, not passwords.
* *"Why not build on the controller?"* — Security and stability: builds run arbitrary code;
  a heavy build on the controller can take the whole CI system down.
* *"How do you trace an image back to source?"* — Tag with the short Git SHA and build number, and
  stamp the revision as an OCI label; `docker inspect` proves it.
* *"What makes a pipeline trustworthy?"* — It fails when it should. I demonstrate that by
  deliberately breaking a test and a secret scan.
* *"Declarative vs scripted pipeline?"* — Declarative gives structure, validation and built-in
  `post` handling; drop into `script {}` only for genuine logic.

**CV bullet:** *Implemented Jenkins CI as code for a product team — containerised multi-stage
builds, parallel static analysis and secret scanning, published JUnit reporting, image smoke
testing and Git-SHA-traceable registry publishing — replacing unreproducible laptop builds with
an auditable automated pipeline.*
