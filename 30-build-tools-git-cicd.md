# 30 — Build Tools (Maven & Gradle), Git, CI/CD & Cloud Basics

## Part A — Maven

### 1. Concepts
- **POM** (`pom.xml`): project coordinates, dependencies, plugins, build config.
- **Coordinates (GAV)**: `groupId:artifactId:version` (+ packaging, classifier). `-SNAPSHOT` = mutable dev version.
- **Repositories**: local (`~/.m2/repository`) → remote (Maven Central, company Nexus/Artifactory).
- **Convention over configuration**: `src/main/java`, `src/main/resources`, `src/test/java`, `target/`.

### 2. Lifecycle & phases
Default lifecycle (each phase runs all previous):
`validate → compile → test → package → verify → install → deploy`
Clean lifecycle: `clean`. Site lifecycle: `site`.
```bash
mvn clean package              # compile, test, build jar
mvn clean install              # + copy to local repo (for multi-module dependencies)
mvn -DskipTests package        # skip running tests (still compiles them); -Dmaven.test.skip=true skips compiling too
mvn test -Dtest=OrderServiceTest#placesOrder
mvn verify                     # runs integration tests (Failsafe *IT)
mvn dependency:tree            # see transitive dependencies & conflicts
mvn versions:display-dependency-updates
mvn spring-boot:run            # run app
mvn spring-boot:build-image    # OCI image via buildpacks
./mvnw ...                     # Maven Wrapper — pinned Maven version, no global install needed
```
Plugins bind **goals** to phases (e.g., `maven-surefire-plugin:test` to `test`, `maven-failsafe-plugin` to `integration-test/verify`, `maven-jar-plugin:jar` to `package`).

### 3. Dependency management
```xml
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```
| Scope | Compile | Test | Runtime | Packaged | Example |
|-------|:-:|:-:|:-:|:-:|---------|
| compile (default) | ✔ | ✔ | ✔ | ✔ | spring-boot-starter-web |
| provided | ✔ | ✔ | ✘ (container provides) | ✘ | servlet API for WAR, Lombok |
| runtime | ✘ | ✔ | ✔ | ✔ | JDBC driver |
| test | ✘ | ✔ | ✘ | ✘ | JUnit |
| system / import | | | | | import used for BOMs in dependencyManagement |

- **Transitive dependencies** resolved automatically; conflicts resolved by **nearest definition wins** (then first declared).
- `<exclusions>` to remove unwanted transitive deps (e.g., exclude Tomcat to use Jetty).
- `<dependencyManagement>` centralizes versions without adding dependencies; **BOMs** (`spring-boot-dependencies`, `spring-cloud-dependencies`) imported with `<scope>import</scope>`.
- `<optional>true</optional>` — not propagated transitively (devtools).

### 4. Multi-module projects
```xml
<!-- parent pom.xml -->
<packaging>pom</packaging>
<modules>
  <module>common</module>
  <module>order-service</module>
  <module>payment-service</module>
</modules>
```
Shared dependency management & plugin config in parent; modules depend on each other via GAV. Build all: `mvn clean install` at root; build one with deps: `mvn -pl order-service -am package`.

### 5. Useful plugins
`spring-boot-maven-plugin`, `maven-compiler-plugin` (release 21, annotation processors for Lombok/MapStruct), `surefire`/`failsafe`, `jacoco-maven-plugin` (coverage), `maven-enforcer-plugin` (Java/Maven versions, banned deps), `spotless`/`checkstyle`/`pmd`/`spotbugs` (code quality), `openapi-generator-maven-plugin`, `flyway-maven-plugin`, `jib-maven-plugin`, `git-commit-id-maven-plugin`, `dependency-check-maven` (OWASP CVEs), `cyclonedx-maven-plugin` (SBOM).

