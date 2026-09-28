pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        IMAGE_NAME = "sample-ci-app"
        REGISTRY = "192.168.232.170/jenkins-ci"
        K8S_NAMESPACE = "cicd-lab"
        K8S_DEPLOYMENT = "sample-ci-app"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh '''
                    docker run --rm \
                      -v /home/dk/jenkins-ci-lab/jenkins_home/workspace/$JOB_NAME:/app \
                      -w /app \
                      node:20-alpine npm install
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker run --rm \
                      -v /home/dk/jenkins-ci-lab/jenkins_home/workspace/$JOB_NAME:/app \
                      -w /app \
                      node:20-alpine npm test
                '''
            }
        }

        stage('Build Docker image') {
            steps {
                sh '''
                    docker build \
                      -t $IMAGE_NAME:$BUILD_NUMBER .
                '''
            }
        }

        stage('Push Harbor') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-jenkins-credentials',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASS'
                    )
                ]) {
                    sh '''
                        echo "$REGISTRY_PASS" | docker login \
                          $REGISTRY \
                          -u "$REGISTRY_USER" \
                          --password-stdin

                        docker tag \
                          $IMAGE_NAME:$BUILD_NUMBER \
                          $REGISTRY/$IMAGE_NAME:$BUILD_NUMBER

                        docker push \
                          $REGISTRY/$IMAGE_NAME:$BUILD_NUMBER

                        docker logout $REGISTRY
                    '''
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'k8s-jenkins-kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        echo "Deploying image:"
                        echo "$REGISTRY/$IMAGE_NAME:$BUILD_NUMBER"

                        kubectl -n $K8S_NAMESPACE set image \
                          deployment/$K8S_DEPLOYMENT \
                          sample-ci-app=$REGISTRY/$IMAGE_NAME:$BUILD_NUMBER

                        kubectl -n $K8S_NAMESPACE rollout status \
                          deployment/$K8S_DEPLOYMENT \
                          --timeout=120s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD completed: image pushed to Harbor and deployed to Kubernetes.'
        }

        failure {
            echo 'Pipeline failed. Review the console output.'
        }

        always {
            cleanWs()
        }
    }
}
