pipeline {
    agent { label 'cnapp' }  // Your Jenkins agent label
    environment {
        AZURE_SUBSCRIPTION = "Kruthika's-Subscription"
        ACR_NAME = "cnappacr2026"
        RESOURCE_GROUP = "Cnapp-RG"
        AKS_CLUSTER = "myAKS-cluster"
        DEPLOYMENT_FILE = "deployment.yaml"
        SERVICE_FILE = "service.yaml"
        IMAGE_NAME = "${ACR_NAME}.azurecr.io/notes-app"
        IMAGE_TAG = "${BUILD_NUMBER}"  // Dynamic image tag per build
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout([$class: 'GitSCM', 
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/kruthikav04/cnapp-app.git',
                        credentialsId: '79c7b2f1-1414-4f07-b7d4-5919b324b242'
                    ]]
                ])
            }
        }

        stage('Azure Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'AZURE_SP',
                    usernameVariable: 'AZURE_CLIENT_ID',
                    passwordVariable: 'AZURE_CLIENT_SECRET'
                )]) {
                    sh """
                    az login --service-principal --username $AZURE_CLIENT_ID --password $AZURE_CLIENT_SECRET --tenant 981439d1-88ac-4c7c-bd5d-d5df66bc0f4c
                    az account set --subscription "${AZURE_SUBSCRIPTION}"
                    """
                }
            }
        }

        stage('Login to ACR') {
            steps {
                sh "az acr login --name ${ACR_NAME}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push Image to ACR') {
            steps {
                sh """
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                docker push ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to AKS') {
            steps {
                sh """
                # Set kubeconfig
                az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER} --overwrite-existing

                # Check if deployment exists
                if kubectl get deployment notes-app; then
                    # Update image
                    kubectl set image deployment/notes-app notes-app=${IMAGE_NAME}:${IMAGE_TAG} --record
                else
                    # First time deployment
                    kubectl apply -f ${DEPLOYMENT_FILE}
                    kubectl apply -f ${SERVICE_FILE}
                fi
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
