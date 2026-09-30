# 🏦 BankApp: Java Banking Web Application with GitHub Actions CI/CD

A Spring Boot banking web application, built, scanned, containerized and deployed to Kubernetes through an automated **GitHub Actions** pipeline.

![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3.3-brightgreen)
![MySQL](https://img.shields.io/badge/MySQL-8-blue)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED)
![Kubernetes](https://img.shields.io/badge/Kubernetes-ready-326CE5)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-black)

---

## 📑 Table of Contents

1. [Overview](#-overview)
2. [Tech Stack](#-tech-stack)
3. [Project Structure](#-project-structure)
4. [CI/CD Pipeline](#-cicd-pipeline)
5. [Prerequisites](#-prerequisites)
6. [Run Locally](#-run-locally)
7. [Docker](#-docker)
8. [Kubernetes Deployment](#-kubernetes-deployment)
9. [Kubernetes RBAC Setup](#-kubernetes-rbac-setup)
10. [GitHub Secrets](#-github-secrets)
11. [Code Quality: SonarQube & JaCoCo](#-code-quality-sonarqube--jacoco)
12. [Security Notes](#-security-notes)
13. [Troubleshooting](#-troubleshooting)

---

## 📖 Overview

BankApp is a server-rendered banking web application (Spring MVC + Thymeleaf) with authentication via Spring Security and persistence in MySQL through Spring Data JPA.

This repository also contains everything needed to automate the full delivery flow:

## **Code push → Build & Test → SonarQube scan → Publish artifact to Nexus → Build & push Docker image → Deploy to Kubernetes**

---

## 🧰 Tech Stack

| Layer | Technology |

|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.3.3 (Web, Security, Data JPA, Thymeleaf) |
| Database | MySQL 8 |
| Build tool | Maven (Maven Wrapper included) |
| Code coverage | JaCoCo 0.8.7 |
| Code quality | SonarQube (project key: `GC-Bank`) |
| Artifact repository | Nexus (releases and snapshots) |
| Containerization | Docker (`eclipse-temurin:17-jdk-alpine`) |
| Orchestration | Kubernetes |
| CI/CD | GitHub Actions |

---

## 📂 Project Structure

```text
.
├── .github/workflows/        # GitHub Actions workflow(s)
├── src/                      # Application source code
├── Dockerfile                # Container image definition
├── ds.yml                    # Kubernetes manifests (MySQL + BankApp + Services)
├── Setup-RBAC.md             # Kubernetes ServiceAccount / Role / RoleBinding setup
├── pom.xml                   # Maven build, JaCoCo and Nexus config
├── sonar-project.properties  # SonarQube project settings
├── mvnw / mvnw.cmd           # Maven Wrapper (Linux/macOS and Windows)
└── README.md
```

---

## 🔄 CI/CD Pipeline

The pipeline runs on every push and pull request to `main`, and deploys only on pushes to `main`.

```text
┌────────┐   ┌──────┐   ┌───────┐   ┌────────┐   ┌───────┐   ┌────────┐   ┌────────────┐
│Checkout│ → │Build │ → │ Test  │ → │ Sonar  │ → │ Nexus │ → │ Docker │ → │ Kubernetes │
│        │   │Maven │   │JaCoCo │   │ Scan   │   │Deploy │   │Build & │   │   Deploy   │
│        │   │      │   │       │   │        │   │       │   │ Push   │   │            │
└────────┘   └──────┘   └───────┘   └────────┘   └───────┘   └────────┘   └────────────┘
```

| Stage | What it does |

|---|---|
| **Build** | Compiles the code with Maven on JDK 17 |
| **Test** | Runs unit tests and generates the JaCoCo coverage report |
| **SonarQube** | Static analysis and coverage upload |
| **Nexus** | Publishes the JAR using the `distributionManagement` section of `pom.xml` |
| **Docker** | Copies the JAR into `app/`, builds the image and pushes it to Docker Hub |
| **Deploy** | Applies `ds.yml` to the Kubernetes cluster |

### Sample workflow

Save as `.github/workflows/ci-cd.yml` and adjust names/secrets to your setup.

```yaml
name: BankApp CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  IMAGE_NAME: youngminds73/bankapp

jobs:
  build-test-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0          # full history helps SonarQube

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven
          server-id: maven-snapshots          # must match <id> in pom.xml
          server-username: NEXUS_USERNAME
          server-password: NEXUS_PASSWORD

      - name: Build and test (with JaCoCo report)
        run: ./mvnw -B clean verify

      - name: SonarQube scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: >
          ./mvnw -B sonar:sonar
          -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }}
          -Dsonar.projectKey=GC-Bank

      - name: Publish artifact to Nexus
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        env:
          NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
        run: ./mvnw -B deploy -DskipTests

      - name: Upload JAR for next job
        uses: actions/upload-artifact@v4
        with:
          name: bankapp-jar
          path: target/*.jar

  docker:
    needs: build-test-scan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download JAR into app/ (Dockerfile expects app/*.jar)
        uses: actions/download-artifact@v4
        with:
          name: bankapp-jar
          path: app

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    needs: docker
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f ds.yml -n webapps
          kubectl rollout restart deployment/bankapp -n webapps
          kubectl rollout status deployment/bankapp -n webapps
```

---

## ✅ Prerequisites

- JDK 17
- Maven 3.9+ (or use the included `./mvnw`)
- MySQL 8 (or Docker to run it)
- Docker (for container builds)
- A Kubernetes cluster and `kubectl` (for deployment)
- A GitHub repository with the secrets listed [below](#-github-secrets)

---

## 💻 Run Locally

## **1. Start MySQL**

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=<your-password> \
  -e MYSQL_DATABASE=bankappdb \
  -p 3306:3306 mysql:8
```

## **2. Run the application**

```bash
export SPRING_DATASOURCE_URL="jdbc:mysql://localhost:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true"
export SPRING_DATASOURCE_USERNAME=root
export SPRING_DATASOURCE_PASSWORD=<your-password>

./mvnw spring-boot:run
```

## **3. Open** `http://localhost:8080`

## **Run tests**

```bash
./mvnw clean test
```

The JaCoCo report is generated at `target/site/jacoco/index.html`.

## **Build the JAR**

```bash
./mvnw clean package
```

---

## 🐳 Docker

The `Dockerfile` expects the built JAR inside an `app/` folder:

```bash
./mvnw clean package -DskipTests
mkdir -p app && cp target/*.jar app/

docker build -t youngminds73/bankapp:latest .
docker run -p 8080:8080 \
  -e SPRING_DATASOURCE_URL="jdbc:mysql://<mysql-host>:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true" \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=<your-password> \
  youngminds73/bankapp:latest
```

---

## ☸️ Kubernetes Deployment

`ds.yml` defines four resources:

| Resource | Kind | Purpose |

|---|---|---|
| `mysql` | Deployment | MySQL 8 with database `bankappdb` (`Recreate` strategy) |
| `mysql-service` | Service | Internal access to MySQL on port 3306 |
| `bankapp` | Deployment | The Spring Boot app (image `youngminds73/bankapp:latest`, port 8080) |
| `bankapp-service` | Service (`LoadBalancer`) | Exposes the app on port 80 → 8080 |

```bash
kubectl create namespace webapps
kubectl apply -f ds.yml -n webapps
kubectl get pods,svc -n webapps
```

Get the external address of the app:

```bash
kubectl get svc bankapp-service -n webapps
```

> ⚠️ The MySQL deployment has no persistent volume, so **database data is lost when the pod restarts**. Add a `PersistentVolumeClaim` for anything beyond a demo.

---

## 🔐 Kubernetes RBAC Setup

The pipeline needs a ServiceAccount with limited permissions in the `webapps` namespace. The full steps are in [Setup-RBAC.md](./Setup-RBAC.md):

1. Create the `jenkins` ServiceAccount in `webapps`
2. Create `app-role` (Role) with access to common namespaced resources
3. Bind it with `app-rolebinding` (RoleBinding)
4. Create `jenkins-cluster-role` and its ClusterRoleBinding for `persistentvolumes`
5. Generate a token for the ServiceAccount and use it to build the kubeconfig stored in the `KUBE_CONFIG` secret

---

## 🔑 GitHub Secrets

Add these in **Repository → Settings → Secrets and variables → Actions**:

| Secret | Description |

|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not your password) |
| `SONAR_TOKEN` | SonarQube authentication token |
| `SONAR_HOST_URL` | SonarQube server URL |
| `NEXUS_USERNAME` | Nexus user with deploy rights |
| `NEXUS_PASSWORD` | Nexus password |
| `KUBE_CONFIG` | Base64-encoded kubeconfig for the deploy ServiceAccount |

Create the kubeconfig value with:

```bash
base64 -w 0 kubeconfig.yaml      # Linux
base64 -i kubeconfig.yaml        # macOS
```

---

## 📊 Code Quality: SonarQube & JaCoCo

- `sonar-project.properties` sets the project key and name to `GC-Bank`.
- JaCoCo is configured in `pom.xml`: `prepare-agent` runs before tests and `report` runs in the `test` phase.
- Run a scan manually:

```bash
./mvnw clean verify sonar:sonar \
  -Dsonar.host.url=<SONAR_URL> \
  -Dsonar.token=<SONAR_TOKEN>
```

---

## 🛡️ Security Notes

Some values in this repo are fine for a demo but should be fixed before real use:

- **Do not commit credentials.** `ds.yml` currently contains the MySQL root password in plain text. Move it to a Kubernetes `Secret` and reference it with `secretKeyRef`.
- **Avoid the `latest` tag** in production. Deploy the commit-SHA tag pushed by the pipeline.
- **Use HTTPS for Nexus.** `pom.xml` points to an `http://` Nexus URL, so credentials travel unencrypted.
- **Use least privilege.** `app-role` grants broad access (including `secrets`). Trim it to what the pipeline really needs.
- **Do not run MySQL as `root`** from the app; create a dedicated database user.

---

## 🧯 Troubleshooting

| Problem | Fix |

|---|---|
| `COPY app/*.jar` fails in Docker build | The JAR must be in `app/` before `docker build`. Check the download/copy step. |
| `403` when pushing to GitHub | You are authenticated as an account without write access. Check `git remote -v` and your credentials. |
| Pod `CrashLoopBackOff` for `bankapp` | MySQL probably isn't ready yet. Check `kubectl logs deploy/bankapp -n webapps`. |
| `Communications link failure` | Verify `SPRING_DATASOURCE_URL` points to `mysql-service` and the MySQL pod is running. |
| Nexus deploy returns `401` | Check `NEXUS_USERNAME/PASSWORD` and that `server-id` matches the `<id>` in `pom.xml`. |
| Sonar step fails to connect | Confirm `SONAR_HOST_URL` is reachable from GitHub runners (use a self-hosted runner for private servers). |

---

## 🤝 Contributing

1. Fork the repo and create a branch: `git switch -c feature/my-change`
2. Commit your changes and push the branch
3. Open a Pull Request. The pipeline will build, test and scan it automatically.

---
