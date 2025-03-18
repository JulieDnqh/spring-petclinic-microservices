pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Determine Build') {
            steps {
                script {
                    // Lấy danh sách các file đã thay đổi
                    env.CHANGED_FILES = sh(returnStdout: true, script: 'git diff --name-only HEAD^ HEAD').trim()
                    // Xác định service nào cần build/test
                    env.SERVICES_TO_BUILD = determineServices(env.CHANGED_FILES.readLines())
                }
            }
        }

        stage('Test') {
            when {
                expression { env.SERVICES_TO_BUILD != null }
            }
            steps {
                script {
                    if (env.SERVICES_TO_BUILD instanceof String) {
                        // Test 1 service
                        echo "Testing service: ${env.SERVICES_TO_BUILD}"
                        sh "./mvnw -f ${env.SERVICES_TO_BUILD}/pom.xml test"

                        // JUnit report
                        junit(
                            testResults: "${env.SERVICES_TO_BUILD}/target/surefire-reports/*.xml",
                            allowEmptyResults: true
                        )

                    } else {
                        // Test nhiều services
                        echo "Testing services: ${env.SERVICES_TO_BUILD}"
                        for (service in env.SERVICES_TO_BUILD) {
                            echo "Testing: ${service}"
                            sh "./mvnw -f ${service}/pom.xml test"

                            // JUnit report (cho từng service)
                            junit(
                                testResults: "${service}/target/surefire-reports/*.xml",
                                allowEmptyResults: true
                            )
                        }
                    }
                }
            }
        }

        stage('Code Coverage') {
           when {
                expression { env.SERVICES_TO_BUILD != null }
           }
            steps {
                script {
                   if (env.SERVICES_TO_BUILD instanceof String) {
                        // Code coverage cho 1 service
                        echo "Generating code coverage for service: ${env.SERVICES_TO_BUILD}"
                        sh "./mvnw -f ${env.SERVICES_TO_BUILD}/pom.xml org.jacoco:jacoco-maven-plugin:report" // Thay đổi ở đây
                        recordCoverage(
                            tools: [[parser: 'JACOCO', pattern: "${env.SERVICES_TO_BUILD}/target/site/jacoco/**/*.xml"]]
                        )
                    } else {
                        // Code coverage cho nhiều service
                         echo "Generating code coverage for services: ${env.SERVICES_TO_BUILD}"
                         for(service in env.SERVICES_TO_BUILD){
                            echo "Generating code coverage for service: ${service}"
                            sh "./mvnw -f ${service}/pom.xml org.jacoco:jacoco-maven-plugin:report" // Thay đổi ở đây
                            recordCoverage(
                                tools: [[parser: 'JACOCO', pattern: "${service}/target/site/jacoco/**/*.xml"]]
                            )
                        }

                    }
                }
            }
        }

          stage('Build') {
            when {
                expression { env.SERVICES_TO_BUILD != null }
            }
            steps {
                script {
                    if (env.SERVICES_TO_BUILD instanceof String) {
                        // Build 1 service
                        echo "Building service: ${env.SERVICES_TO_BUILD}"
                        sh "./mvnw -f ${env.SERVICES_TO_BUILD}/pom.xml clean install -DskipTests"
                        archiveArtifacts artifacts: "${env.SERVICES_TO_BUILD}/target/*.jar"

                    } else {
                         // Build nhiều services
                        echo "Building services: ${env.SERVICES_TO_BUILD}"
                        for(service in env.SERVICES_TO_BUILD){
                          echo "Buiding ${service}"
                          sh "./mvnw -f ${service}/pom.xml clean install -DskipTests"
                          archiveArtifacts artifacts: "${service}/target/*.jar"

                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Finished pipeline"
        }
    }
}

// Function to determine which services to build/test
def determineServices(changedFiles) {
    println "Changed Files: ${changedFiles}"
    def services = [
        'spring-petclinic-customers-service',
        'spring-petclinic-vets-service',
        'spring-petclinic-visits-service',
        'spring-petclinic-api-gateway',
        'spring-petclinic-admin-server'
    ]
    def servicesToBuild = []

    for (service in services) {
         if (changedFiles.any { it.contains(service) }) {
            servicesToBuild.add(service)
        }
    }

  if (servicesToBuild.isEmpty()) {
        return null
    } else if (servicesToBuild.size() == 1) {
      return servicesToBuild[0]
    }
    else {
        return servicesToBuild
    }
}
