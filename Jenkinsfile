pipeline {
    agent any

    environment {
        IMAGE_NAME = "sample-ci-app"
        CONTAINER_NAME = "sample-ci-app"
	REGISTRY = "192.168.232.170/jenkins-ci"
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
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
            }
        }

	stage('Push Registry') {
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

        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f $CONTAINER_NAME || true

                    docker run -d \
                      --name $CONTAINER_NAME \
                      -p 3000:3000 \
                      --restart unless-stopped \
                      $IMAGE_NAME:$BUILD_NUMBER
                '''
            }
        }
    }

    post {
        success {
            echo 'Build and deployment completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the console output.'
        }
    }
}
