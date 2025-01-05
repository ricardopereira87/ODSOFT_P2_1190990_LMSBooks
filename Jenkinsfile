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


        stage('Blue-Green Deployment with Docker Compose') {
            steps {
                script {
                    if (env.BRANCH == 'main') {
                        echo "Performing Blue-Green Deployment..."

                        // Create new containers (Blue) while keeping old ones (Green) running
                        sh """
                            docker-compose -f docker-compose.yml -p blue up -d --force-recreate --no-deps
                        """

                        // Wait for Blue containers to become healthy
                        waitUntil {
                            def containerStatus = sh(script: "docker inspect --format='{{.State.Health.Status}}' ${IMAGE_NAME}_blue_in_lms_network", returnStdout: true).trim()
                            return containerStatus == 'healthy'
                        }

                        // Once new containers are healthy, stop the old containers (Green)
                        sh """
                            docker-compose -f docker-compose.yml -p green stop
                        """

                        echo "Blue-Green Deployment successful!"
                    } else if (env.BRANCH == 'preprod') {
                        echo "Preprod deployment done in main branch."
                    }
                }
            }
        }


        stage('Verify Deployment') {
            steps {
                script {
                    
                    if (env.BRANCH == 'main') {
                        echo "Verifying deployment on 'main' branch..."

                        // Check if the container is running
                        def containerStatus = sh(script: "docker ps --filter 'name=${IMAGE_NAME}' --format '{{.Status}}'", returnStdout: true).trim()
                        echo "Container Status: ${containerStatus}"

                        // If the container is not running or in an error state, perform rollback
                        if (containerStatus ==~ /.*Exited.*/) {
                            echo "Container failed on 'main' branch. Rolling back..."
                            sh """
                                docker tag ${IMAGE_NAME}:rollback ${IMAGE_NAME}:${IMAGE_TAG}
                                docker-compose -f docker-compose.yml up -d --force-recreate
                            """
                            error "Deployment failed on 'main'. Rolled back to the previous image."
                        } else {
                            echo "Container is running fine on 'main'."
                        }
                    } else if (env.BRANCH == 'preprod') {
                        echo "Skipping rollback check for 'preprod' branch."
                    } else {
                        error "Unknown branch: ${env.BRANCH}. No rollback performed."
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


        stage('Notify Admin for Approval') {
            when {
                expression {
                    env.BRANCH == 'preprod'
                }
            }
            steps {
                script {
                    echo "Sending email notification to admin for branch: ${env.BRANCH}..."
                    emailext (
                        subject: 'Preprod Deployment Successful - Approval Needed for Merge',
                        body: 'The preprod deployment has been successful. Please confirm to merge preprod into main.',
                        to: '1190990@isep.ipp.pt' 
                    )
                }
            }
        }

        stage('Wait for Admin Confirmation') {
            when {
                expression {
                    env.BRANCH == 'preprod'
                }
            }
            steps {
                input message: 'Approve merge of preprod into main?', ok: 'Yes, Merge'
            }
        }

        stage('Merge Preprod into Main') {
            when {
                expression {
                    env.BRANCH == 'preprod'
                }
            }
            steps {
                script {
                    echo "Merging 'preprod' into main..."
                    sshagent(['github-ssh']) {
                        sh """
                        git remote set-url origin git@github.com:ricardopereira87/ODSOFT_P2_1190990_LMSBooks.git
                        git checkout main
                        git pull origin main
                        git merge origin/preprod
                        git push origin main
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
