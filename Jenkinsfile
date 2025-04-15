// Jenkinsfile
pipeline {
    agent any // Or specify a node with Docker installed: agent { label 'docker-agent' }

    environment {
        // Use the ID you created in Jenkins Credentials for Docker Hub
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-2'
        // Extract Docker Hub username from the credentials
        // This requires the Credentials Binding plugin
        //DOCKERHUB_USERNAME     = credentials(DOCKERHUB_CREDENTIALS_ID).username
        // Define your Docker Hub repository prefix (usually your username)
        //DOCKER_REGISTRY        = "${env.DOCKERHUB_USERNAME}"
    }

    options {
        // Set a timeout for the entire pipeline
        timeout(time: 30, unit: 'MINUTES')
        // Keep a reasonable number of build logs
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Do not allow parallel builds of the same branch
        disableConcurrentBuilds()
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    
                    //env.COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    //env.COMMIT_ID = bat(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    
                    if (env.GIT_COMMIT) {
                        env.COMMIT_ID = env.GIT_COMMIT.take(7)
                    } else {
                         // Cách 2: Dùng bat (Windows)
                        env.COMMIT_ID = bat(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        // Hoặc powershell:
                        // env.COMMIT_ID = powershell(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        // Hoặc sh (Linux/macOS):
                        // env.COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    }

                    // Lấy username từ credentials và định nghĩa DOCKERHUB_USERNAME, DOCKER_REGISTRY
                    //def dockerCreds = credentials(env.DOCKERHUB_CREDENTIALS_ID)
                    //env.DOCKERHUB_USERNAME = dockerCreds.username
                    //env.DOCKER_REGISTRY = "${env.DOCKERHUB_USERNAME}"
                    env.DOCKERHUB_USERNAME = '22127146'
                    env.DOCKER_REGISTRY = '22127146'
                    
                    env.BRANCH_NAME = env.BRANCH_NAME // Already available in Multibranch Pipeline
                    echo "Building branch: ${env.BRANCH_NAME}"
                    echo "Commit ID: ${env.COMMIT_ID}"
                    echo "Docker Registry: ${env.DOCKER_REGISTRY}"

                    // In ra để kiểm tra
                    env.BRANCH_NAME = env.BRANCH_NAME // Already available
                    echo "Building branch: ${env.BRANCH_NAME}"
                    echo "Commit ID: ${env.COMMIT_ID}"
                    echo "Docker Hub Credentials ID: ${env.DOCKERHUB_CREDENTIALS_ID}"
                    echo "Docker Hub Username: ${env.DOCKERHUB_USERNAME}"
                    echo "Docker Registry: ${env.DOCKER_REGISTRY}"

                    // Kiểm tra xem có lấy được username không
                    if (!env.DOCKERHUB_USERNAME) {
                        error("Could not retrieve Docker Hub username from credentials ID: ${env.DOCKERHUB_CREDENTIALS_ID}")
                    }
                }
            }
        }

        stage('Build and Push Microservice Images') {
            // Run builds for different services in parallel for speed
            parallel {
                stage('Build & Push Customers Service') {
                    steps {
                        script {
                            def serviceName = 'customers-service'
                            def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                            buildAndPushImage(serviceName, imageName)
                        }
                    }
                }
                stage('Build & Push Vets Service') {
                    steps {
                        script {
                            def serviceName = 'vets-service'
                            def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                            buildAndPushImage(serviceName, imageName)
                        }
                    }
                }
                stage('Build & Push Visits Service') {
                    steps {
                        script {
                            def serviceName = 'visits-service'
                            def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                            buildAndPushImage(serviceName, imageName)
                        }
                    }
                }
                stage('Build & Push API Gateway') {
                    steps {
                        script {
                            def serviceName = 'api-gateway'
                            def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                            buildAndPushImage(serviceName, imageName)
                        }
                    }
                }
                 stage('Build & Push Config Server') {
                     steps {
                         script {
                             def serviceName = 'config-server'
                             def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                             buildAndPushImage(serviceName, imageName)
                         }
                     }
                 }
                 stage('Build & Push Discovery Server') {
                     steps {
                         script {
                             def serviceName = 'discovery-server'
                             def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                             buildAndPushImage(serviceName, imageName)
                         }
                     }
                 }
                 stage('Build & Push Admin Server') {
                     steps {
                         script {
                             def serviceName = 'admin-server'
                             def imageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${serviceName}:${env.COMMIT_ID}"
                             buildAndPushImage(serviceName, imageName)
                         }
                     }
                 }
                // Add stages for other services (config-server, discovery-server, admin-server) similarly
                // Note: The genai-service is not in the base repo, handle it if you added it separately.
            }
        }
    }

    post {
        // Always run regardless of build status
        always {
            echo 'CI Pipeline finished.'
            // Optional: Clean up workspace, Docker images etc.
            // cleanWs()
        }
        success {
            echo "Successfully built and pushed images for commit ${env.COMMIT_ID} on branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed for commit ${env.COMMIT_ID} on branch ${env.BRANCH_NAME}"
            // Add notifications here (e.g., email, Slack) if desired
        }
    }
}

// Helper function to build and push a Docker image for a specific service
// Assumes each service has a Dockerfile in its root directory
// Assumes Maven wrapper (mvnw) is used for building the JAR
def buildAndPushImage(String serviceName, String imageName) {
    // Construct the directory path based on the standard project structure
    def serviceDir = "spring-petclinic-${serviceName}"

    echo "Building Docker image for ${serviceName}..."
    echo "Image Name: ${imageName}"
    echo "Service Directory: ${serviceDir}"

    // Use docker.withRegistry to handle Docker Hub authentication
    docker.withRegistry("https://index.docker.io/v1/", DOCKERHUB_CREDENTIALS_ID) {
        try {
            // Change directory to the specific microservice
            dir(serviceDir) {
                // Ensure mvnw is executable (sometimes lost on clone)
                sh 'chmod +x ./mvnw'

                // Build the application JAR using Maven wrapper
                // Skip tests for faster CI build, run tests in a separate stage/job if needed
                echo "Building JAR for ${serviceName}..."
                sh './mvnw clean package -DskipTests'

                // Build the Docker image using the Dockerfile in the service directory
                echo "Building Docker image ${imageName}..."
                // The Dockerfile likely copies the JAR from target/, so the JAR build must happen first.
                // Adjust context ('.') and Dockerfile path ('Dockerfile') if needed.
                def builtImage = docker.build(imageName, "--pull -f Dockerfile .") // Use '.' as context within the serviceDir

                // Push the image to Docker Hub
                echo "Pushing Docker image ${imageName}..."
                builtImage.push()
                echo "Successfully pushed ${imageName}"

                // Optional: Push a branch-specific tag as well?
                // builtImage.push("${env.BRANCH_NAME.replace('/', '-')}") // Example: Pushing tag 'feature-xyz'
            }
        } catch (e) {
            echo "Error building/pushing image for ${serviceName}: ${e.getMessage()}"
            // Fail the stage explicitly
            error("Failed to build and push ${serviceName}")
        } finally {
            // Optional: Clean up the built image locally to save space on the agent
             // sh "docker rmi ${imageName}"
        }
    }
}
