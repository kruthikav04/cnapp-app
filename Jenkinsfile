pipeline {
    agent any

    environment {
        DOCKERHUB = credentials('dockerhub-creds')
        IMAGE_NAME = "airoswaraj/notes-app"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "cnapp-cluster"

        LACEWORK_ACCOUNT = "719551"
    }

    stages {

        stage('Docker Login') {
            steps {
                sh '''
                echo $DOCKERHUB_PSW | docker login -u $DOCKERHUB_USR --password-stdin
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:${BUILD_NUMBER} .
                '''
            }
        }

        // ⭐⭐⭐ MOST IMPORTANT STAGE ⭐⭐⭐
        stage('Scan Image with Lacework') {
            steps {
                withCredentials([
                    string(credentialsId: 'LACEWORK-ACCESS-KEY', variable: 'LW_ACCESS'),
                    string(credentialsId: 'LW_SECRET_KEY', variable: 'LW_SECRET')
                ]) {

                    sh '''

                    export LW_ACCOUNT=$LACEWORK_ACCOUNT
                    export LW_API_KEY=$LW_ACCESS
                    export LW_API_SECRET=$LW_SECRET

                    # Scan container BEFORE push
                    lacework vulnerability container scan \
                        $IMAGE_NAME:${BUILD_NUMBER} \
                        --fail-on-high
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                docker push $IMAGE_NAME:${BUILD_NUMBER}
                '''
            }
        }

        stage('Configure Kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                  --region $AWS_REGION \
                  --name $CLUSTER_NAME \
                  --kubeconfig $KUBECONFIG
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                kubectl set image deployment/notes-app \
                  notes-app=$IMAGE_NAME:${BUILD_NUMBER}

                kubectl rollout status deployment/notes-app
                '''
            }
        }
    }

    post {
        always {
            sh '''
            rm -f $KUBECONFIG
            '''
        }
    }
}
