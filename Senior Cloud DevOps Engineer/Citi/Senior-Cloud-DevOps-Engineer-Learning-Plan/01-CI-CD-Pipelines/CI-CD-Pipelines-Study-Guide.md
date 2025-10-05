# CI/CD Pipelines Learning Module

## Learning Objectives
By the end of this module, you will be able to:
- Design and implement enterprise-grade CI/CD pipelines
- Master Jenkins, GitLab CI/CD, GitHub Actions, and Bitbucket Pipelines
- Implement security scanning and compliance controls in pipelines
- Build deployment strategies suitable for financial services

---

## Module 1: Jenkins Enterprise Mastery (Week 1)

### Day 1-2: Jenkins Architecture and Configuration
**Study Topics:**
- Jenkins master-agent architecture
- Plugin management and security
- Global configuration and system management
- Pipeline as Code with Jenkinsfile

**Enterprise Features:**
- Role-based access control (RBAC)
- Integration with enterprise directories (LDAP/AD)
- Audit logging and compliance reporting
- High availability and disaster recovery

**Hands-On Labs:**
1. Set up Jenkins master with multiple agents
2. Configure RBAC with different user roles
3. Implement pipeline libraries for code reuse
4. Set up Jenkins backup and restore procedures

**Sample Jenkins Enterprise Pipeline:**
```groovy
@Library('enterprise-pipeline-library') _

pipeline {
    agent {
        label 'docker-enabled'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('SAST') {
                    steps {
                        // Static Application Security Testing
                        sh 'sonar-scanner -Dsonar.projectKey=banking-app'
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: 'sonar-reports',
                            reportFiles: 'index.html',
                            reportName: 'SonarQube Report'
                        ])
                    }
                }
                
                stage('Dependency Check') {
                    steps {
                        // OWASP Dependency Check
                        sh 'dependency-check --project banking-app --scan . --format HTML'
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: 'dependency-check-report',
                            reportFiles: 'dependency-check-report.html',
                            reportName: 'OWASP Dependency Check'
                        ])
                    }
                }
            }
        }
        
        stage('Build & Test') {
            steps {
                script {
                    def testResults = runTests()
                    publishTestResults testDataPublishers: [
                        [$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/*.xml']
                    ]
                    
                    if (testResults.failureCount > 0) {
                        error("Tests failed: ${testResults.failureCount} failures")
                    }
                }
            }
        }
        
        stage('Container Build') {
            steps {
                script {
                    def image = docker.build("banking-app:${env.GIT_COMMIT_SHORT}")
                    
                    // Container security scanning
                    sh "trivy image --format json --output trivy-report.json banking-app:${env.GIT_COMMIT_SHORT}"
                    
                    // Push to enterprise registry
                    docker.withRegistry('https://registry.enterprise.com', 'registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    deployToEnvironment('development', env.GIT_COMMIT_SHORT)
                    runIntegrationTests('development')
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Require approval for staging deployment
                    input message: 'Deploy to staging?', ok: 'Deploy',
                          submitterParameter: 'DEPLOYER'
                    
                    deployToEnvironment('staging', env.GIT_COMMIT_SHORT)
                    runIntegrationTests('staging')
                    runPerformanceTests('staging')
                }
            }
        }
        
        stage('Production Deployment') {
            when {
                allOf {
                    branch 'main'
                    expression { return params.DEPLOY_TO_PROD == true }
                }
            }
            steps {
                script {
                    // Additional approval for production
                    input message: 'Deploy to production?', ok: 'Deploy',
                          submitterParameter: 'PROD_DEPLOYER'
                    
                    // Blue-green deployment
                    deployBlueGreen('production', env.GIT_COMMIT_SHORT)
                }
            }
        }
    }
    
    post {
        always {
            // Cleanup
            sh 'docker system prune -f'
            
            // Archive artifacts
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            
            // Publish reports
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: 'index.html',
                reportName: 'Test Report'
            ])
        }
        
        success {
            // Notify success
            notifySlack(
                channel: '#devops-notifications',
                color: 'good',
                message: "✅ Pipeline succeeded for ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        
        failure {
            // Notify failure
            notifySlack(
                channel: '#devops-alerts',
                color: 'danger',
                message: "❌ Pipeline failed for ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
            
            // Create incident if production deployment fails
            if (env.BRANCH_NAME == 'main') {
                createServiceNowIncident(
                    summary: "Production deployment failure: ${env.JOB_NAME}",
                    description: "Pipeline failed during production deployment. Build: ${env.BUILD_URL}"
                )
            }
        }
    }
}

// Custom functions
def runTests() {
    sh 'mvn clean test'
    return currentBuild.result
}

def deployToEnvironment(environment, version) {
    sh """
        helm upgrade --install banking-app-${environment} ./helm-chart \\
            --namespace ${environment} \\
            --set image.tag=${version} \\
            --set environment=${environment} \\
            --wait --timeout=600s
    """
}

def deployBlueGreen(environment, version) {
    // Blue-green deployment logic
    sh """
        # Deploy to green environment
        helm upgrade --install banking-app-green ./helm-chart \\
            --namespace ${environment}-green \\
            --set image.tag=${version} \\
            --wait --timeout=600s
        
        # Run smoke tests
        ./scripts/smoke-tests.sh ${environment}-green
        
        # Switch traffic to green
        kubectl patch service banking-app-service \\
            --namespace ${environment} \\
            --patch '{"spec":{"selector":{"version":"green"}}}'
        
        # Wait and verify
        sleep 30
        ./scripts/health-check.sh ${environment}
        
        # Cleanup old blue environment
        helm uninstall banking-app-blue --namespace ${environment}-blue || true
    """
}
```

