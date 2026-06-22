pipeline {
    agent { label 'linux' }

    tools {
        maven 'Maven3'
    }

    environment {
        SCANNER_HOME = tool 'SonarScanner'
        IMAGE_NAME   = 'mvnwebapp'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        WAR_NAME     = 'mvnwebapp'
        GIT_SHA      = "${GIT_COMMIT[0..6]}"   // short SHA, e.g. "a1b2c3d"
    }

    stages {

        // ═══════════════════════════════════════════════════════
        // PHASE 1 — FAST FEEDBACK (runs on every branch/commit)
        // ═══════════════════════════════════════════════════════

        stage('Build & Unit Test') {
            when {
                not { branch 'master' }
            }
            steps {
                sh 'mvn -B clean package'
            }
        }

        stage('Fast Code Check') {
            when {
                allOf {
                    not { changeRequest() }
                    not { branch 'master' }
                    not { branch pattern: 'release/.*', comparator: 'REGEXP' }
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=mvnwebapp \
                          -Dsonar.projectName='MVN WebApp' \
                          -Dsonar.sources=src/main/webapp \
                          -Dsonar.qualitygate.wait=false
                    '''
                }
            }
        }

        // ═══════════════════════════════════════════════════════
        // PHASE 2 — PULL REQUEST GATE (runs only on PRs)
        // ═══════════════════════════════════════════════════════

        stage('Full Quality & Security Gate') {
            when {
                changeRequest()   // only true when triggered by a PR
            }
            stages {
                stage('Quality & Security Scans') {
                    parallel {
                        stage('SonarQube Full Scan') {
                            steps {
                                withSonarQubeEnv('SonarQube') {
                                    sh '''
                                        ${SCANNER_HOME}/bin/sonar-scanner \
                                          -Dsonar.projectKey=mvnwebapp \
                                          -Dsonar.projectName='MVN WebApp' \
                                          -Dsonar.sources=src/main/webapp
                                    '''
                                }
                            }
                        }
                        stage('Snyk Dependency Scan') {
                            steps {
                                withVault(
                                    vaultSecrets: [[
                                        path: 'secret/jenkins/snyk',
                                        secretValues: [
                                            [envVar: 'SNYK_TOKEN', vaultKey: 'token']
                                        ]
                                    ]]
                                ) {
                                    sh '''
                                        snyk auth $SNYK_TOKEN
                                        snyk test --severity-threshold=high || true
                                    '''
                                }
                            }
                        }
                    }
                }

                stage('Quality Gate') {
                    steps {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage('Validate Docker Build') {
                    steps {
                        writeFile file: 'Dockerfile', text: '''
                            FROM tomcat:10-jdk17-temurin
                            RUN apk update 2>/dev/null || true
                            COPY target/*.war /usr/local/tomcat/webapps/
                        '''
                        sh 'docker build -t mvnwebapp:pr-validation-${BUILD_NUMBER} .'
                        sh 'docker rmi mvnwebapp:pr-validation-${BUILD_NUMBER} || true'
                    }
                }
            }
        }

        // ═══════════════════════════════════════════════════════
        // PHASE 3 — RELEASE/MASTER PIPELINE (build, scan, deploy)
        // ═══════════════════════════════════════════════════════

        stage('Release Pipeline') {
            when {
                anyOf {
                    branch 'master'
                    branch 'release/*'
                }
            }
            stages {

                stage('Create Dockerfile') {
                    when {
                        branch pattern: 'release/.*', comparator: 'REGEXP'
                    }
                    steps {
                        writeFile file: 'Dockerfile', text: '''
                            FROM tomcat:10-jdk17-temurin
                            COPY target/*.war /usr/local/tomcat/webapps/
                        '''
                    }
                }

                stage('Docker Build (release only)') {
                    when {
                        branch pattern: 'release/.*', comparator: 'REGEXP'
                    }
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/dockerhub',
                                secretValues: [
                                    [envVar: 'DOCKER_USER', vaultKey: 'username']
                                ]
                            ]]
                        ) {
                            sh '''
                                docker build -t $DOCKER_USER/$IMAGE_NAME:rc-$GIT_SHA .
                            '''
                        }
                    }
                }

                stage('Trivy Security Scan (release only)') {
                    when {
                        branch pattern: 'release/.*', comparator: 'REGEXP'
                    }
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/dockerhub',
                                secretValues: [
                                    [envVar: 'DOCKER_USER', vaultKey: 'username']
                                ]
                            ]]
                        ) {
                            sh '''
                                trivy image --severity HIGH,CRITICAL \
                                  --timeout 15m \
                                  --exit-code 0 \
                                  --format table \
                                  -o trivy-report.txt \
                                  $DOCKER_USER/$IMAGE_NAME:rc-$GIT_SHA

                                cat trivy-report.txt
                            '''
                        }
                    }
                }

                stage('Docker Push RC (release only)') {
                    when {
                        branch pattern: 'release/.*', comparator: 'REGEXP'
                    }
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/dockerhub',
                                secretValues: [
                                    [envVar: 'DOCKER_USER', vaultKey: 'username'],
                                    [envVar: 'DOCKER_PASS', vaultKey: 'password']
                                ]
                            ]]
                        ) {
                            sh '''
                                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                                docker push $DOCKER_USER/$IMAGE_NAME:rc-$GIT_SHA
                            '''
                        }
                        script {
                            env.IMAGE_BRANCH_TAG = "rc-${GIT_SHA}"
                        }
                    }
                }

                stage('Promote RC to Production (master only)') {
                    when {
                        branch 'master'
                    }
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/dockerhub',
                                secretValues: [
                                    [envVar: 'DOCKER_USER', vaultKey: 'username'],
                                    [envVar: 'DOCKER_PASS', vaultKey: 'password']
                                ]
                            ]]
                        ) {
                            sh '''
                                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                                # GIT_PREVIOUS_COMMIT = the commit that was on release/1.0
                                # before this merge commit landed on master.
                                # That commit's short SHA is what release/1.0 tagged as rc-<sha>.
                                MERGE_PARENT_SHA=$(git log -1 --pretty=%P HEAD | awk '{print $2}' | cut -c1-7)

                                echo "Looking for release candidate: rc-$MERGE_PARENT_SHA"

                                docker pull $DOCKER_USER/$IMAGE_NAME:rc-$MERGE_PARENT_SHA

                                docker tag $DOCKER_USER/$IMAGE_NAME:rc-$MERGE_PARENT_SHA $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                                docker tag $DOCKER_USER/$IMAGE_NAME:rc-$MERGE_PARENT_SHA $DOCKER_USER/$IMAGE_NAME:latest

                                docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                                docker push $DOCKER_USER/$IMAGE_NAME:latest

                                echo "IMAGE_BRANCH_TAG=$IMAGE_TAG" > image-tag.env
                            '''
                            script {
                                env.IMAGE_BRANCH_TAG = env.IMAGE_TAG
                            }
                        }
                    }
                }

                stage('Deploy to Kubernetes') {
                    steps {
                        script {
                            env.K8S_NAMESPACE = (env.BRANCH_NAME == 'master') ? 'prod' : 'dev'
                        }
                        withVault(
                            vaultSecrets: [
                                [
                                    path: 'secret/jenkins/kubeconfig',
                                    secretValues: [[envVar: 'KUBECONFIG_CONTENT', vaultKey: 'config']]
                                ],
                                [
                                    path: 'secret/jenkins/dockerhub',
                                    secretValues: [[envVar: 'DOCKER_USER', vaultKey: 'username']]
                                ]
                            ]
                        ) {
                            sh '''
                                echo "$KUBECONFIG_CONTENT" > /tmp/kubeconfig-${BUILD_NUMBER}
                                export KUBECONFIG=/tmp/kubeconfig-${BUILD_NUMBER}

                                sed "s|__DOCKER_USER__|$DOCKER_USER|g; s|__IMAGE_TAG__|$IMAGE_BRANCH_TAG|g" \
                                  k8s/deployment-${K8S_NAMESPACE}.yaml > k8s-final-${K8S_NAMESPACE}.yaml

                                kubectl apply -f k8s-final-${K8S_NAMESPACE}.yaml
                                kubectl rollout status deployment/mvnwebapp -n ${K8S_NAMESPACE} --timeout=300s

                                rm -f /tmp/kubeconfig-${BUILD_NUMBER}
                            '''
                            echo "Deployed to Kubernetes namespace: ${env.K8S_NAMESPACE}"
                        }
                    }
                }

                stage('Cleanup Local Images') {
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/dockerhub',
                                secretValues: [
                                    [envVar: 'DOCKER_USER', vaultKey: 'username']
                                ]
                            ]]
                        ) {
                            sh '''
                                docker rmi $DOCKER_USER/$IMAGE_NAME:$IMAGE_BRANCH_TAG || true
                                docker rmi $DOCKER_USER/$IMAGE_NAME:latest || true
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.txt, target/*.war', allowEmptyArchive: true
        }
        success {
            echo "✅ Pipeline succeeded on branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline failed on branch: ${env.BRANCH_NAME} - check console output"
        }
    }
}
