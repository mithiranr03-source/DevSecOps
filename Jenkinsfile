pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = 'mithiranaws/amazon'
    }

    stages {

        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/mithiranr03-source/DevSecOps.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=amazon \
                    -Dsonar.projectKey=amazon
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh """
                trivy fs . \
                --skip-dirs node_modules,target,.git \
                --severity HIGH,CRITICAL \
                --format table \
                -o trivyfs.txt
                """
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    env.IMAGE_TAG = "${IMAGE_NAME}:${BUILD_NUMBER}"
                    sh "docker build -t ${IMAGE_TAG} ."
                }
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                withCredentials([string(credentialsId: 'docker-cred', variable: 'dockerpwd')]) {
                    sh """
                    docker login -u mithiranaws -p ${dockerpwd}
                    docker push ${IMAGE_TAG}
                    docker tag ${IMAGE_TAG} ${IMAGE_NAME}:latest
                    docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage("Trivy Image Scan") {
            steps {
                sh """
                trivy image \
                --severity HIGH,CRITICAL \
                --format json \
                -o trivy-image.json \
                ${IMAGE_TAG}

                trivy image \
                --severity HIGH,CRITICAL \
                --format table \
                -o trivy-image.txt \
                ${IMAGE_TAG}
                """
            }
        }

        stage("Deploy to Container") {
            steps {
                sh """
                docker rm -f amazon || true
                docker run -d --name amazon -p 80:80 ${IMAGE_TAG}
                """
            }
        }
    }

    post {
        always {
            emailext(
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h3>Amazon DevSecOps Pipeline</h3>
                    <p><b>Status:</b> ${currentBuild.currentResult}</p>
                    <p><b>Build:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'mithiranr03@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivyfs.txt,trivy-image.json,trivy-image.txt'
            )
        }
    }
}

