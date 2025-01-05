pipeline {
    agent any

    environment {

        DOCKER_REPO = '1190990/lmsbooks'

        GIT_REPO_URL = 'https://github.com/ricardopereira87/ODSOFT_P2_1190990_LMSBooks'  // Your Git repository URL
        GIT_BRANCH = 'main'  // Specify the branch to check out
        GIT_BRANCH_PRE = 'preprod'  // Specify the branch to check out
        //CREDENTIALS_ID = 'x'  // Credentials ID for authentication

        SERVER_PORT = '2226'

        IMAGE_NAME = 'lmsbooks'
        IMAGE_TAG = 'latest'

        
    }

    stages {

        stage('Determine Branch') {
            steps {
                script {
                    // Access the globally defined ENVIRONMENT variable
                    def environment = env.ENVIRONMENT
                    echo "Environment Variable: ${environment}"
                    
                    // Set the branch based on environment
                    if (environment == 'preprod') {
                        env.BRANCH = 'preprod'
                    } else if (environment == 'prod') {
                        env.BRANCH = 'main'
                    } else {
                        error "Unknown environment: ${environment}"
                    }

                    // Output the determined branch
                    echo "Branch to be used: ${env.BRANCH}"
                }
            }
        }

        stage('Debug Environment') {
            steps {
                sh 'env'
            }
        }


        stage('Check Docker') {
            steps {
                sh 'docker --version'
            }
        }

        stage('Checkout') {
            steps {
                script {
                    if (env.BRANCH == 'main') {
                        git branch: "${env.BRANCH}",
                            url: "${GIT_REPO_URL}"
                    } else if (env.BRANCH == 'preprod') {
                        git branch: "${env.BRANCH}",
                            url: "${GIT_REPO_URL}"
                    }
                }
            }
        }

        stage('Clean') {
            steps {
                script {
                    sh """
                        mvn clean install
                    """
                }
            }
        }

        stage('Validate') {
            steps {
                script {
                    sh "mvn validate"
                }
            }
        }

        stage('Compile') {
            steps {
                script {
                    sh "mvn compile"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    if (env.BRANCH == 'main') {
                        echo "Tests were performed in preprod stage"
                    } else if (env.BRANCH == 'preprod') {
                        sh "mvn test"
                    }
                }
            }
        }

        stage('Package') {
            steps {
                script {
                    // Package the application
                    sh "mvn package -DskipTests"
                }
            }
        }

        stage('Tag Previous Docker Image') {
            steps {
                script {
                    def imageExists = sh(script: "docker images -q ${IMAGE_NAME}:${IMAGE_TAG}", returnStdout: true).trim()
                    
                    if (imageExists) {
                        sh """
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:rollback
                        """
                    } else {
                        echo "Docker image ${IMAGE_NAME}:${IMAGE_TAG} not found, skipping tag step."
                    }
                }
            }
        }


        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }


        stage('Deploy with Docker Compose') {
            steps {
                script {
                    
                    if (env.BRANCH == 'main') {
                    sh """

                        if ! docker ps --filter "name=rabbitmq_in_lms_network" --format '{{.Names}}' | grep -q rabbitmq_in_lms_network; then
                          docker-compose -f docker-compose-rabbitmq+postgres.yml up -d
                        else
                          echo "RabbitMQ + postgres container already running."
                        fi


                        docker-compose -f docker-compose.yml up -d --force-recreate

                    """
                    } else if (env.BRANCH == 'preprod') {
                       echo "Deploy is done in prod..."
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing image ..."
                    withCredentials([string(credentialsId: 'docker', variable: 'DOCKER_PASSWORD')]) {
                        sh """
                        echo \$DOCKER_PASSWORD | docker login -u 1190990 --password-stdin &&
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REPO}:${IMAGE_TAG} &&
                        docker push ${DOCKER_REPO}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }

////if containers are to be removed
//     post {
//         always {
//             echo 'Cleaning up...'
//             sh """
//                 docker compose -f docker-compose.yml down || true
//             """
//         }
//     }

}
