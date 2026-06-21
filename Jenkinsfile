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
                                sh 'mvn dependency:copy-dependencies -DoutputDirectory=target/dependency'
                                withSonarQubeEnv('SonarQube') {
                                    sh '''
                                        ${SCANNER_HOME}/bin/sonar-scanner \
                                          -Dsonar.projectKey=mvnwebapp \
                                          -Dsonar.projectName='MVN WebApp' \
                                          -Dsonar.sources=src/main/java \
                                          -Dsonar.java.binaries=target/classes \
                                          -Dsonar.java.libraries=target/dependency/*.jar
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
                            sh '''
                                docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .
                                docker tag $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG $DOCKER_USER/$IMAGE_NAME:latest
                            '''
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
                                  $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG

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
                                docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                                docker push $DOCKER_USER/$IMAGE_NAME:latest
                            '''
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
                            sh '''
                                curl -u $TOMCAT_USER:$TOMCAT_PASS \
                                  -T target/${WAR_NAME}.war \
                                  "$TOMCAT_URL/manager/text/deploy?path=/${WAR_NAME}&update=true"
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
                                docker rmi $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG || true
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