## Part B — Gradle (essentials)
```kotlin
// build.gradle.kts
plugins {
    java
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
repositories { mavenCentral() }
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    runtimeOnly("org.postgresql:postgresql")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
tasks.test { useJUnitPlatform() }
```
```bash
./gradlew build | test | bootRun | bootJar | bootBuildImage | dependencies
```
`implementation` (not leaked to consumers) vs `api` (leaked; java-library plugin). Gradle is faster (incremental builds, build cache, daemon) and scriptable; Maven is more declarative/standardized. Both are fine — know the one your team uses.

## Part C — Git

### 6. Core concepts
Working directory → **staging area (index)** → local repository (commits) → remote (origin). Commits are snapshots identified by SHA; branches are movable pointers; `HEAD` points to current branch/commit.

### 7. Everyday commands
```bash
git clone <url>; git status; git diff; git diff --staged
git add -p                         # stage hunks interactively
git commit -m "feat(order): add cancel endpoint"
git log --oneline --graph --decorate --all
git switch -c feature/ORD-123-cancel-order     # (git checkout -b)
git fetch; git pull --rebase                   # rebase local commits on top of remote
git push -u origin feature/ORD-123-cancel-order
git merge main                                  # merge commit
git rebase main                                 # replay commits on top of main (linear history)
git rebase -i HEAD~3                            # squash/reword/reorder before PR
git stash push -m "wip"; git stash pop
git cherry-pick <sha>                           # apply one commit to current branch (hotfix backport)
git restore file; git restore --staged file     # discard / unstage
git reset --soft HEAD~1                         # undo commit, keep changes staged
git reset --hard origin/main                    # discard everything (dangerous)
git revert <sha>                                # new commit undoing <sha> (safe on shared branches)
git reflog                                      # recover "lost" commits
git blame file; git bisect start/good/bad       # find who/which commit introduced a bug
git tag -a v1.2.0 -m "release"; git push --tags
```

### 8. Merge vs rebase
| | Merge | Rebase |
|--|--|--|
| History | Preserves true history, merge commits | Linear, rewritten commits |
| Safety | Safe on shared branches | **Never rebase public/shared branches** |
| Conflicts | Resolved once | Possibly per commit |
Squash merge PRs for a clean main history is common.

### 9. Branching strategies
- **GitHub Flow / trunk-based**: short-lived feature branches → PR → main → deploy (with feature flags). Favoured for CI/CD.
- **Git Flow**: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` — heavier, for scheduled releases.
- **Conventional Commits** (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`; `BREAKING CHANGE`) → automated changelogs & semantic versioning (MAJOR.MINOR.PATCH).

### 10. Pull request / code review practices
Small focused PRs, descriptive title & context, link ticket, tests included, CI green, self-review first; reviewers check correctness, readability, tests, security, performance, naming; be kind and specific.

## Part D — CI/CD

### 11. Concepts
- **Continuous Integration**: every push builds & tests automatically; fast feedback; main always green.
- **Continuous Delivery**: every change is deployable (manual approval to prod).
- **Continuous Deployment**: every passing change goes to prod automatically.

### 12. Typical pipeline for a Spring Boot service
```
Push / PR
 ├─ Checkout, set up JDK (cache ~/.m2)
 ├─ Static analysis: formatting (Spotless), Checkstyle/PMD/SpotBugs, SonarQube quality gate
 ├─ Unit tests + slice tests (Surefire) + coverage (JaCoCo)
 ├─ Integration tests with Testcontainers (Failsafe)
 ├─ Security: dependency CVE scan (OWASP/Snyk/Dependabot), secrets scan (gitleaks), SAST
 ├─ Build jar → build OCI image (Buildpacks/Jib/Dockerfile) → scan image (Trivy) → sign (cosign) → push (tag = git SHA)
 ├─ Deploy to dev/staging (Helm/Kustomize/GitOps update)
 ├─ Smoke / contract / E2E / performance tests
 ├─ Manual approval (if Continuous Delivery)
 └─ Prod: canary/blue-green → monitor SLOs → auto-rollback on failure
```

