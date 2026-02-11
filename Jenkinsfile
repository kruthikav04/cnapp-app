pipeline {
    agent any

    environment {
        IMAGE_NAME = "cnappacr2026.azurecr.io/notes-app"
        RESOURCE_GROUP = "Cnapp-RG"
        AKS_CLUSTER = "myAKS-cluster"
        ACR_NAME = "cnappacr2026"
        LACEWORK_ACCOUNT = "719551"
        TENANT_ID = "981439d1-88ac-4c7c-bd5d-d5df66bc0f4c"
        SUBSCRIPTION_ID = "442c2f42-962b-4a21-bea5-db44a51fb466"
    }

    stages {

        stage('Azure Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'azure-sp-creds',
                    usernameVariable: 'AZURE_CLIENT_ID',
                    passwordVariable: 'AZURE_CLIENT_SECRET'
                )]) {

                    sh '''
                    set -e

                    az login --service-principal \
                        --username $AZURE_CLIENT_ID \
                        --password $AZURE_CLIENT_SECRET \
                        --tenant $TENANT_ID

                    az account set --subscription $SUBSCRIPTION_ID
                    '''
                }
            }
        }

        stage('Login to ACR') {
            steps {
                sh '''
                set -e
                az acr login --name $ACR_NAME
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                set -e

                docker build -t $IMAGE_NAME:${BUILD_NUMBER} .
                docker tag $IMAGE_NAME:${BUILD_NUMBER} $IMAGE_NAME:latest
                '''
            }
        }

        stage('Scan Image with Lacework') {
            steps {
                withCredentials([
                    string(credentialsId: 'LACEWORK-ACCESS-KEY', variable: 'LW_ACCESS'),
                    string(credentialsId: 'LACEWORK-SECRET-KEY', variable: 'LW_SECRET')
                ]) {

                    sh '''
                    set -e
                    sleep 5

                    lacework vulnerability container scan \
                        cnappacr2026.azurecr.io \
                        notes-app \
                        ${BUILD_NUMBER} \
                        -a $LACEWORK_ACCOUNT \
                        -k $LW_ACCESS \
                        -s $LW_SECRET
                    '''
                }
            }
        }

        stage('Push Image to ACR') {
            steps {
                sh '''
                set -e

                docker push $IMAGE_NAME:${BUILD_NUMBER}
                docker push $IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy to AKS') {
            steps {
                sh '''
                set -e

                az aks get-credentials \
                    --resource-group $RESOURCE_GROUP \
                    --name $AKS_CLUSTER \
                    --overwrite-existing

                export KUBECONFIG=~/.kube/config

                kubectl set image deployment/notes-app \
                    notes-app=$IMAGE_NAME:${BUILD_NUMBER}

                kubectl rollout status deployment/notes-app
                '''
            }
        }

        stage('Cleanup Docker Images') {
            steps {
                sh '''
                docker system prune -f || true
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
