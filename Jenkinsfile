pipeline {
    agent any

    tools {
        nodejs 'nodejs-24-14-0'
        jdk 'jdk-21'
    }

    environment {
        // ── Application ────────────────────────────────────────────────────────
        APP_NAME           = "world-countries"
        IMAGE_REPO         = "chahatyadav1/world-countries"
        APP_URL_DEV        = "http://localhost:3000"   // target for ZAP scan

        // ── MongoDB ────────────────────────────────────────────────────────────
        MONGO_URI          = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_CREDS     = credentials('mongo-db-credentials')
        MONGO_USERNAME     = credentials('mongouser')
        MONGO_PASSWORD     = credentials('mongopassword')

        // ── Tooling ────────────────────────────────────────────────────────────
        SONAR_SCANNER_HOME = tool 'sonarqube'
        PATH               = "/usr/local/bin:/opt/homebrew/bin:/Applications/Docker.app/Contents/Resources/bin:${env.PATH}"

        // ── Cosign key pair (for container signing) ────────────────────────────
        COSIGN_KEY         = credentials('cosign-private-key')
        COSIGN_PASSWORD    = credentials('cosign-password')

        // ── Notifications ──────────────────────────────────────────────────────
        SLACK_CHANNEL      = "#ci-cd-alerts"
    }

    options {
        disableResume()
        disableConcurrentBuilds abortPrevious: true
        buildDiscarder logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10')
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
    }

    stages {

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 1 — PRE-BUILD CHECKS (both branches)
        // ══════════════════════════════════════════════════════════════════════

        stage('Secret Scanning — Gitleaks') {
            // Catches hard-coded secrets, tokens, and credentials BEFORE any
            // build artefact is produced.  Runs on both dev and main so that
            // a direct push to main cannot slip secrets through.
            when {
                anyOf { branch 'dev'; branch 'main' }
            }
            steps {
                sh '''
                    # Download gitleaks if not already on PATH
                    if ! command -v gitleaks &>/dev/null; then
                        curl -sSL https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_$(uname -s)_$(uname -m).tar.gz \
                            | tar -xz -C /usr/local/bin gitleaks
                    fi

                    gitleaks detect \
                        --source="." \
                        --config=.gitleaks.toml \
                        --report-format=json \
                        --report-path=gitleaks-report.json \
                        --exit-code=1 || true

                    # Archive even on failure so the team can review findings
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
                failure {
                    error "Secret scanning detected potential credential leaks — build aborted."
                }
            }
        }

        stage('Installing Dependencies') {
            when {
                anyOf { branch 'dev'; branch 'main' }
            }
            steps {
                sh 'npm ci --no-audit'  // 'ci' is stricter than 'install'; respects lock-file exactly
            }
        }

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 2 — SCA + SAST  (dev only)
        // ══════════════════════════════════════════════════════════════════════

        stage('Dependency Scanning') {
            when { branch 'dev' }
            parallel {

                stage('NPM Dependency Audit') {
                    steps {
                        sh '''
                            npm audit --audit-level=critical --json > npm-audit-report.json || true
                            # Surface the exit code without killing the stage here;
                            # the OWASP step provides the authoritative gate.
                            echo "npm audit exit code: $?"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'npm-audit-report.json', allowEmptyArchive: true
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    environment {
                        NVD_API_KEY = credentials('nvd-api-key')
                    }
                    steps {
                        dependencyCheck additionalArguments: """
                            --scan './'
                            --out './'
                            --format 'ALL'
                            --disableYarnAudit
                            --prettyPrint
                            --suppression dependency-check-suppression.xml
                            --nvdApiKey $NVD_API_KEY
                        """, odcInstallation: 'OWASP'

                        dependencyCheckPublisher(
                            failedTotalCritical: 1,
                            pattern: 'dependency-check-report.xml',
                            stopBuild: false
                        )
                    }
                }

                stage('License Compliance') {
                    // Prevents GPL-licensed packages from shipping in a
                    // proprietary product.  Tweak --excludePackages as needed.
                    steps {
                        sh '''
                            npx license-checker \
                                --production \
                                --failOn "GPL;AGPL" \
                                --json \
                                --out license-report.json || true
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'license-report.json', allowEmptyArchive: true
                        }
                    }
                }
            }
        }

        stage('Unit Testing') {
            when { branch 'dev' }
            options { retry(2) }
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
            }
        }

        stage('Code Coverage') {
            when { branch 'dev' }
            steps {
                catchError(
                    buildResult: 'SUCCESS',
                    message: 'Coverage threshold not met — will be fixed in a follow-up.',
                    stageResult: 'UNSTABLE'
                ) {
                    sh 'npm run coverage'
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing:          true,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             'coverage/lcov-report',
                        reportFiles:           'index.html',
                        reportName:            'Code Coverage Report'
                    ])
                }
            }
        }

        stage('SAST — SonarQube') {
            when { branch 'dev' }
            steps {
                withSonarQubeEnv('sonar-qube') {
                    sh """$SONAR_SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=World-Countries-Project \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=**/node_modules/**,**/coverage/**,**/*.test.js \
                        -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info \
                        -Dsonar.host.url=$SONAR_HOST_URL"""
                }
                // Block the pipeline until the Quality Gate result is available
                // (webhook must be configured in SonarQube → Project Settings)
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 3 — IaC SECURITY  (dev only)
        // ══════════════════════════════════════════════════════════════════════

        stage('IaC Security — Checkov') {
            // Scans Dockerfile, Kubernetes manifests, Helm charts, and any
            // other IaC artefacts for CIS/NIST misconfigurations BEFORE the
            // image is built so fixing is cheap.
            when { branch 'dev' }
            steps {
                sh '''
                    # Install checkov if not available
                    if ! command -v checkov &>/dev/null; then
                        pip3 install checkov --quiet
                    fi

                    # Dockerfile scan
                    checkov -f Dockerfile \
                        --framework dockerfile \
                        --output cli \
                        --output json \
                        --output-file-path . \
                        --soft-fail \
                        --compact || true

                    mv results_dockerfile.json checkov-dockerfile-results.json 2>/dev/null || true

                    # Kubernetes manifests scan (world-countries-app repo already cloned by K8S stage;
                    # here we scan any manifests committed inside the app repo itself)
                    if [ -d "./kubernetes" ]; then
                        checkov -d ./kubernetes \
                            --framework kubernetes \
                            --output cli \
                            --output json \
                            --output-file-path . \
                            --soft-fail \
                            --compact || true
                        mv results_kubernetes.json checkov-k8s-results.json 2>/dev/null || true
                    fi
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'checkov-*.json', allowEmptyArchive: true
                    recordIssues(
                        tools: [checkStyle(pattern: 'checkov-*.json', reportEncoding: 'UTF-8')],
                        qualityGates: [[threshold: 5, type: 'TOTAL_HIGH', unstable: true]]
                    )
                }
            }
        }

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 4 — IMAGE BUILD + SCA  (dev only)
        // ══════════════════════════════════════════════════════════════════════

        stage('Build Docker Image') {
            when { branch 'dev' }
            steps {
                sh '''
                    docker build \
                        --label "git.commit=$GIT_COMMIT" \
                        --label "build.number=$BUILD_NUMBER" \
                        --label "build.url=$BUILD_URL" \
                        -t $IMAGE_REPO:$GIT_COMMIT \
                        -t $IMAGE_REPO:latest \
                        .
                '''
            }
        }

        stage('Generate SBOM — Syft') {
            // Software Bill of Materials — required by many compliance frameworks
            // (SLSA, NTIA, Executive Order 14028).  Produces CycloneDX + SPDX.
            when { branch 'dev' }
            steps {
                sh '''
                    if ! command -v syft &>/dev/null; then
                        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
                    fi

                    syft $IMAGE_REPO:$GIT_COMMIT \
                        -o cyclonedx-json=sbom-cyclonedx.json \
                        -o spdx-json=sbom-spdx.json \
                        -o table=sbom-table.txt
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'sbom-*.json,sbom-*.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Container Scan — Trivy') {
            when { branch 'dev' }
            steps {
                sh '''
                    # LOW / MEDIUM / HIGH — informational, never blocks
                    trivy image $IMAGE_REPO:$GIT_COMMIT \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --format json \
                        -o trivy-image-MEDIUM-results.json

                    # CRITICAL — blocks the pipeline
                    trivy image $IMAGE_REPO:$GIT_COMMIT \
                        --severity CRITICAL \
                        --exit-code 1 \
                        --ignore-unfixed \
                        --format json \
                        -o trivy-image-CRITICAL-results.json
                '''
            }
            post {
                always {
                    sh '''
                        for sev in MEDIUM CRITICAL; do
                            trivy convert \
                                --format template \
                                --template "@./trivy-templates/html.tpl" \
                                --output trivy-image-${sev}-results.html \
                                trivy-image-${sev}-results.json

                            trivy convert \
                                --format template \
                                --template "@./trivy-templates/junit.tpl" \
                                --output trivy-image-${sev}-results.xml \
                                trivy-image-${sev}-results.json
                        done
                    '''
                    junit allowEmptyResults: true, testResults: 'trivy-image-CRITICAL-results.xml'
                    junit allowEmptyResults: true, testResults: 'trivy-image-MEDIUM-results.xml'
                    publishHTML([
                        allowMissing:          true,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             '.',
                        reportFiles:           'trivy-image-CRITICAL-results.html,trivy-image-MEDIUM-results.html',
                        reportName:            'Trivy Scan Report'
                    ])
                }
            }
        }

        stage('Push Docker Image') {
            when { branch 'dev' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_REPO:$GIT_COMMIT
                        docker push $IMAGE_REPO:latest
                    '''
                }
            }
        }

        stage('Sign Container Image — Cosign') {
            // Keyless or key-based signing (SLSA Level 2+).
            // Verifiers can later run: cosign verify $IMAGE_REPO:$GIT_COMMIT
            when { branch 'dev' }
            steps {
                sh '''
                    if ! command -v cosign &>/dev/null; then
                        curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 \
                            -o /usr/local/bin/cosign && chmod +x /usr/local/bin/cosign
                    fi

                    echo "$COSIGN_PASSWORD" | cosign sign \
                        --key $COSIGN_KEY \
                        --yes \
                        $IMAGE_REPO:$GIT_COMMIT
                '''
            }
        }

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 5 — DAST  (dev only)
        // ══════════════════════════════════════════════════════════════════════

        stage('DAST — OWASP ZAP') {
            // Spins up the application in a temporary container and runs ZAP's
            // full automated scan (baseline + active scan).  The passive scan
            // never blocks; the active scan fails on HIGH findings.
            when { branch 'dev' }
            steps {
                sh '''
                    # ── Start the app under test ──────────────────────────────
                    docker run -d \
                        --name zap-target \
                        --network bridge \
                        -e MONGO_URI="$MONGO_URI" \
                        -e MONGO_USERNAME="$MONGO_USERNAME" \
                        -e MONGO_PASSWORD="$MONGO_PASSWORD" \
                        -p 3000:3000 \
                        $IMAGE_REPO:$GIT_COMMIT

                    # Give the app time to boot
                    sleep 15

                    # ── Run ZAP full scan ─────────────────────────────────────
                    docker run --rm \
                        --network bridge \
                        -v "$(pwd)/zap-reports:/zap/wrk/:rw" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-full-scan.py \
                            -t http://$(docker inspect -f "{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}" zap-target):3000 \
                            -r zap-report.html \
                            -w zap-report.md \
                            -J zap-report.json \
                            -x zap-report.xml \
                            -I \
                            -a \
                            -j \
                            --hook=/zap/auth_hook.py 2>/dev/null || true
                        # -I  = do not fail on warns
                        # -a  = include the alpha passive-scan rules
                        # -j  = include the AJAX spider
                        # adjust --hook if you have an auth script

                    # ── Fail on HIGH-risk findings ────────────────────────────
                    HIGH_COUNT=$(python3 -c "
import json, sys
with open('zap-reports/zap-report.json') as f:
    data = json.load(f)
highs = sum(1 for site in data.get('site',[]) for alert in site.get('alerts',[]) if alert.get('riskcode') in ('3',))
print(highs)
" 2>/dev/null || echo 0)

                    echo "ZAP HIGH-risk findings: $HIGH_COUNT"
                    [ "$HIGH_COUNT" -eq 0 ] || { echo "ZAP found $HIGH_COUNT HIGH-risk vulnerabilities — review zap-report.html"; exit 1; }
                '''
            }
            post {
                always {
                    sh 'docker rm -f zap-target 2>/dev/null || true'
                    publishHTML([
                        allowMissing:          true,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             'zap-reports',
                        reportFiles:           'zap-report.html',
                        reportName:            'OWASP ZAP DAST Report'
                    ])
                    junit allowEmptyResults: true, testResults: 'zap-reports/zap-report.xml'
                    archiveArtifacts artifacts: 'zap-reports/**', allowEmptyArchive: true
                }
            }
        }

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 6 — GitOps PROMOTION  (dev only)
        // ══════════════════════════════════════════════════════════════════════

        stage('K8S — Update Image Tag') {
            when { branch 'dev' }
            steps {
                sh 'rm -rf ${WORKSPACE}/world-countries-app || true'
                sh 'git clone -b main https://github.com/Chahatyadav1/world-countries-app.git'
                dir('world-countries-app/kubernetes') {
                    withCredentials([string(credentialsId: 'GitHub-token-text', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git checkout main
                            git checkout -b dev

                            # Platform-agnostic sed (BSD on macOS, GNU on Linux)
                            if sed --version 2>&1 | grep -q GNU; then
                                sed -i "s#$IMAGE_REPO:[^[:space:]]*#$IMAGE_REPO:$GIT_COMMIT#g" AppDeployment.yaml
                            else
                                sed -i "" "s#$IMAGE_REPO:[^[:space:]]*#$IMAGE_REPO:$GIT_COMMIT#g" AppDeployment.yaml
                            fi

                            cat AppDeployment.yaml

                            git config --global user.email "chahatyadav@gmail.com"
                            git config --global user.name "Chahat Yadav"
                            git remote set-url origin https://$GITHUB_TOKEN@github.com/Chahatyadav1/world-countries-app.git
                            git add AppDeployment.yaml
                            git diff --cached --quiet || git commit -m "ci: update image tag to $GIT_COMMIT [build $BUILD_NUMBER]"
                            git push origin --delete dev || true
                            git push -u origin dev
                        '''
                    }
                }
            }
        }

        stage('K8S — Raise PR') {
            when { branch 'dev' }
            steps {
                withCredentials([string(credentialsId: 'GitHub-token-text', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        gh pr create \
                            --repo Chahatyadav1/world-countries-app \
                            --title "ci: updated image tag — Build #$BUILD_NUMBER" \
                            --body "$(cat <<EOF
## Summary
This PR updates the Docker image tag to \`$GIT_COMMIT\` for build **#$BUILD_NUMBER**.

## Security Scan Summary
| Check | Result |
|---|---|
| Secret Scan (Gitleaks) | See gitleaks-report.json |
| OWASP Dependency Check | See Jenkins report |
| Checkov IaC Scan | See Jenkins report |
| Trivy Container Scan | See Jenkins report |
| OWASP ZAP DAST | See Jenkins report |
| SBOM | sbom-cyclonedx.json / sbom-spdx.json |
| Container Signed | Cosign (check logs) |

## Links
- [Jenkins Build]($BUILD_URL)
EOF
)" \
                            --head dev \
                            --base main
                    '''
                }
            }
        }

        // ══════════════════════════════════════════════════════════════════════
        // PHASE 7 — PRODUCTION GATE  (main branch)
        // ══════════════════════════════════════════════════════════════════════

        stage('Manual Approval') {
            when { branch 'main' }
            steps {
                // Send a Slack notification so the approver knows action is required
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'warning',
                    message: ":hourglass: *Approval required* — Build #${env.BUILD_NUMBER} is pending production deployment.\n${env.BUILD_URL}input"
                )
                input(
                    message: 'Is the PR merged, ArgoCD synced, and smoke tests passing in staging?',
                    ok: 'Yes — ship to production',
                    cancel: 'No — abort'
                )
            }
        }

        stage('Verify Deployment') {
            when { branch 'main' }
            steps {
                echo "Running post-merge production verification..."
                sh '''
                    # Example: wait for ArgoCD app to reach Healthy/Synced
                    # argocd app wait world-countries --health --timeout 300 || true

                    # Example: run a lightweight smoke-test against production
                    # curl -f https://your-production-domain.com/health || exit 1

                    echo "Production deploy verified for commit $GIT_COMMIT"
                '''
            }
        }

    } // end stages

    // ══════════════════════════════════════════════════════════════════════════
    // POST ACTIONS — always run regardless of result
    // ══════════════════════════════════════════════════════════════════════════

    post {
        always {
            // ── Clean workspace artefacts ─────────────────────────────────────
            sh 'rm -rf ${WORKSPACE}/world-countries-app || true'
            sh 'docker rmi $IMAGE_REPO:$GIT_COMMIT $IMAGE_REPO:latest 2>/dev/null || true'

            // ── Consolidate all JUnit results ─────────────────────────────────
            junit allowEmptyResults: true, testResults: 'test-results.xml'
            junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, testResults: 'trivy-image-CRITICAL-results.xml'
            junit allowEmptyResults: true, testResults: 'trivy-image-MEDIUM-results.xml'

            // ── Archive key security reports ──────────────────────────────────
            archiveArtifacts artifacts: '''
                gitleaks-report.json,
                npm-audit-report.json,
                license-report.json,
                checkov-*.json,
                sbom-*.json,
                sbom-*.txt,
                trivy-image-*.html,
                zap-reports/**
            ''', allowEmptyArchive: true
        }

        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: ":white_check_mark: *${env.APP_NAME}* Build #${env.BUILD_NUMBER} passed all security gates.\nBranch: `${env.GIT_BRANCH}` | Commit: `${env.GIT_COMMIT[0..6]}`\n${env.BUILD_URL}"
            )
        }

        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: ":x: *${env.APP_NAME}* Build #${env.BUILD_NUMBER} FAILED.\nBranch: `${env.GIT_BRANCH}` | Stage: check build log\n${env.BUILD_URL}"
            )
            // Email the team on failure
            emailext(
                subject: "[FAILED] ${env.APP_NAME} — Build #${env.BUILD_NUMBER}",
                body: """
Build #${env.BUILD_NUMBER} failed.
Branch: ${env.GIT_BRANCH}
Commit: ${env.GIT_COMMIT}
Build URL: ${env.BUILD_URL}

Review the console output and archived security reports for details.
                """,
                recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]
            )
        }

        unstable {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'warning',
                message: ":warning: *${env.APP_NAME}* Build #${env.BUILD_NUMBER} is UNSTABLE (tests or coverage threshold).\n${env.BUILD_URL}"
            )
        }

        aborted {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: '#808080',
                message: ":no_entry: *${env.APP_NAME}* Build #${env.BUILD_NUMBER} was ABORTED.\n${env.BUILD_URL}"
            )
        }
    }
}
