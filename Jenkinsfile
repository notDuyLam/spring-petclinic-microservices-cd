pipeline {
    agent {
        label 'development-server'
    }

    options {
        // Clean before build
        skipDefaultCheckout(true)
    }

    tools {
        maven '3.9.9'
    }

    environment {
        USERNAME = "tienminhktvn2"
        REGISTRY = "registry.caotienminh.software"
        REPO = "springcommunity"
    }

    stages {
        stage('Check SCM') {
            steps {
                cleanWs()

                checkout scm

                script {
                    // Get the first 8 characters of the SHA Git Commit
                    def gitCommitHash = sh(script: "git rev-parse --short=8 HEAD", returnStdout: true).trim()
                    env.COMMIT_HASH = gitCommitHash
                }
            }
        }

        stage('Check Changed Files') {
            steps {
                script {
                    def branch_name = ""

                    if (env.CHANGE_ID) {
                        branch_name = "${env.CHANGE_TARGET}"

                        // Fetch main branch if it is a Pull Request
                        sh("git fetch origin ${branch_name}:${branch_name} --no-tags")
                    }
                    else {
                        branch_name = 'HEAD~1'
                    }

                    def changedFiles = sh(script: "git diff --name-only ${branch_name}", returnStdout: true).trim()

                    echo "${changedFiles}"

                    def folderList = ['spring-petclinic-customers-service', 'spring-petclinic-vets-service', 'spring-petclinic-visits-service']
                    
                    def changedFolders = changedFiles.split('\n')
                        .collect { it.split('/')[0] }
                        .unique()
                        .findAll { folderList.contains(it) }
                    
                    echo "Changed Folders: \n${changedFolders.join('\n')}"
                    
                    env.CHANGED_MODULES = changedFolders.join(',')
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES ? env.CHANGED_MODULES.split(',') : []

                    def imageTag = env.COMMIT_HASH

                    def gitTag = sh(script: "git describe --tags --exact-match || true", returnStdout: true).trim()

                    sh "git fetch origin main:refs/remotes/origin/main --no-tags"

                    def isMainBranch = sh(script: "git branch -r --contains HEAD | grep 'origin/main' || true", returnStdout: true).trim() != ""

                    echo "${gitTag}, ${isMainBranch}"

                    if (gitTag != "" && isMainBranch) {
                        imageTag = gitTag
                    }
                    else if (gitTag == "" && isMainBranch) {
                        imageTag = "dev"
                    }

                    env.IMAGE_TAG = imageTag

                    if (modules.size() > 0) {

                        // Build and Tag Images for changed modules
                        for (module in modules) {
                            echo "Run unit test for: ${module}"
                            sh "mvn test -pl ${module}"
                            def buildImagesCommand = "bash ./mvnw clean install -pl ${module} -PbuildDocker -DskipTests"
                            echo "Build Images for affected modules: ${module}"
                            sh "${buildImagesCommand}"
                            sh "docker tag ${REPO}/${module}:latest ${REGISTRY}/${REPO}/${module}:${imageTag}"
                        }
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES ? env.CHANGED_MODULES.split(',') : []

                    if (modules.size() > 0) {
                        withCredentials([usernamePassword(
                            credentialsId: 'REGISTRY_CREDENTIALS',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"

                            for (module in modules) {
                                def imageName = "${USERNAME}/${module}:${env.IMAGE_TAG}"
                                echo "Pushing Docker image: ${imageName}"
                                sh "docker push ${imageName}"
                            }
                        }
                    } 
                    else {
                        echo "No changed modules; skipping Docker push."
                    }
                }
            }
        }

        stage('Update Helm Chart for new Tags') {
            when {
                expression {
                    // Run if current build is a Git tag
                    return sh(script: "git describe --tags --exact-match || true", returnStdout: true).trim() != ""
                }
            }
            steps {
                script {
                    // Check if the git tag is on the main branch
                    def isMainBranch = sh(script: "git branch -r --contains HEAD | grep 'origin/main' || true", returnStdout: true).trim() != ""

                    if (!isMainBranch) {
                        echo "Tag '${env.IMAGE_TAG}' is not on 'main' branch. Skipping Helm update."
                        return
                    }

                    def modules = env.CHANGED_MODULES ? env.CHANGED_MODULES.split(',') : []

                    if (modules.size() > 0) {
                        echo "Triggering Helm chart update for new tag '${env.IMAGE_TAG}' and changed modules: ${modules.join(', ')}"

                        build job: 'update_helm_chart_staging',
                        wait: false,
                        parameters: [
                            string(name: 'CUSTOMERS_SERVICE_TAG', value: modules.contains('spring-petclinic-customers-service') ? env.IMAGE_TAG : 'latest'),
                            string(name: 'VETS_SERVICE_TAG', value: modules.contains('spring-petclinic-vets-service') ? env.IMAGE_TAG : 'latest'),
                            string(name: 'VISITS_SERVICE_TAG', value: modules.contains('spring-petclinic-visits-service') ? env.IMAGE_TAG : 'latest')
                        ]
                    } else {
                        echo "No changed modules for tag '${env.IMAGE_TAG}'. Skipping deployment."
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Logging out of Docker Hub'
            sh 'docker logout'

            echo 'Cleaning up all Docker images…'
            sh 'docker image prune -af'

            echo 'Clean Workspace'
            cleanWs()
        }
    }
}