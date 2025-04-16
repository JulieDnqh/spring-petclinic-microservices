// Jenkinsfile
pipeline {
    agent any

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
            }
        }

        stage('Initialize') {
            steps {
                script {
                    // Lấy commit ID ngắn
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

        stage('Build Images via Maven') {
            steps {
                script{
                    docker.withRegistry("https://index.docker.io/v1/", env.DOCKERHUB_CREDENTIALS_ID){
                        try{
                            echo "Checking for mvnw.cmd in current directory: ${pwd()}"
                            bat 'dir mvnw.cmd' // Kiểm tra file tồn tại

                            // Xây dựng lệnh Maven - **BỎ --push**, chỉ build và load vào local Docker
                            def mvnCommand = ".\\mvnw.cmd clean install -P buildDocker -DskipTests " +
                                             "-Ddocker.image.prefix=${env.DOCKER_REGISTRY} " // Vẫn cần prefix để tên image đúng

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
                // // Chỉ cần login ở đây, không cần push từ Maven
                // withDockerRegistry("https://index.docker.io/v1/", env.DOCKERHUB_CREDENTIALS_ID) {
                //     script {
                //         try{
                //             echo "Checking for mvnw.cmd in current directory: ${pwd()}"
                //             bat 'dir mvnw.cmd' // Kiểm tra file tồn tại

                //             // Xây dựng lệnh Maven - **BỎ --push**, chỉ build và load vào local Docker
                //             def mvnCommand = ".\\mvnw.cmd clean install -P buildDocker -DskipTests " +
                //                              "-Ddocker.image.prefix=${env.DOCKER_REGISTRY} " // Vẫn cần prefix để tên image đúng

                //             echo "Executing Maven command on Windows to build images: ${mvnCommand}"
                //             bat mvnCommand // Thực thi lệnh build

                //             echo "Maven build completed successfully."
                //         }
                //         catch (e) {
                //             echo "Error building images via Maven: ${e.getMessage()}"
                //             error(message: "Failed to build images via Maven")
                //         }
                //     }
                // }
            }
        }

        // --- STAGE MỚI ĐỂ TAG VÀ PUSH ---
        stage('Tag and Push Images') {
            steps {
                script {
                    // Đăng nhập lại để thực hiện tag/push
                    docker.withRegistry("https://index.docker.io/v1/", env.DOCKERHUB_CREDENTIALS_ID) {
                        // Danh sách các artifactId (tên module/service)
                        def services = ['admin-server', 'customers-service', 'vets-service', 'visits-service', 'genai-service', 'config-server', 'discovery-server', 'api-gateway']

                        services.each { service ->
                            // Xây dựng tên image cơ bản (như Maven đã build)
                            def baseImageName = "${env.DOCKER_REGISTRY}/spring-petclinic-${service}"
                            def latestTagImage = "${baseImageName}:latest" // Image Maven đã tạo ra
                            def commitTagImage = "${baseImageName}:${env.COMMIT_ID}" // Image với tag mong muốn

                            try {
                                echo "Processing service: ${service}"

                                echo "Tagging ${latestTagImage} as ${commitTagImage}"
                                docker.image(latestTagImage).tag(commitTagImage)

                                echo "Pushing ${commitTagImage}"
                                docker.image(commitTagImage).push()

                                echo "Successfully tagged and pushed ${commitTagImage}"

                            } catch (e) {
                                echo "Error tagging/pushing image for ${service}: ${e.getMessage()}"
                                error(message: "Failed to tag/push image for ${service}")
                            }
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
