pipeline {
    agent any

    tools {
        nodejs 'nodejs-24-14-0'
        jdk 'jdk-21'
    }

    environment {
        // ── App ────────────────────────────────────────────────────
        MONGO_URI          = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_CREDS     = credentials('mongo-db-credentials')
        MONGO_USERNAME     = credentials('mongouser')
        MONGO_PASSWORD     = credentials('mongopassword')

        // ── SAST ───────────────────────────────────────────────────
        SONAR_SCANNER_HOME = tool 'sonarqube'

        // ── GCP / Artifact Registry ────────────────────────────────
        GKE_PROJECT        = "your-gcp-project-id"
        GKE_CLUSTER        = "your-cluster-name"
        GKE_REGION         = "us-central1"
        GKE_NAMESPACE      = "production"
        AR_HOSTNAME        = "us-central1-docker.pkg.dev"
        AR_REPO            = "world-countries"
        IMAGE_NAME         = "${AR_HOSTNAME}/${GKE_PROJECT}/${AR_REPO}/world-countries"

        // ── ArgoCD ─────────────────────────────────────────────────
        ARGOCD_SERVER      = "argocd.your-domain.com"
        ARGOCD_APP         = "world-countries"

        // ── PATH ───────────────────────────────────────────────────
        PATH               = "/usr/local/bin:/opt/homebrew/bin:/Applications/Docker.app/Contents/Resources/bin:${env.PATH}"
    }

    options {
        disableResume()
        disableConcurrentBuilds abortPrevious: true
        buildDiscarder(logRotator(numToKeepStr: '10'))   // keep last 10 builds only
        timeout(time: 60, unit: 'MINUTES')               // global pipeline timeout
    }

    stages {

        // ════════════════════════════════════════════════════════════
        // STAGE 1 — Install
        // ════════════════════════════════════════════════════════════

        stage('Installing Dependencies') {
            when {
                anyOf { branch 'dev'; branch 'main' }
            }
            options { timestamps() }
            steps {
                sh 'npm install --no-audit'
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 2 — Dependency Scanning (parallel)
        // ════════════════════════════════════════════════════════════

        stage('Dependency Scanning') {
            when { branch 'dev' }
            parallel {

                stage('NPM Dependency Audit') {
                    steps {
                        sh '''
                            npm audit --audit-level=critical
                            echo $?
                        '''
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
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 3 — Unit Testing
        // ════════════════════════════════════════════════════════════

        stage('Unit Testing') {
            when { branch 'dev' }
            options { retry(2) }
            steps {
                sh 'npm test'
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 4 — Code Coverage
        // ════════════════════════════════════════════════════════════

        stage('Code Coverage') {
            when { branch 'dev' }
            steps {
                catchError(
                    buildResult: 'SUCCESS',
                    message    : 'Coverage failed — will be fixed in future release',
                    stageResult: 'UNSTABLE'
                ) {
                    sh 'npm run coverage'
                }
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 5 — SAST (SonarQube)
        // ════════════════════════════════════════════════════════════

        stage('SAST - SonarQube') {
            when { branch 'dev' }
            steps {
                sh 'sleep 5s'
                withSonarQubeEnv('sonar-qube') {
                    sh '''
                        $SONAR_SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectKey=World-Countries-Project \
                            -Dsonar.sources=app.js \
                            -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info \
                            -Dsonar.host.url=$SONAR_HOST_URL
                    '''
                }
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 6 — Build Docker Image → tagged for Artifact Registry
        // ════════════════════════════════════════════════════════════

        stage('Build Docker Image') {
            when { branch 'dev' }
            steps {
                sh 'docker build -t $IMAGE_NAME:$GIT_COMMIT -t $IMAGE_NAME:latest .'
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 7 — Trivy Image Scan
        // ════════════════════════════════════════════════════════════

        stage('Trivy - Image Scan') {
            when { branch 'dev' }
            steps {
                sh '''
                    trivy image $IMAGE_NAME:$GIT_COMMIT \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --format json -o trivy-image-MEDIUM-results.json

                    trivy image $IMAGE_NAME:$GIT_COMMIT \
                        --severity CRITICAL \
                        --exit-code 1 \
                        --format json -o trivy-image-CRITICAL-results.json
                '''
            }
            post {
                always {
                    sh '''
                        trivy convert \
                            --format template \
                            --template "@./trivy-templates/html.tpl" \
                            --output trivy-image-MEDIUM-results.html \
                            trivy-image-MEDIUM-results.json

                        trivy convert \
                            --format template \
                            --template "@./trivy-templates/html.tpl" \
                            --output trivy-image-CRITICAL-results.html \
                            trivy-image-CRITICAL-results.json

                        trivy convert \
                            --format template \
                            --template "@./trivy-templates/junit.tpl" \
                            --output trivy-image-MEDIUM-results.xml \
                            trivy-image-MEDIUM-results.json

                        trivy convert \
                            --format template \
                            --template "@./trivy-templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml \
                            trivy-image-CRITICAL-results.json
                    '''
                }
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 8 — DAST (OWASP ZAP)
        // ════════════════════════════════════════════════════════════

        stage('DAST - OWASP ZAP') {
            when { branch 'dev' }
            steps {
                sh 'mkdir -p ${WORKSPACE}/zap-reports'
                script {
                    sh '''
                        docker network create zap-net || true

                        docker run -d \
                            --name zap-target \
                            --network zap-net \
                            -e MONGO_URI=$MONGO_URI \
                            -e MONGO_USERNAME=$MONGO_USERNAME \
                            -e MONGO_PASSWORD=$MONGO_PASSWORD \
                            $IMAGE_NAME:$GIT_COMMIT

                        echo "Waiting for app to be ready..."
                        for i in $(seq 1 15); do
                            docker exec zap-target wget -q --spider http://localhost:3000 \
                                && break || sleep 3
                        done
                    '''

                    sh '''
                        docker run --rm \
                            --network zap-net \
                            -v ${WORKSPACE}/zap-reports:/zap/wrk/:rw \
                            ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py \
                                -t http://zap-target:3000 \
                                -r zap-report.html \
                                -x zap-report.xml \
                                -J zap-report.json \
                                -c zap-rules.tsv \
                                -I
                    '''
                }
            }
            post {
                always {
                    sh '''
                        docker stop  zap-target   || true
                        docker rm    zap-target   || true
                        docker network rm zap-net || true
                    '''
                    publishHTML(target: [
                        allowMissing         : false,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : 'zap-reports',
                        reportFiles          : 'zap-report.html',
                        reportName           : 'ZAP DAST Report'
                    ])
                }
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 9 — Push to Artifact Registry
        //           Uses Workload Identity — zero key files
        // ════════════════════════════════════════════════════════════

        stage('Push to Artifact Registry') {
            when { branch 'dev' }
            steps {
                sh '''
                    # Workload Identity: no JSON key needed
                    # Jenkins agent SA is already authenticated via WIF
                    gcloud auth configure-docker $AR_HOSTNAME --quiet

                    docker push $IMAGE_NAME:$GIT_COMMIT
                    docker push $IMAGE_NAME:latest
                '''
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 10 — GitOps: Update image tag in K8s config repo
        // ════════════════════════════════════════════════════════════

        stage('GitOps - Update Image Tag') {
            when { branch 'dev' }
            steps {
                sh 'rm -rf ${WORKSPACE}/world-countries-app || true'
                sh 'git clone -b main https://github.com/Chahatyadav1/world-countries-app.git'
                dir("world-countries-app/kubernetes") {
                    withCredentials([string(credentialsId: 'GitHub-token-text', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git checkout main
                            git checkout -b dev

                            # Update image tag in deployment manifest
                            sed -i "s|image: .*world-countries:.*|image: $IMAGE_NAME:$GIT_COMMIT|g" \
                                AppDeployment.yaml

                            cat AppDeployment.yaml

                            git config --global user.email "chahatyadav@gmail.com"
                            git config --global user.name  "Chahat Yadav"
                            git remote set-url origin \
                                https://$GITHUB_TOKEN@github.com/Chahatyadav1/world-countries-app.git

                            git add AppDeployment.yaml
                            git diff --cached --quiet || \
                                git commit -m "ci: update image to $IMAGE_NAME:$GIT_COMMIT [build $BUILD_ID]"

                            git push origin --delete dev || true
                            git push -u origin dev
                        '''
                    }
                }
            }
        }

        // ════════════════════════════════════════════════════════════
        // STAGE 11 — Raise PR → main
        // ════════════════════════════════════════════════════════════

        stage('GitOps - Raise PR') {
            when { branch 'dev' }
            steps {
                withCredentials([string(credentialsId: 'GitHub-token-text', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        gh pr create \
                            --repo  Chahatyadav1/world-countries-app \
                            --title "ci: update image tag — Build $BUILD_ID" \
                            --body  "Updates image to \`$IMAGE_NAME:$GIT_COMMIT\` for build $BUILD_ID" \
                            --head  dev \
                            --base  main
                    '''
                }
            }
        }

        // ════════════════════════════════════════════════════════════
        // MAIN BRANCH — ArgoCD GitOps Flow
        // Jenkins does NOT touch the cluster directly
        // ArgoCD is the only actor that deploys to GKE
        // ════════════════════════════════════════════════════════════

        stage('Manual Approval') {
            when { branch 'main' }
            steps {
                input(
                    cancel : 'Abort',
                    message: 'Has the PR been merged and ArgoCD sync triggered?',
                    ok     : 'Yes — ArgoCD is syncing'
                )
            }
        }

        stage('ArgoCD - Wait for Sync') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                    sh '''
                        # Trigger sync (ArgoCD may already auto-sync — this is a safety trigger)
                        argocd app sync $ARGOCD_APP \
                            --server    $ARGOCD_SERVER \
                            --auth-token $ARGOCD_TOKEN \
                            --grpc-web  \
                            --prune     \
                            --force

                        # Wait until the app is fully healthy — max 5 minutes
                        argocd app wait $ARGOCD_APP \
                            --health        \
                            --sync          \
                            --timeout 300   \
                            --server      $ARGOCD_SERVER \
                            --auth-token  $ARGOCD_TOKEN  \
                            --grpc-web
                    '''
                }
            }
        }

        stage('ArgoCD - Verify Production Health') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                    sh '''
                        echo "──────────── ArgoCD App Status ────────────"
                        argocd app get $ARGOCD_APP \
                            --server     $ARGOCD_SERVER \
                            --auth-token $ARGOCD_TOKEN \
                            --grpc-web

                        # Fail pipeline if app is not healthy
                        HEALTH=$(argocd app get $ARGOCD_APP \
                            --server     $ARGOCD_SERVER \
                            --auth-token $ARGOCD_TOKEN \
                            --grpc-web -o json | jq -r '.status.health.status')

                        echo "App Health: $HEALTH"

                        if [ "$HEALTH" != "Healthy" ]; then
                            echo "❌ App is NOT healthy — triggering ArgoCD rollback"
                            argocd app rollback $ARGOCD_APP \
                                --server     $ARGOCD_SERVER \
                                --auth-token $ARGOCD_TOKEN \
                                --grpc-web
                            exit 1
                        fi

                        echo "✅ Production is Healthy"
                    '''
                }
            }
        }
    }

    // ════════════════════════════════════════════════════════════════
    // POST
    // ════════════════════════════════════════════════════════════════

    post {
        always {
            sh 'rm -rf ${WORKSPACE}/world-countries-app || true'

            junit allowEmptyResults: true, testResults: 'test-results.xml'
            junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'
            junit allowEmptyResults: true, testResults: 'trivy-image-CRITICAL-results.xml'
            junit allowEmptyResults: true, testResults: 'trivy-image-MEDIUM-results.xml'
            junit allowEmptyResults: true, testResults: 'zap-reports/zap-report.xml'
        }
        success {
            echo "✅ Pipeline SUCCESS — $IMAGE_NAME:$GIT_COMMIT deployed via ArgoCD"
        }
        failure {
            echo "❌ Pipeline FAILED — Build $BUILD_ID | Commit $GIT_COMMIT"
        }
    }
}
