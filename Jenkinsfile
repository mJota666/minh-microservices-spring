pipeline {
    agent { label 'universal-agent' }

    tools {
        jdk 'jdk21'
        maven 'M3'
    }

    environment {
        JAVA_HOME = "${tool 'jdk21'}"
        PATH = "${env.JAVA_HOME}/bin${isUnix() ? ':' : ';'}${env.PATH}"
    }

    stages {
        stage('📥 Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME ?: 'main'}",
                    url: 'https://github.com/NPT0116/thanh-microservices-spring.git'
            }
        }

        stage('🧬 Detect changed services') {
            steps {
                script {
                    def commitId = isUnix()
                        ? sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        : bat(script: "git rev-parse --short HEAD", returnStdout: true).trim().readLines().last().trim()

                    echo "🔍 Commit ID: ${commitId}"

                    // Lấy danh sách file thay đổi
                    def changedFiles = isUnix()
                        ? sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().split("\n")
                        : bat(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().readLines()

                    echo "📂 File thay đổi:\n${changedFiles.join('\n')}"

                    def allServices = [
                        'vets-service',
                        'customers-service',
                        'visits-service',
                        'api-gateway',
                        'config-server',
                        'discovery-server',
                        'admin-server'
                    ]

                    def changedServices = [] as Set

                    allServices.each { svc ->
                        changedFiles.each { file ->
                            if (file.startsWith("spring-petclinic-${svc}/")) {
                                changedServices << svc
                            }
                        }
                    }

                    if (changedServices.isEmpty()) {
                        echo "✅ Không có service nào thay đổi, skip build."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "🧱 Các service thay đổi: ${changedServices.join(', ')}"

                    // Build và push
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-login', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        def loginCmd = isUnix()
                            ? "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                            : "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"

                        isUnix() ? sh(loginCmd) : bat(loginCmd)

                        changedServices.each { svc ->
                            def image = "npt1601/${svc}:${commitId}"
                            def buildCmd = isUnix()
                                ? "./mvnw -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
                                : "mvnw.cmd -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"

                            def pushCmd = "docker push ${image}"

                            echo "🚧 Đang build image cho ${svc} → ${image}"
                            isUnix() ? sh(buildCmd) : bat(buildCmd)

                            echo "📤 Push image lên DockerHub: ${image}"
                            isUnix() ? sh(pushCmd) : bat(pushCmd)
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build thành công!"
                    script {
if ((env.BRANCH_NAME ?: 'main') == 'main') {
    echo "🔁 Đang gọi job update-argoCD-deploy-config..."
    
    build job: 'update-argoCD-deploy-config',
          wait: false, // hoặc true nếu bạn muốn đợi chạy xong
          parameters: [] // có thể truyền params tại đây nếu cần
} else {
    echo "⏭ Không phải branch main, không gọi job update-argoCD-deploy-config."
}

        }
        }
        failure {
            echo "❌ Build thất bại!"
        }
    }
}
