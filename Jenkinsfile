// ── shared list of microservices ──
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

    stages {
        stage('📥 Checkout') {
            steps {
                // checkout whatever ref (branch or tag) Jenkins detected
                checkout scm
            }
        }

        stage('🧬 Detect changed services') {
            when { not { buildingTag() } }  // skip for tag‐builds
            steps {
                script {
                    // grab the short commit ID
                    def commitId = isUnix()
                        ? sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        : bat(script: "git rev-parse --short HEAD", returnStdout: true).trim().readLines().last().trim()

                    echo "🔍 Commit ID: ${commitId}"

                    // what files changed
                    def changedFiles = isUnix()
                        ? sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().split('\n')
                        : bat(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().readLines()

                    echo "📂 Files changed:\n${changedFiles.join('\n')}"

                    // detect which services changed
                    def changedServices = [] as Set
                    allServices.each { svc ->
                        changedFiles.each { file ->
                            if (file.startsWith("spring-petclinic-${svc}/")) {
                                changedServices << svc
                            }
                        }
                    }

                    if (changedServices.isEmpty()) {
                        echo "✅ No service changes detected, skipping build."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "🧱 Changed services: ${changedServices.join(', ')}"

                    // build & push by commitId
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-login',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        // login
                        def loginCmd = isUnix()
                            ? "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                            : "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
                        isUnix() ? sh(loginCmd) : bat(loginCmd)

                        // build & push each changed service
                        changedServices.each { svc ->
                            def image = "npt1601/${svc}:${commitId}"
                            echo "🚧 Building ${svc} → ${image}"
                            def buildCmd = isUnix()
                                ? "./mvnw -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
                                : "mvnw.cmd -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${image}"
                            isUnix() ? sh(buildCmd) : bat(buildCmd)

                            echo "📤 Pushing ${image}"
                            isUnix() ? sh("docker push ${image}") : bat("docker push ${image}")
                        }
                    }
                }
            }
        }

            stage('🔖 Release / Staging Build') {
            when {
                allOf {
                    buildingTag()                              // chỉ chạy khi build tag
                    expression { 
                        // tên tag phải đúng semver v1.2.3
                        env.BRANCH_NAME ==~ /^v\d+\.\d+\.\d+$/ 
                    }
                }
            }
            steps {
                script {
                    def tag = env.BRANCH_NAME
                    echo "🎯 Building RELEASE for tag ${tag}"

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-login',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        // Docker login
                        def login = isUnix()
                            ? "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                            : "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
                        isUnix() ? sh(login) : bat(login)

                        // build & push all services với tag này
                        allServices.each { svc ->
                            def img = "npt1601/${svc}:${tag}"
                            echo "🚧 Building ${svc} → ${img}"
                            def buildCmd = isUnix()
                                ? "./mvnw -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${img}"
                                : "mvnw.cmd -pl spring-petclinic-${svc} spring-boot:build-image -DskipTests -Dspring-boot.build-image.imageName=${img}"
                            isUnix() ? sh(buildCmd) : bat(buildCmd)

                            echo "📤 Pushing ${img}"
                            isUnix() ? sh("docker push ${img}") : bat("docker push ${img}")
                        }
                    }
                }
            }
        }


    }

    post {
        success { echo "✅ Pipeline complete!" 
                                    script {
if ((env.BRANCH_NAME ?: 'main') == 'main') {
    echo "🔁 Đang gọi job update-argoCD-deploy-config..."
    
    build job: 'update-argoCD-deploy-config',
          wait: false, // hoặc true nếu bạn muốn đợi chạy xong
          parameters: [] // có thể truyền params tại đây nếu cần
} else {
    echo "⏭ Không phải branch main, không gọi job update-argoCD-deploy-config."
}

        }}
        failure { echo "❌ Something failed." }
    }
}
