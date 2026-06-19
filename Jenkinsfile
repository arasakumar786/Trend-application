pipeline {
    agent any

    environment {
        DOCKERHUB_CREDS = credentials('DockerHub')
        DOCKERHUB_USER  = "arasakumar786"
        IMAGE_NAME      = "trend-app"
        IMAGE_TAG       = "v${BUILD_NUMBER}"
        IMAGE_FULL      = "${DOCKERHUB_USER}/${IMAGE_NAME}"
        AWS_REGION      = "ap-southeast-1"
        CLUSTER_NAME    = "dev-cluster"
        K8S_DIR         = "k8s"
        SLACK_CHANNEL   = "#all-arasan"
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_FULL}:${IMAGE_TAG} ."
            }
        }

        stage('Docker Hub Login & Push') {
            steps {
                sh """
                    echo "${DOCKERHUB_CREDS_PSW}" | docker login -u "${DOCKERHUB_CREDS_USR}" --password-stdin
                    docker tag ${IMAGE_FULL}:${IMAGE_TAG} ${IMAGE_FULL}:latest
                    docker push ${IMAGE_FULL}:${IMAGE_TAG}
                    docker push ${IMAGE_FULL}:latest
                """
            }
        }

        stage('Update Kubeconfig') {
            steps {
                sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}"
            }
        }

        stage('Update Image Tag in Manifest') {
            steps {
                sh """
                    sed -i 's|image:.*|image: ${IMAGE_FULL}:${IMAGE_TAG}|' ${K8S_DIR}/deployment.yaml
                """
            }
        }

        stage('Approval Request') {
            steps {
                script {
                    def causes = currentBuild.getBuildCauses()
                    def triggeredBy = causes ? causes[0].shortDescription : "Unknown"

                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: "#FFFF00",
                        message: """
🚨 Deployment Approval Required

Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Triggered By: ${triggeredBy}
Image: ${IMAGE_FULL}:${IMAGE_TAG}

Review and approve here:
${env.BUILD_URL}
""".stripIndent()
                    )
                }
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Deploy this image to EKS?",
                        ok: "Deploy"
                    )
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                    kubectl apply -f ${K8S_DIR}/deployment.yaml
                    kubectl apply -f ${K8S_DIR}/service.yaml
                    kubectl rollout status deployment/trend-deployment --timeout=120s
                """
            }
        }
    }

    post {
        success {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "good",
                message: """
✅ Application Deployed Successfully

Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Image: ${IMAGE_FULL}:${IMAGE_TAG}

${env.BUILD_URL}
""".stripIndent()
            )
        }

        failure {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "danger",
                message: """
❌ Application Deployment Failed

Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}

${env.BUILD_URL}
""".stripIndent()
            )
        }

        always {
            sh 'docker logout || true'
        }
    }
}