### Day 3-4: Pipeline Security and Compliance
**Study Topics:**
- Secret management with HashiCorp Vault
- Security scanning integration (SAST, DAST, SCA)
- Compliance automation and reporting
- Audit trails and change tracking

**Financial Services Requirements:**
- SOX compliance for change management
- PCI-DSS for payment processing applications
- Segregation of duties and approval workflows
- Immutable deployment artifacts

**Hands-On Labs:**
1. Integrate HashiCorp Vault for secret management
2. Set up SonarQube for static code analysis
3. Implement OWASP ZAP for dynamic security testing
4. Configure compliance reporting dashboards

### Day 5-7: Advanced Jenkins Features
**Study Topics:**
- Jenkins Pipeline Global Libraries
- Blue Ocean interface and visualization
- Jenkins Configuration as Code (JCasC)
- Integration with enterprise tools (JIRA, ServiceNow)

**Hands-On Labs:**
1. Create shared pipeline libraries
2. Implement Jenkins Configuration as Code
3. Set up Blue Ocean for pipeline visualization
4. Configure JIRA integration for release management

---

## Module 2: GitLab CI/CD and GitHub Actions (Week 2)

### Day 1-2: GitLab CI/CD Enterprise Features
**Study Topics:**
- GitLab CI/CD architecture and runners
- Multi-project pipelines and dependencies
- GitLab Security features and compliance
- Container registry and package management

**Enterprise Capabilities:**
- SAML/LDAP integration
- Compliance pipelines and security scanning
- Merge request approvals and policies
- Advanced CI/CD analytics

**Sample GitLab CI/CD Pipeline:**
```yaml
# .gitlab-ci.yml for banking application
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  SECURE_LOG_LEVEL: debug

stages:
  - validate
  - security
  - build
  - test
  - deploy

# Pre-commit validation
validate:
  stage: validate
  script:
    - echo "Validating code quality and standards"
    - pre-commit run --all-files
    - terraform validate
    - terraform fmt -check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# Security scanning
sast:
  stage: security
  dependencies: []
  artifacts:
    reports:
      sast: gl-sast-report.json
    expire_in: 1 week

dependency_scanning:
  stage: security
  dependencies: []
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
    expire_in: 1 week

# Build and test
build:
  stage: build
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      cat > Dockerfile << EOF
      FROM openjdk:11-jre-slim
      COPY target/banking-app.jar app.jar
      EXPOSE 8080
      ENTRYPOINT ["java", "-jar", "/app.jar"]
      EOF
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
    paths:
      - target/

# Container security scanning
container_scanning:
  stage: security
  dependencies:
    - build
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
    expire_in: 1 week

# Integration tests
integration_tests:
  stage: test
  image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  services:
    - postgres:13
    - redis:6
  variables:
    POSTGRES_DB: banking_test
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    REDIS_URL: redis://redis:6379
  script:
    - mvn test -Dspring.profiles.active=integration
  artifacts:
    reports:
      junit: target/failsafe-reports/TEST-*.xml
    expire_in: 1 week

# Deployment jobs
.deploy_template: &deploy_template
  image: bitnami/kubectl:1.25
  before_script:
    - kubectl config use-context $CLUSTER_CONTEXT
    - helm repo add stable https://charts.helm.sh/stable
    - helm repo update

deploy_dev:
  <<: *deploy_template
  stage: deploy
  script:
    - |
      helm upgrade --install banking-app-dev ./helm-chart \
        --namespace development \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHA \
        --set environment=development \
        --wait --timeout=600s
  environment:
    name: development
    url: https://banking-app-dev.company.com
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy_staging:
  <<: *deploy_template
  stage: deploy
  script:
    - |
      helm upgrade --install banking-app-staging ./helm-chart \
        --namespace staging \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHA \
        --set environment=staging \
        --wait --timeout=600s
  environment:
    name: staging
    url: https://banking-app-staging.company.com
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy_production:
  <<: *deploy_template
  stage: deploy
  script:
    - |
      # Blue-green deployment
      helm upgrade --install banking-app-prod ./helm-chart \
        --namespace production \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHA \
        --set environment=production \
        --set deployment.strategy=blue-green \
        --wait --timeout=600s
      
      # Run smoke tests
      ./scripts/smoke-tests.sh production
  environment:
    name: production
    url: https://banking-app.company.com
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
      allow_failure: false
```

### Day 3-4: GitHub Actions for Enterprise
**Study Topics:**
- GitHub Actions architecture and runners
- Workflow security and secrets management
- Enterprise features and compliance
- Integration with enterprise systems

