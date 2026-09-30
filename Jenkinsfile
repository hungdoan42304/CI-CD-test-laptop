pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        IMAGE_NAME = "sample-ci-app"
        REGISTRY = "192.168.232.170/jenkins-ci"
        GIT_BRANCH_NAME = "main"
        GIT_MANIFEST = "k8s/deployment.yaml"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Commit') {
            steps {
                script {
                    env.LAST_COMMIT_AUTHOR = sh(
                        script: 'git log -1 --pretty=%an',
                        returnStdout: true
                    ).trim()

                    echo "Last commit author: ${env.LAST_COMMIT_AUTHOR}"

                    if (env.LAST_COMMIT_AUTHOR == 'Jenkins CI') {
                        env.SKIP_CI = 'true'
                        echo 'Commit was created by Jenkins CI. Skipping CI stages.'
                    } else {
                        env.SKIP_CI = 'false'
                    }
                }
            }
        }

        stage('Install dependencies') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

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
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

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
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t $IMAGE_NAME:$BUILD_NUMBER .
                '''
            }
        }

        stage('Push Harbor') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

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

        stage('Update GitOps Manifest') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-ci-cd-credentials laptop',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e

                        sed -i \
                          "s|image: $REGISTRY/$IMAGE_NAME:.*|image: $REGISTRY/$IMAGE_NAME:$BUILD_NUMBER|" \
                          "$GIT_MANIFEST"

                        echo "Updated manifest:"
                        grep "image:" "$GIT_MANIFEST"

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@lab.local"

                        git add "$GIT_MANIFEST"

                        if git diff --cached --quiet; then
                            echo "No manifest change detected."
                        else
                            git commit \
                              -m "Update sample-ci-app to build $BUILD_NUMBER [skip ci]"

                            git push \
                              "https://${GIT_USER}:${GIT_TOKEN}@github.com/hungdoan42304/CI-CD-test-laptop.git" \
                              HEAD:$GIT_BRANCH_NAME
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the console output.'
        }

        always {
            cleanWs()
        }
    }
}
