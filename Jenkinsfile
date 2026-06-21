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
    }

    stages {

        // ═══════════════════════════════════════════════════════
        // PHASE 1 — FAST FEEDBACK (runs on every branch/commit)
        // ═══════════════════════════════════════════════════════

        stage('Build & Unit Test') {
            steps {
                sh 'mvn -B clean package'
            }
        }

        stage('Fast Code Check') {
            when {
                not { changeRequest() }   // skip on PRs, full scan happens there instead
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
                    steps {
                        writeFile file: 'Dockerfile', text: '''
                            FROM tomcat:10-jdk17-temurin
                            COPY target/*.war /usr/local/tomcat/webapps/
                        '''
                    }
                }

                stage('Docker Build') {
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/dockerhub',
                                secretValues: [
                                    [envVar: 'DOCKER_USER', vaultKey: 'username']
                                ]
                            ]]
                        ) {
                            script {
                                env.IMAGE_BRANCH_TAG = (env.BRANCH_NAME == 'master') ? IMAGE_TAG : "rc-${IMAGE_TAG}"
                            }
                            sh '''
                                docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_BRANCH_TAG .
                            '''
                            script {
                                if (env.BRANCH_NAME == 'master') {
                                    sh "docker tag $DOCKER_USER/$IMAGE_NAME:$IMAGE_BRANCH_TAG $DOCKER_USER/$IMAGE_NAME:latest"
                                }
                            }
                        }
                    }
                }

                stage('Trivy Security Scan') {
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
                                  $DOCKER_USER/$IMAGE_NAME:$IMAGE_BRANCH_TAG

                                cat trivy-report.txt
                            '''
                        }
                    }
                }

                stage('Docker Push') {
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
                                docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_BRANCH_TAG
                            '''
                            script {
                                if (env.BRANCH_NAME == 'master') {
                                    sh "docker push $DOCKER_USER/$IMAGE_NAME:latest"
                                }
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
                                kubectl rollout status deployment/mvnwebapp -n ${K8S_NAMESPACE} --timeout=120s

                                rm -f /tmp/kubeconfig-${BUILD_NUMBER}
                            '''
                            echo "Deployed to Kubernetes namespace: ${env.K8S_NAMESPACE}"
                        }
                    }
                }

                stage('Deploy to Tomcat') {
                    steps {
                        withVault(
                            vaultSecrets: [[
                                path: 'secret/jenkins/tomcat',
                                secretValues: [
                                    [envVar: 'TOMCAT_USER', vaultKey: 'username'],
                                    [envVar: 'TOMCAT_PASS', vaultKey: 'password'],
                                    [envVar: 'TOMCAT_URL',  vaultKey: 'url']
                                ]
                            ]]
                        ) {
                            script {
                                env.DEPLOY_PATH = (env.BRANCH_NAME == 'master') ? WAR_NAME : "${WAR_NAME}-staging"
                            }
                            sh '''
                                curl -u $TOMCAT_USER:$TOMCAT_PASS \
                                  -T target/${WAR_NAME}.war \
                                  "$TOMCAT_URL/manager/text/deploy?path=/${DEPLOY_PATH}&update=true"
                                echo "Deployed to: $TOMCAT_URL/${DEPLOY_PATH}/"
                            '''
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
