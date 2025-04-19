    def allServices = [
        'vets-service',
        'customers-service',
        'visits-service',
        'api-gateway',
        'config-server',
        'discovery-server',
        'admin-server'
    ]
pipeline {
    agent { label 'universal-agent' }

    tools {
        jdk 'jdk21'
        maven 'M3'
    }

    environment {
        JAVA_HOME = "${tool 'jdk21'}"
        PATH      = "${env.JAVA_HOME}/bin${isUnix() ? ':' : ';'}${env.PATH}"
    }

    // list all micro‑services here so we can reuse it


    stages {
        stage('📥 Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME ?: 'main'}",
                    url: 'https://github.com/NPT0116/thanh-microservices-spring.git'
            }
        }

        stage('🧬 Detect changed services') {
            when { not { buildingTag() } }  // skip on tags
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

        // ───────────────────────────────────────────────────────────────────
        // New stage: if we’re building a tag like “v1.2.3”, do a full staging build
        // ───────────────────────────────────────────────────────────────────
        stage('🔖 Release / Staging Build') {
            // only run on tags that look like v1.2.3
            when {
                allOf {
     buildingTag()
      expression { env.BRANCH_NAME ==~ /^v\d+\.\d+\.\d+$/ }
                }
            }
            steps {
                script {
                    def tag = env.BRANCH_NAME
                    echo "🎯 Release tag detected: ${tag}"

                    // Docker Hub login
                    withCredentials([
                      usernamePassword(
                        credentialsId: 'dockerhub-login',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                      )
                    ]) {
                        def loginCmd = isUnix()
                          ? "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                          : "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"

                        isUnix() ? sh(loginCmd) : bat(loginCmd)

                        // build & push every service under this tag
                        allServices.each { svc ->
                            def imageName = "npt1601/${svc}:${tag}"
                            echo "🚧 Building ${svc} → ${imageName}"
                            def buildCmd = isUnix()
                              ? "./mvnw -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${imageName}"
                              : "mvnw.cmd -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${imageName}"

                            isUnix() ? sh(buildCmd) : bat(buildCmd)
                            echo "📤 Pushing ${imageName}"
                            isUnix() ? sh("docker push ${imageName}") : bat("docker push ${imageName}")
                        }
                    }
                }
            }
        }

        // ───────────────────────────────────────────────────────────────────
        // (Optional) deploy into staging namespace via kubectl
        // replace or expand with your Helm/ArgoCD logic as desired
        // ───────────────────────────────────────────────────────────────────
        stage('🚀 Deploy to Staging') {
            when { buildingTag() }
            steps {
                script {
                    def tag = env.BRANCH_NAME
                    allServices.each { svc ->
                        // patch the image in the existing Deployment
                        sh """
                          kubectl set image deployment/${svc}-deployment \
                            ${svc}=npt1601/${svc}:${tag} \
                            -n staging
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline complete!"
        }
        failure {
            echo "❌ Something failed."
        }
    }
}
