pipeline {
    agent any

    environment {
        IMAGE_NAME = "sample-ci-app"
        CONTAINER_NAME = "sample-ci-app"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'docker run --rm -v "$WORKSPACE":/app -w /app node:20-alpine npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'docker run --rm -v "$WORKSPACE":/app -w /app node:20-alpine npm test'
            }
        }

        stage('Build Docker image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
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
