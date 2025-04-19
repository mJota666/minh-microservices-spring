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

  stages {
    // ────────────────────────────────────────────────────────────
    // Stage MỚI: Nếu build này là một release tag (vd: 1.2.3) thì
    // build & push image cho tất cả các service với tag đó
    // ────────────────────────────────────────────────────────────
    stage('🚀 Release build: build & push all services') {
      when {
        expression {
          // Giả sử bạn đặt tên tag dạng digit.digit.digit (ví dụ 1.2.3)
          return env.BRANCH_NAME ==~ /\d+\.\d+\.\d+/
        }
      }
      steps {
        script {
          def releaseTag = env.BRANCH_NAME   // e.g. "1.2.3"
          echo "🔖 Detected release tag ${releaseTag}, building ALL services..."

          def services = [
            'discovery-server','config-server','admin-server',
            'api-gateway','customers-service','visits-service',
            'vets-service','genai-service'
          ]

          // Docker login
          withCredentials([usernamePassword(
             credentialsId: 'dockerhub-login',
             usernameVariable: 'DOCKER_USER',
             passwordVariable: 'DOCKER_PASS'
          )]) {
            if (isUnix()) {
              sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
            } else {
              bat "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
            }
          }

          // Build & push mỗi service với tag releaseTag
          services.each { svc ->
            def image = "npt1601/${svc}:${releaseTag}"
            echo "🛠 Building ${image}"
            if (isUnix()) {
              sh "./mvnw -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
              sh "docker push ${image}"
            } else {
              bat "mvnw.cmd -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
              bat "docker push ${image}"
            }
          }

          // Sau khi build & push xong, có thể trigger job CD/staging ở đây
          echo "✅ All services built and pushed with tag ${releaseTag}"
        }
      }
    }

    // ────────────────────────────────────────────────────────────
    // Stage nguyên bản: chỉ build những service thay đổi
    // Chỉ chạy khi không phải release-tag
    // ────────────────────────────────────────────────────────────
    stage('🧬 Detect changed services') {
      when {
        expression {
          return !(env.BRANCH_NAME ==~ /\d+\.\d+\.\d+/)
        }
      }
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
      echo "✅ Build pipeline hoàn tất!"
      script {
        // Nếu đây là branch main chứ không phải tag, trigger tiếp job ArgoCD
        if ((env.BRANCH_NAME ?: 'main') == 'main' && !(env.BRANCH_NAME ==~ /\d+\.\d+\.\d+/)) {
          build job: 'update-argoCD-deploy-config', wait: false
        }
      }
    }
    failure {
      echo "❌ Có lỗi trong quá trình build!"
    }
  }
}