### 13. GitHub Actions example
```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '21', cache: maven }
      - name: Build & test
        run: ./mvnw -B verify
      - name: Build image
        if: github.ref == 'refs/heads/main'
        run: ./mvnw -B spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=ghcr.io/${{ github.repository }}:${{ github.sha }}
      - name: Push image
        if: github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```
Other CI tools: Jenkins (Jenkinsfile, declarative pipelines), GitLab CI, Azure DevOps, CircleCI, Tekton, AWS CodePipeline.

### 14. Release & config hygiene
- Semantic versioning; immutable artifacts promoted across environments (build once, deploy many).
- Environment-specific config outside the artifact.
- Database migrations part of deployment, backward compatible.
- Feature flags to decouple deploy from release.
- Rollback plan and automated health verification.

## Part E — Cloud basics for backend devs (AWS names; GCP/Azure have equivalents)
| Need | AWS | GCP | Azure |
|------|-----|-----|-------|
| VMs | EC2 | Compute Engine | Virtual Machines |
| Managed Kubernetes | EKS | GKE | AKS |
| Containers serverless | ECS/Fargate, App Runner | Cloud Run | Container Apps |
| Functions | Lambda | Cloud Functions | Azure Functions |
| Object storage | S3 | Cloud Storage | Blob Storage |
| Relational DB | RDS / Aurora | Cloud SQL / AlloyDB | Azure SQL / Database for PostgreSQL |
| NoSQL | DynamoDB | Firestore / Bigtable | Cosmos DB |
| Cache | ElastiCache (Redis/Valkey) | Memorystore | Azure Cache for Redis |
| Queue / pub-sub | SQS / SNS / EventBridge | Pub/Sub | Service Bus / Event Grid |
| Streaming | MSK (Kafka), Kinesis | Managed Kafka, Pub/Sub | Event Hubs |
| Secrets | Secrets Manager / SSM Parameter Store | Secret Manager | Key Vault |
| IAM | IAM roles/policies | IAM | Entra ID / RBAC |
| Monitoring | CloudWatch, X-Ray | Cloud Monitoring/Trace | Azure Monitor / App Insights |
| CDN / DNS | CloudFront / Route 53 | Cloud CDN / Cloud DNS | Front Door / Azure DNS |
| Load balancer | ALB / NLB | Cloud Load Balancing | Application Gateway / LB |
| IaC | CloudFormation, CDK | Deployment Manager | Bicep/ARM | (+ **Terraform** cross-cloud) |
Concepts: regions & availability zones, VPC/subnets/security groups, IAM least privilege (prefer roles/workload identity over static keys), managed services over self-hosting, cost awareness (autoscaling, right-sizing, reserved/spot instances), **Infrastructure as Code** (Terraform/OpenTofu, Pulumi). Spring Cloud AWS / Spring Cloud GCP / Spring Cloud Azure integrate with these services.

## Interview Q&A
**Q. Maven lifecycle phases?** validate, compile, test, package, verify, install, deploy.

**Q. How do you resolve dependency conflicts in Maven?** `mvn dependency:tree`, then exclusions or pin the version in dependencyManagement / BOM.

**Q. `install` vs `deploy`?** Copy artifact to local repo vs upload to remote repository.

**Q. Merge vs rebase?** Merge preserves history with merge commits; rebase rewrites commits onto a new base for linear history (don't rebase shared branches).

**Q. `git reset` vs `git revert`?** Reset moves branch pointer (rewrites history); revert adds an inverse commit (safe for shared history).

**Q. What does your CI/CD pipeline look like?** Build, test, quality & security gates, image build/scan/push, deploy to environments with GitOps/Helm, smoke tests, progressive delivery and monitoring with rollback.

**Q. Continuous delivery vs deployment?** Delivery: always deployable, manual release step; Deployment: automatic release to production.