**Enterprise Security Features:**
- Required workflows and organization policies
- OIDC integration for secure cloud deployments
- Code scanning and secret scanning
- Advanced security and compliance reporting

**Sample GitHub Actions Workflow:**
```yaml
# .github/workflows/banking-app-cicd.yml
name: Banking App CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: read
  packages: write
  security-events: write
  id-token: write

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  build-and-test:
    runs-on: ubuntu-latest
    needs: security-scan
    strategy:
      matrix:
        java-version: [11, 17]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK ${{ matrix.java-version }}
        uses: actions/setup-java@v3
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'

      - name: Cache Maven dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Run tests
        run: mvn clean test

      - name: Generate test report
        uses: dorny/test-reporter@v1
        if: success() || failure()
        with:
          name: Maven Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit

      - name: Build application
        run: mvn clean package -DskipTests

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: banking-app-${{ matrix.java-version }}
          path: target/*.jar

  build-container:
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.event_name != 'pull_request'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan container image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'container-trivy-results.sarif'

      - name: Upload container scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'container-trivy-results.sarif'

  deploy-dev:
    runs-on: ubuntu-latest
    needs: build-container
    if: github.ref == 'refs/heads/develop'
    environment: development
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name banking-cluster-dev

      - name: Deploy to development
        run: |
          helm upgrade --install banking-app-dev ./helm-chart \
            --namespace development \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set environment=development \
            --wait --timeout=600s

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-container
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name banking-cluster-staging

      - name: Deploy to staging
        run: |
          helm upgrade --install banking-app-staging ./helm-chart \
            --namespace staging \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --wait --timeout=600s

      - name: Run integration tests
        run: ./scripts/integration-tests.sh staging

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name banking-cluster-prod

      - name: Blue-Green Deployment
        run: |
          # Deploy to green environment
          helm upgrade --install banking-app-green ./helm-chart \
            --namespace production-green \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --wait --timeout=600s

          # Run smoke tests
          ./scripts/smoke-tests.sh production-green

          # Switch traffic
          kubectl patch service banking-app-service \
            --namespace production \
            --patch '{"spec":{"selector":{"version":"green"}}}'

          # Verify deployment
          sleep 30
          ./scripts/health-check.sh production

      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        with:
          status: success
          channel: '#deployments'
          text: '🚀 Banking app successfully deployed to production'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Day 5-7: Pipeline Integration and Optimization
**Study Topics:**
- Cross-platform pipeline integration
- Performance optimization and caching
- Artifact management and distribution
- Monitoring and observability for pipelines

**Hands-On Labs:**
1. Set up cross-platform pipeline triggers
2. Implement advanced caching strategies
3. Configure artifact repositories (Nexus, Artifactory)
4. Set up pipeline monitoring with Prometheus

---

## Assessment and Capstone Project

### Week 2 Capstone: Enterprise Banking CI/CD Platform
**Requirements:**
1. **Multi-Technology Support:** Java, Python, Node.js applications
2. **Security Integration:** SAST, DAST, SCA, container scanning
3. **Compliance Automation:** SOX, PCI-DSS compliance checks
4. **Multi-Environment:** Dev, staging, production deployments
5. **Approval Workflows:** Segregation of duties for production
6. **Monitoring:** Comprehensive pipeline and application monitoring

**Deliverables:**
- Jenkins master with agent architecture
- GitLab CI/CD enterprise configuration
- GitHub Actions enterprise workflows
- Security scanning integration
- Compliance reporting dashboard
- Performance benchmarks and optimization

**Success Criteria:**
- [ ] Support 50+ concurrent pipeline executions
- [ ] Sub-5 minute build times for typical applications
- [ ] 100% security scan coverage
- [ ] Automated compliance reporting
- [ ] Zero-downtime deployments
- [ ] Comprehensive audit trails

---

## Study Materials and Resources

### Official Documentation
- [Jenkins User Documentation](https://www.jenkins.io/doc/)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Bitbucket Pipelines Documentation](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-bitbucket-pipelines/)

### Books and Guides
- "Continuous Delivery" by Jez Humble and David Farley
- "The DevOps Handbook" by Gene Kim, Patrick Debois, John Willis
- "Jenkins: The Definitive Guide" by John Ferguson Smart
- "GitLab CI/CD Quick Start Guide" by Adam O'Grady

### Online Courses
- Jenkins certification training
- GitLab CI/CD courses on Pluralsight
- GitHub Actions fundamentals
- DevOps Institute CI/CD certification

### Tools and Environments
- Local Jenkins installation
- GitLab.com or self-hosted GitLab
- GitHub Enterprise or GitHub.com
- Docker and Kubernetes for testing
- Cloud providers for deployment targets

---

## Next Steps
Upon completion of this module:
1. **Module 2:** Container Orchestration with Kubernetes
2. **Module 3:** Infrastructure as Code with Terraform
3. **Certification Preparation:** Jenkins Engineer, GitLab CI/CD Specialist
4. **Real-world Application:** Implement enterprise CI/CD pipeline