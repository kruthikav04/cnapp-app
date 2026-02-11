pipeline {
    agent { label 'cnapp' }

    environment {
        IMAGE_NAME = "cnappacr2026.azurecr.io/notes-app"
        RESOURCE_GROUP = "Cnapp-RG"
        AKS_CLUSTER = "myAKS-cluster"
        ACR_NAME = "cnappacr2026"
        TENANT_ID = "981439d1-88ac-4c7c-bd5d-d5df66bc0f4c"
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
                    az login --service-principal \
                        --username $AZURE_CLIENT_ID \
                        --password $AZURE_CLIENT_SECRET \
                        --tenant $TENANT_ID
                    az account set --subscription "Kruthika's-Subscription"
                    '''
                }
            }
        }

        stage('Login to ACR') {
            steps {
                sh '''az acr login --name $ACR_NAME'''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:${BUILD_NUMBER} .
                docker tag $IMAGE_NAME:${BUILD_NUMBER} $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Image to ACR') {
            steps {
                sh '''
                docker push $IMAGE_NAME:${BUILD_NUMBER}
                docker push $IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy to AKS') {
            steps {
                sh '''
                az aks get-credentials \
                  --resource-group $RESOURCE_GROUP \
                  --name $AKS_CLUSTER \
                  --overwrite-existing

                if kubectl get deployment notes-app; then
                  echo "Updating existing deployment..."
                  kubectl set image deployment/notes-app notes-app=$IMAGE_NAME:${BUILD_NUMBER}
                else
                  echo "Creating new deployment..."
                  kubectl create deployment notes-app --image=$IMAGE_NAME:${BUILD_NUMBER}
                  kubectl expose deployment notes-app --type=LoadBalancer --port=5000 --target-port=5000
                fi

                kubectl rollout status deployment/notes-app
                '''
            }
        }
    }
}
