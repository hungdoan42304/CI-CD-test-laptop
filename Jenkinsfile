pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        IMAGE_NAME = "sample-ci-app"
        REGISTRY = "192.168.232.170/jenkins-ci"

        GITOPS_REPO = "https://github.com/Hungsky123456/CI-CD-test-laptop-gitops.git"
        GITOPS_BRANCH = "main"
        GITOPS_MANIFEST = "k8s/deployment.yaml"
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

        stage('Update GitOps Repository') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-gitops-credentials',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e

                        rm -rf gitops-repo

                        git clone \
                          --branch "$GITOPS_BRANCH" \
                          "https://${GIT_USER}:${GIT_TOKEN}@github.com/Hungsky123456/CI-CD-test-laptop-gitops.git" \
                          gitops-repo

                        cd gitops-repo

                        sed -i \
                          "s|image: 192.168.232.170/jenkins-ci/sample-ci-app:.*|image: $REGISTRY/$IMAGE_NAME:$BUILD_NUMBER|" \
                          "$GITOPS_MANIFEST"

                        echo "Updated manifest:"
                        grep "image:" "$GITOPS_MANIFEST"

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@lab.local"

                        git add "$GITOPS_MANIFEST"

                        if git diff --cached --quiet; then
                            echo "No GitOps manifest change detected."
                        else
                            git commit -m "Deploy sample-ci-app build $BUILD_NUMBER"
                            git push origin "$GITOPS_BRANCH"
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI completed: image pushed to Harbor and GitOps desired state updated.'
        }

        failure {
            echo 'Pipeline failed. Review the console output.'
        }

        always {
            cleanWs()
        }
    }
}
