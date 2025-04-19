pipeline {
  agent any

  tools {
    jdk 'jdk21'
    maven 'M3'
  }

  environment {
    JAVA_HOME = "${tool 'jdk21'}"
    PATH      = "${env.JAVA_HOME}/bin${isUnix()?':':';'}${env.PATH}"
  }

  stages {

    // === 1) Checkout bất kỳ branch hoặc tag nào ===
    stage('📥 Checkout') {
      steps {
        checkout scm
      }
    }

    // === 2) CI: Build & Push chỉ service thay đổi ===
    stage('🧬 CI: Build Changed Services') {
      when { not { buildingTag() } }
      steps {
        script {
          // 2.1. Lấy commitId ngắn của HEAD
          def commitId = isUnix()
            ? sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
            : bat(script: "git rev-parse --short HEAD", returnStdout: true).readLines().last().trim()

          echo "🔍 CI Commit ID: ${commitId}"

          // 2.2. Tìm file thay đổi giữa 2 commit
          def changedFiles = (isUnix()
            ? sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true)
            : bat(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true)
          ).trim().split('\n')

          // 2.3. Danh sách tất cả service
          def allSvcs = [
            'vets-service','customers-service','visits-service',
            'api-gateway','config-server','discovery-server','admin-server'
          ]
          def toBuild = allSvcs.findAll { svc ->
            changedFiles.any { it.startsWith("spring-petclinic-${svc}/") }
          }
          if (!toBuild) {
            echo "✅ No services changed → skip CI build."
            return
          }
          echo "🚧 CI will build: ${toBuild}"

          // 2.4. Build & push images cho từng service thay đổi
          withCredentials([usernamePassword(
            credentialsId: 'dockerhub-login',
            usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS'
          )]) {
            def login = isUnix()
              ? "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
              : "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
            isUnix()? sh(login) : bat(login)

            toBuild.each { svc ->
              def image = "npt1601/${svc}:${commitId}"
              echo "🔨 Building ${svc}"
              if (isUnix()) {
                sh "./mvnw -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
                sh "docker push ${image}"
              } else {
                bat "mvnw.cmd -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
                bat "docker push ${image}"
              }
            }
          }
        }
      }
    }

    // === 3) CI on main: trigger job cập nhật Dev ArgoCD ===
    stage('🔁 CI: Trigger Dev-CD') {
      when {
        allOf {
          branch 'main'
          not { buildingTag() }
        }
      }
      steps {
        echo "Triggering update-argoCD-deploy-config job…"
        build job: 'update-argoCD-deploy-config', wait: false
      }
    }

    // === 4) Release: chỉ khi build từ Git tag vX.Y.Z ===
    stage('🏷 Release: Detect Tag') {
      when { buildingTag() }
      steps {
        script {
          RELEASE_TAG = env.GIT_TAG
          echo "🎉 Release detected: ${RELEASE_TAG}"
        }
      }
    }

    stage('🔨 Release: Build & Push All Images') {
      when { buildingTag() }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-login',
          usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS'
        )]) {
          script {
            // Docker login
            def login = isUnix()
              ? "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
              : "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
            isUnix()? sh(login) : bat(login)

            // Build & push tất cả service với đúng tag release
            def services = [
              'discovery-server','config-server','admin-server','api-gateway',
              'customers-service','visits-service','vets-service','genai-service'
            ]
            services.each { svc ->
              def img = "npt1601/${svc}:${RELEASE_TAG}"
              echo "Building & pushing ${img}"
              if (isUnix()) {
                sh "docker build -t ${img} ./spring-petclinic-${svc}"
                sh "docker push ${img}"
              } else {
                bat "docker build -t ${img} .\\spring-petclinic-${svc}"
                bat "docker push ${img}"
              }
            }
          }
        }
      }
    }

    stage('✏️ Release: Update staging.yaml & Push') {
      when { buildingTag() }
      steps {
        script {
          def globalHost   = 'spring.staging.com'
          def namespace    = 'petclinic-staging'
          def portMap = [
            'discovery-server' : 8761,
            'config-server'    : 8888,
            'admin-server'     : 9090,
            'api-gateway'      : 8080,
            'customers-service': 8081,
            'visits-service'   : 8082,
            'vets-service'     : 8083,
            'genai-service'    : 8084
          ]

          // Xây YAML
          def yaml = """\
globalHost: ${globalHost}
ingress:
  enabled: true
  host: ${globalHost}
services:
"""
          portMap.each { svc, p ->
            yaml += """  - name: ${svc}
      image: npt1601/${svc}
      tag:   ${RELEASE_TAG}
      port:  ${p}
"""
            if (svc!='discovery-server') {
              yaml += """      env:
        - name: EUREKA_SERVER_URL
          value: http://discovery-server.${namespace}.svc.cluster.local:8761
        - name: CONFIG_SERVER_URL
          value: http://config-server.${namespace}.svc.cluster.local:8888

"""
            }
          }

          // Commit & push lên repo config
          dir('petclinic-deploy-config') {
            deleteDir()
            checkout([
              $class: 'GitSCM',
              branches: [[name:'main']],
              userRemoteConfigs: [[
                url: 'https://github.com/NPT0116/petclinic-deploy-config.git',
                credentialsId: 'github-thanh-token'
              ]]
            ])
            writeFile file: 'values/staging.yaml', text: yaml

            if (isUnix()) {
              sh '''
                git config user.name "jenkins"
                git config user.email "jenkins@ci.local"
                git add values/staging.yaml
                git commit -m "ci: bump staging.yaml → ${RELEASE_TAG}"
                git push origin main
              '''
            } else {
              bat '''
                git config user.name "jenkins"
                git config user.email "jenkins@ci.local"
                git add values\\staging.yaml
                git commit -m "ci: bump staging.yaml → %RELEASE_TAG%"
                git push origin main
              '''
            }
          }
        }
      }
    }


  } // end stages

  post {
    success {
      echo "✅ Pipeline succeeded${env.GIT_TAG? " on release ${GIT_TAG}" : ''}."
    }
    failure {
      echo "❌ Pipeline failed."
    }
  }
}
