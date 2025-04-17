// Jenkinsfile
pipeline {
    agent any

    // tools{
    //     jdk 'JDK21'
    // }

    options {
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub'
        DOCKER_REGISTRY        = '22127146'
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                echo 'Cleaning workspace...'
                cleanWs()
                checkout scm
            }
        }

        stage('Initialize') {
            steps {
                script {
                    // Lấy commit ID
                    if (env.GIT_COMMIT) {
                        env.COMMIT_ID = env.GIT_COMMIT.take(7)
                    } else {
                        env.COMMIT_ID = bat(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    }
                    // In thông tin
                    env.BRANCH_NAME = env.BRANCH_NAME
                    echo "Building branch: ${env.BRANCH_NAME}"
                    echo "Commit ID: ${env.COMMIT_ID}"
                    echo "Docker Hub Credentials ID: ${env.DOCKERHUB_CREDENTIALS_ID}"
                    echo "Docker Registry (for image prefix): ${env.DOCKER_REGISTRY}"
                }
            }
        }

        stage('Build and Push Images to Docker Hub') {
            steps {
                script{
                    docker.withRegistry("https://index.docker.io/v1/", env.DOCKERHUB_CREDENTIALS_ID){
                        try{
                            // Xây dựng lệnh Maven - **BỎ --push**, chỉ build và load vào local Docker
                            def mvnCommand = ".\\mvnw.cmd clean install -P buildDocker -DskipTests "+
                                             "-Ddocker.image.prefix=${env.DOCKER_REGISTRY} "+
                                             "-Ddocker.image.tag.commit=${env.COMMIT_ID} "+
                                             "-Dcontainer.platform=\"linux/amd64\" "
                                             "-Dcontainer.build.extraagr=\"--push\""

                            echo "Executing Maven command on Windows to build images: ${mvnCommand}"
                            bat mvnCommand // Thực thi lệnh build

                            echo "Maven build completed successfully."
                        }
                        catch (e) {
                            echo "Error building images via Maven: ${e.getMessage()}"
                            error(message: "Failed to build images via Maven")
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'CI Pipeline finished.'
        }
        success {
             // Cập nhật thông báo thành công
             echo "Successfully built, tagged (${env.COMMIT_ID}), and pushed images for branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed for commit ${env.COMMIT_ID} on branch ${env.BRANCH_NAME}"
        }
    }
}
