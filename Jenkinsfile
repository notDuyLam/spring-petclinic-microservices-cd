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
                    if (modules.size() > 0) {

                        // Build and Tag Images for changed modules
                        for (module in modules) {
                            echo "Run unit test for: ${module}"
                            sh "mvn test -pl ${module}"
                            def buildImagesCommand = "./mvnw clean install -pl ${module} -PbuildDocker -DskipTests"
                            echo "Build Images for affected modules: ${module}"
                            sh "${buildImagesCommand}"
                            sh "docker tag springcommunity/${module}:latest ${USERNAME}/${module}:${env.COMMIT_HASH}"
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
                            credentialsId: 'DOCKER_HUB_CREDENTIALS',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                            for (module in modules) {
                                def imageName = "${USERNAME}/${module}:${env.COMMIT_HASH}"
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

        stage('Trigger Developer Build Job') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES ? env.CHANGED_MODULES.split(',') : []

                    def customersBranch = 'main'
                    def vetsBranch = 'main'
                    def visitsBranch = 'main'
                    def genaiBranch = 'main'

                    if (modules.contains('spring-petclinic-customers-service')) {
                        customersBranch = env.BRANCH_NAME
                    }
                    if (modules.contains('spring-petclinic-vets-service')) {
                        vetsBranch = env.BRANCH_NAME
                    }
                    if (modules.contains('spring-petclinic-visits-service')) {
                        visitsBranch = env.BRANCH_NAME
                    }

                    build job: 'developer_build', 
                        parameters: [
                            string(name: 'CUSTOMERS_IMAGE_TAG', value: customersBranch),
                            string(name: 'VETS_IMAGE_TAG',      value: vetsBranch),
                            string(name: 'VISITS_IMAGE_TAG',    value: visitsBranch),
                            string(name: 'GENAI_IMAGE_TAG',     value: genaiBranch)
                        ],
                        wait: false
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
        }
    }
}