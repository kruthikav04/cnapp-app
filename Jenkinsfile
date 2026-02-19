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
                    string(credentialsId: 'LW_ACCOUNT_NAME', variable: 'LW_ACCOUNT_NAME'),
                    string(credentialsId: 'LW_API_KEY', variable: 'LW_API_KEY'),
                    string(credentialsId: 'LW_API_SECRET', variable: 'LW_API_SECRET')
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

                    echo "Preparing env.list for IaC scan..."

                    echo "LW_ACCOUNT=$LW_ACCOUNT_NAME" > env.list
                    echo "LW_API_KEY=$LW_API_KEY" >> env.list
                    echo "LW_API_SECRET=$LW_API_SECRET" >> env.list
                    echo "SCAN_COMMAND=k8s-scan" >> env.list
                    echo "WORKSPACE=/app/src" >> env.list
                    echo "SCAN_DIR=k8s" >> env.list

                    echo "Running Lacework IaC Scan (Kubernetes YAML)..."

                    docker run --rm \
                        --env-file env.list \
                        -e EXIT_FLAG=none \
                        -v $(pwd):/app/src \
                        lacework/codesec-iac:stable \
                        lacework iac scan --directory=/app/src/k8s

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
