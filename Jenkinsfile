pipeline {
    agent any

    environment {
        VENV_DIR = "${WORKSPACE}/venv"
        SONARQUBE = 'SonarQubeServer'
        scannerHome = tool 'SonarQubeScanner'
        IMAGE_NAME = 'ashish142/your-app-name'
    }

    stages {
        stage('Setup Python') {
            steps {
                sh 'python3 -m venv venv'
                sh './venv/bin/pip install --upgrade pip'
                sh './venv/bin/pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'mkdir -p reports'
                sh './venv/bin/pytest --junitxml=reports/test-results.xml'
                sh './venv/bin/coverage.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                script {
                    sh '''
                        mkdir -p trivy-reports

                        docker run --rm \
                            -v $(pwd):/project \
                            -v trivy-cache:/root/.cache/ \
                            aquasec/trivy:latest \
                            fs /project \
                            --exit-code 1 \
                            --severity HIGH,CRITICAL \
                            --format table \
                            --output /project/trivy-reports/fs-scan.txt
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-reports/fs-scan.txt', fingerprint: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def GIT_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "${IMAGE_NAME}:${BUILD_NUMBER}-${GIT_COMMIT}"
                    echo "Building Docker image: ${env.IMAGE_TAG}"
                    sh "docker build -t ${env.IMAGE_TAG} ."
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    sh 'mkdir -p trivy-reports'

                    def trivyStatus = sh(
                        script: """
                            docker run --rm \
                                -v ${env.WORKSPACE}:/workspace \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v trivy-cache:/root/.cache/ \
                                aquasec/trivy:latest image ${env.IMAGE_TAG} \
                                --exit-code 1 \
                                --severity HIGH,CRITICAL \
                                --format table \
                                --output /workspace/trivy-reports/image-scan.txt

                            docker run --rm \
                                -v ${env.WORKSPACE}:/workspace \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v trivy-cache:/root/.cache/ \
                                aquasec/trivy:latest image ${env.IMAGE_TAG} \
                                --exit-code 0 \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output /workspace/trivy-reports/image-scan.json
                        """,
                        returnStatus: true
                    )

                    if (trivyStatus != 0) {
                        def userInput = input(
                            message: "Trivy found HIGH or CRITICAL vulnerabilities in the Docker image. Proceed anyway?",
                            parameters: [
                                choice(name: 'Continue?', choices: ['No', 'Yes'], description: 'Select whether to proceed')
                            ]
                        )
                        if (userInput == 'No') {
                            error "Aborting pipeline due to Trivy scan failures."
                        } else {
                            echo "User approved to proceed despite Trivy vulnerabilities."
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-reports/image-scan.*', fingerprint: true
                }
            }
        }

        stage('Approval Before Push') {
            steps {
                input message: "Trivy scan passed (or was approved). Proceed with Docker image push?"
                echo "User approved Docker push"
                // Add your docker push or tag logic here
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying image ${env.IMAGE_TAG}..."
                // Add your deployment commands here
            }
        }
    }

    post {
        always {
            junit 'reports/test-results.xml'

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '',
                reportFiles: 'coverage.xml',
                reportName: 'Coverage Report'
            ])
        }
    }
}
