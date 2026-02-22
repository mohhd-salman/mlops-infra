pipeline {
    agent any

    options {
        timestamps()
        ansiColor('xterm')
    }

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'GitHub Repository URL')
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'GitHub Branch Name')
        string(name: 'DEPLOYMENT_ID', defaultValue: '', description: 'Unique Deployment ID')
        password(name: 'GITHUB_TOKEN', defaultValue: '', description: 'User GitHub token (temporary)')
    }

    environment {
        GCP_PROJECT = "${env.GCP_PROJECT}"
        GKE_CLUSTER = "${env.GKE_CLUSTER}"
        GKE_LOCATION = "${env.GKE_LOCATION}"
        GAR_HOST = "${env.GAR_HOST}"
        GAR_REPO = "${env.GAR_REPO}"
        
        PIPELINE_REPO_URL = "${env.PIPELINE_REPO_URL}"
        
        REPO_NAME = sh(script: """echo "${params.REPO_URL}" | sed 's/\\.git\$//' | xargs basename""", returnStdout: true).trim()
        DOCKER_IMAGE_LOCAL = "${REPO_NAME}:${params.BRANCH_NAME}"
        IMAGE_TAG = "${params.DEPLOYMENT_ID}"
        FINAL_IMAGE_URL = "${GAR_HOST}/${GCP_PROJECT}/${GAR_REPO}/${REPO_NAME}:${IMAGE_TAG}"

        DEPLOY_NAME = "deploy-${params.DEPLOYMENT_ID}".toLowerCase()
        SVC_NAME = "svc-${params.DEPLOYMENT_ID}".toLowerCase()
        APP_LABEL = "app-${params.DEPLOYMENT_ID}".toLowerCase()
        IMAGE = "${FINAL_IMAGE_URL}"
    }

    stages {
        stage('Validate Inputs') {
            steps {
                script {
                    if (!params.REPO_URL?.trim()) error("REPO_URL is required")
                    if (!params.DEPLOYMENT_ID?.trim()) error("DEPLOYMENT_ID is required")
                    BUILD_LOG_MESSAGE = 'Parameters validated.'
                }
            }
        }

        stage('Clone User Repo') {
            steps {
                script {
                    try {
                        sh """
                            rm -rf ${REPO_NAME}
                            git clone https://${params.GITHUB_TOKEN}@${params.REPO_URL.replace('https://', '')} ${REPO_NAME}
                            cd ${REPO_NAME}
                            git checkout ${params.BRANCH_NAME}
                        """
                        BUILD_LOG_MESSAGE = 'Repo cloned.'
                    } catch (e) {
                        BUILD_LOG_MESSAGE = "Clone failed: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }

        stage('Check Dockerfile') {
            steps {
                dir("${REPO_NAME}") {
                    script {
                        if (!fileExists('Dockerfile')) {
                            BUILD_LOG_MESSAGE = "No Dockerfile found."
                            error(BUILD_LOG_MESSAGE)
                        }
                        BUILD_LOG_MESSAGE = "Dockerfile found."
                    }
                }
            }
        }

        stage('Skip Build Image & Push') {
            steps {
                echo "Skipping build and push steps for placeholder testing"
            }
        }

        stage('Test Placeholder Replacement') {
            steps {
                script {
                    echo "Testing placeholder replacement for deployment.yaml"

                    // Displaying variables
                    echo "DEPLOY_NAME: ${DEPLOY_NAME}"
                    echo "SVC_NAME: ${SVC_NAME}"
                    echo "APP_LABEL: ${APP_LABEL}"
                    echo "IMAGE: ${IMAGE}"

                    // Clean any previous rendered files from /tmp/
                    sh "rm -f /tmp/deployment.rendered.yaml"

                    // Running sed command on the deployment.yaml file
                    sh """
                        sed -e "s|\${MODEL_DEPLOYMENT_NAME}|${DEPLOY_NAME}|g" \
                            -e "s|\${MODEL_SERVICE_NAME}|${SVC_NAME}|g" \
                            -e "s|\${MODEL_APP_LABEL}|${APP_LABEL}|g" \
                            -e "s|\${PLACEHOLDER_IMAGE}|${IMAGE}|g" \
                            platform-manifests/k8s/deployment.yaml > /tmp/deployment.rendered.yaml
                        cat /tmp/deployment.rendered.yaml
                    """
                }
            }
        }

        stage('Fetch Service Endpoint') {
            steps {
                script {
                    echo "Skipping this stage for testing placeholder replacement only."
                }
            }
        }
    }

    post {
        success {
            script {
                def payload = [
                    deployment_id: params.DEPLOYMENT_ID,
                    status: 'success',
                    build_number: env.BUILD_NUMBER,
                    build_log: BUILD_LOG_MESSAGE,
                    jenkins_job_name: env.JOB_NAME
                ]

                httpRequest(
                    url: "${env.BACKEND_URL}/deployments/jenkins/callback",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: groovy.json.JsonOutput.toJson(payload)
                )
            }
        }

        failure {
            script {
                def payload = [
                    deployment_id: params.DEPLOYMENT_ID,
                    status: 'failed',
                    build_number: env.BUILD_NUMBER,
                    build_log: BUILD_LOG_MESSAGE,
                    jenkins_job_name: env.JOB_NAME
                ]

                httpRequest(
                    url: "${env.BACKEND_URL}/deployments/jenkins/callback",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: groovy.json.JsonOutput.toJson(payload)
                )
            }
        }

        always {
            script {
                echo "Cleaning up after testing."
                sh "docker system prune -a -f || true"
            }
        }
    }
}