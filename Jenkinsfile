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
                sh 'az acr login --name $ACR_NAME'
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
 
        stage('Lacework Scan') {
            steps {
                withCredentials([
                    string(credentialsId: 'LW_API_KEY', variable: 'LW_API_KEY'),
                    string(credentialsId: 'LW_API_SECRET', variable: 'LW_API_SECRET')
                ]) {
                    sh '''
                    set +e
                    echo "Triggering Lacework container scan..."
                    lacework vulnerability container scan \
                      cnappacr2026.azurecr.io \
                      notes-app \
                      ${BUILD_NUMBER} \
                      --account 719551 \
                      --api_key $LW_API_KEY \
                      --api_secret $LW_API_SECRET \
                      --details \
                      --poll
                    echo "Lacework scan completed ✅"
                    exit 0
                    '''
                }
            }
        }
        // ✅ EXTRA STAGE (FIXED)
        stage('Lacework Plugin Scan') {
            steps {
                lacework(
                    imageName: "${IMAGE_NAME}",
                    imageTag: "${BUILD_NUMBER}",
                    fixableOnly: false,
                    customFlags: "",
                    noPull: false,
                    evaluatePolicies: true,
                    saveToLacework: true,
                    disableLibraryPackageScanning: false,
                    showEvaluationExceptions: true,
                    tags: ""
                )
            }
        }
        stage('Deploy to AKS') {
            steps {
                sh '''
                az aks get-credentials \
                  --resource-group $RESOURCE_GROUP \
                  --name $AKS_CLUSTER \
                  --overwrite-existing
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
                if kubectl get deployment notes-app; then
                  kubectl set image deployment/notes-app notes-app=$IMAGE_NAME:${BUILD_NUMBER}
                fi
                kubectl rollout status deployment/notes-app
                '''
            }
        }
    }
}
