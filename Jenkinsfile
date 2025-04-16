// Jenkinsfile
pipeline {
    // Chỉ định agent Linux của bạn
    agent { label 'huong-agent' } // Đảm bảo label này đúng với agent Ubuntu của bạn

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub'
        // Bạn CÓ THỂ lấy username nếu cần, nhưng hãy đảm bảo Credentials Binding Plugin được cài đặt
        // DOCKERHUB_USERNAME     = credentials(DOCKERHUB_CREDENTIALS_ID).username
        DOCKER_REGISTRY        = '22127146' // Tên Docker Hub username hoặc organization của bạn
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                echo 'Cleaning workspace...'
                // Xóa workspace để đảm bảo môi trường sạch
                cleanWs()
                // Checkout code từ SCM đã cấu hình
                echo 'Checking out source code...'
                checkout scm
                // Quan trọng: Đảm bảo script mvnw có quyền thực thi trên Linux
                echo 'Setting execute permission for mvnw script...'
                sh 'chmod +x mvnw'
            }
        }

        stage('Initialize') {
            steps {
                script {
                    // Lấy 7 ký tự đầu của commit hash
                    // Biến env.GIT_COMMIT thường được cung cấp bởi Jenkins sau khi checkout scm
                    if (env.GIT_COMMIT) {
                        env.COMMIT_ID = env.GIT_COMMIT.take(7)
                    } else {
                        // Nếu env.GIT_COMMIT không có sẵn, thử lấy bằng lệnh git trực tiếp
                        echo 'env.GIT_COMMIT not found, trying git command...'
                        // Sử dụng sh cho Linux
                        env.COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    }

                    // In ra các biến môi trường để kiểm tra
                    env.BRANCH_NAME = env.BRANCH_NAME // Jenkins tự cung cấp
                    echo "Building branch: ${env.BRANCH_NAME}"
                    echo "Commit ID: ${env.COMMIT_ID}"
                    echo "Docker Hub Credentials ID: ${env.DOCKERHUB_CREDENTIALS_ID}"
                    echo "Docker Registry (for image prefix): ${env.DOCKER_REGISTRY}"

                    // Kiểm tra xem COMMIT_ID có giá trị không
                    if (!env.COMMIT_ID) {
                       error("Could not determine Commit ID.")
                    }
                }
            }
        }

        // Stage này bị comment, giữ nguyên hoặc xóa nếu không dùng
        // stage('Build and Push Microservice Images') { ... }

        stage('Build and Push All Images via Maven') {
            steps {
                script {
                    // Đăng nhập vào Docker Hub sử dụng credentials đã lưu trong Jenkins
                    docker.withRegistry("https://index.docker.io/v1/", env.DOCKERHUB_CREDENTIALS_ID) {
                        try {
                            echo "Current directory before Maven build: ${pwd()}"
                            // Kiểm tra sự tồn tại của mvnw (tùy chọn)
                            echo "Checking for mvnw presence:"
                            sh 'ls -l mvnw' // Sử dụng ls trên Linux

                            // Xây dựng lệnh Maven cho Linux
                            // Sử dụng ./mvnw thay vì .\\mvnw.cmd
                            def mvnCommand = "./mvnw clean install -P buildDocker -DskipTests " +
                                             "-Ddocker.image.prefix=${env.DOCKER_REGISTRY} " +
                                             "-Ddocker.image.tag=${env.COMMIT_ID} " +
                                             "-Dcontainer.build.extraarg=\"--push\" " + // Đẩy image sau khi build
                                             "-Dcontainer.platform=\"linux/amd64\"" // Chỉ định platform (hữu ích cho cross-compiling, nhưng có thể không cần nếu agent là linux/amd64)

                            echo "Executing Maven command on Linux: ${mvnCommand}"
                            // Sử dụng sh để chạy lệnh trên Linux
                            sh mvnCommand
                        }
                        catch (e) {
                            echo "Error building/pushing images via Maven: ${e.getMessage()}"
                            // Gây lỗi stage một cách rõ ràng
                            error(message: "Failed to build and push images via Maven: ${e.getMessage()}")
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'CI Pipeline finished.'
            // Có thể thêm các bước dọn dẹp khác ở đây nếu cần
            // cleanWs() // Có thể dọn dẹp lại workspace nếu muốn
        }
        success {
            echo "Successfully built and pushed images for commit ${env.COMMIT_ID} on branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed for commit ${env.COMMIT_ID} on branch ${env.BRANCH_NAME}"
            // Thêm thông báo lỗi (email, Slack,...) nếu muốn
        }
    }
}

// Helper function này đang bị comment ở stage gọi nó, nhưng nếu dùng thì cũng cần sửa
// def buildAndPushImage(String serviceName, String imageName) { ... }
// Nếu bạn dùng lại hàm này, hãy đảm bảo bên trong nó cũng sử dụng sh và ./mvnw
/*
// Ví dụ sửa hàm buildAndPushImage cho Linux:
def buildAndPushImage(String serviceName, String imageName) {
    def serviceDir = "spring-petclinic-${serviceName}"
    echo "Building Docker image for ${serviceName}..."
    echo "Image Name: ${imageName}"
    echo "Service Directory: ${serviceDir}"

    docker.withRegistry("https://index.docker.io/v1/", DOCKERHUB_CREDENTIALS_ID) {
        try {
            dir(serviceDir) {
                // Đảm bảo mvnw trong thư mục con cũng có quyền thực thi
                sh 'chmod +x mvnw'
                // Sử dụng sh và ./mvnw
                sh "./mvnw clean install -P buildDocker -DskipTests " +
                   "-Ddocker.image.prefix=${env.DOCKER_REGISTRY} " +
                   "-Ddocker.image.tag=${env.COMMIT_ID} " +
                   "-Dcontainer.build.extraarg=\"--push\" " +
                   "-Dcontainer.platform=\"linux/amd64\""
            }
        } catch (e) {
            echo "Error building/pushing image for ${serviceName}: ${e.getMessage()}"
            error("Failed to build and push ${serviceName}: ${e.getMessage()}")
        } finally {
            // Tùy chọn: Xóa image local để tiết kiệm dung lượng agent
            // try { sh "docker rmi ${imageName}" } catch(any) { echo "Could not remove image ${imageName}"}
        }
    }
}
*/
