pipeline {
    agent any

    environment {
        DOCKERHUB = credentials('dockerhub-creds')
        IMAGE_NAME = "airoswaraj/notes-app"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "cnapp-cluster"

        // Your Lacework account (tenant) — from https://<ACCOUNT>.lacework.net
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
        stage('Scan Image with Lacework') {
            steps {
                withCredentials([
                    // use the exact credential IDs from your Jenkins
                    string(credentialsId: 'LACEWORK-ACCESS-KEY', variable: 'LW_ACCESS'),
                    string(credentialsId: 'LW_SECRET_KEY', variable: 'LW_SECRET')
                ]) {
                    sh '''

                    sleep 45
                    
                    # Note: lacework CLI expects: <registry> <repository> <tag>
                    # For Docker Hub, registry is "docker.io"
                    registry="docker.io"
                    repository="$IMAGE_NAME"
                    tag="${BUILD_NUMBER}"

                    # run the scan, wait until completed (--poll), fail on severity >= HIGH
                    lacework vulnerability container scan \
                        $registry \
                        $repository \
                        $tag \
                        -a $LACEWORK_ACCOUNT -k $LW_ACCESS -s $LW_SECRET \
                        --poll
                        --details
                    '''
                }
            }
        }       
         
    post {
        always {
            sh 'rm -f $KUBECONFIG'
        }
    }
}
