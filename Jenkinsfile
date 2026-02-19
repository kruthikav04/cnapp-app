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

        // ✅ Lacework Stage (Image + IaC Scan)
        stage('Lacework Scan') {
            steps {
                withCredentials([
                    string(credentialsId: 'LW_ACCESS_TOKEN', variable: 'LW_ACCESS_TOKEN'),
                    string(credentialsId: 'LW_ACCOUNT_NAME', variable: 'LW_ACCOUNT_NAME')
                ]) {
                    sh '''
                    echo "Downloading Lacework inline scanner..."
                    curl -L https://github.com/lacework/lacework-vulnerability-scanner/releases/latest/download/lw-scanner-linux-amd64 -o lw-scanner
                    chmod +x lw-scanner

                    echo "Running Lacework Image Scan..."

                    ./lw-scanner image evaluate \
                        $IMAGE_NAME \
                        ${BUILD_NUMBER} \
                        --account-name $LW_ACCOUNT_NAME \
                        --access-token $LW_ACCESS_TOKEN \
                        --build-id ${BUILD_NUMBER} \
                        --build-plan cnapp-app \
                        --ci-build \
                        --save \
                        --fail-on-violation-exit-code 0

                    echo "Running Lacework IaC Scan (Kubernetes YAML)..."

                    docker run --rm \
                        -e LW_ACCOUNT_NAME=$LW_ACCOUNT_NAME \
                        -e LW_ACCESS_TOKEN=$LW_ACCESS_TOKEN \
                        -e LW_BUILD_ID=${BUILD_NUMBER} \
                        -e LW_BUILD_PLAN=cnapp-app \
                        -e LW_CI_BUILD=true \
                        -v $(pwd):/workspace \
                        lacework/codesec:latest \
                        scan iac  /workspace/k8s

                    echo "Lacework scans completed"
                    '''
                }
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
