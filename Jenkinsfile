pipeline {
    agent {
        node {
            label 'k8s_master'
        }
    }

    parameters {
        string(name: 'SERVICE_NAMES', description: 'Comma-separated list of service names to update')
        string(name: 'IMAGE_TAGS', description: 'Comma-separated list of image tags to deploy')
        string(name: 'VALUES_FILE', description: 'The values file to update (e.g., values-staging.yaml or values-dev.yaml)')
    }

    stages {
        stage('Prepare Git') {
            steps {
                script {
                    sh "git checkout main"
                    sh "git config user.email 'jenkins@example.com'"
                    sh "git config user.name 'Jenkins CI'"
                    withCredentials([usernamePassword(
                        credentialsId: 'gitHubHeml',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )]) {
                        sh "git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mandeotv1234/k8s_helm_lab2"
                        sh "git pull origin main"
                    }
                }
            }
        }

        stage('Update Image Tags') {
            steps {
                script {
                    def serviceNames = params.SERVICE_NAMES.split(',')
                    def imageTags = params.IMAGE_TAGS.split(',')
                    def valuesFile = params.VALUES_FILE ?: "values-dev.yaml"

                    echo "SERVICE_NAMES: ${params.SERVICE_NAMES}"
                    echo "IMAGE_TAGS: ${params.IMAGE_TAGS}"
                    echo "VALUES_FILE: ${valuesFile}"

                    if (serviceNames.size() != imageTags.size()) {
                        error "Mismatch between number of services and tags"
                    }

                    for (int i = 0; i < serviceNames.size(); i++) {
                        def service = serviceNames[i].trim()
                        def tag = imageTags[i].trim()
                        def yamlServiceKey = service.replace('-', '')
                        if (service.contains('-')) {
                            def parts = service.split('-')
                            yamlServiceKey = parts[0]
                            for (int j = 1; j < parts.length; j++) {
                                yamlServiceKey += parts[j].capitalize()
                            }
                        }

                        echo "Updating ${service} (${yamlServiceKey}) to tag ${tag} in ${valuesFile}"

                        sh """
                            cd petclinic-helm/petclinic
                            if [ ! -f "${valuesFile}" ]; then
                                echo "Creating ${valuesFile} from values.yaml"
                                cp values.yaml ${valuesFile}
                            fi
                        """

                        sh """
                            cd petclinic-helm/petclinic
                            sed -i '/  ${yamlServiceKey}:/,+5 s/tag: .*/tag: ${tag}/g' ${valuesFile}
                        """

                        sh """
                            cd petclinic-helm/petclinic
                            echo "Updated section for ${yamlServiceKey}:"
                            grep -A 5 "${yamlServiceKey}:" ${valuesFile} || echo "Key not found."
                        """
                    }
                }
            }
        }

        stage('Commit and Push') {
            steps {
                script {
                    def valuesFile = params.VALUES_FILE ?: "values-dev.yaml"
                    sh "cd petclinic-helm/petclinic && git status --porcelain > /tmp/changes.txt"
                    def hasChanges = readFile('/tmp/changes.txt').trim()

                    if (hasChanges) {
                        withCredentials([usernamePassword(
                            credentialsId: 'gitHubHeml',
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_PASSWORD'
                        )]) {
                            sh """
                                cd petclinic-helm/petclinic
                                git add ${valuesFile}
                                git commit -m "Update services: ${params.SERVICE_NAMES} with tags: ${params.IMAGE_TAGS} in ${valuesFile}"
                                git push origin main
                            """
                        }
                        echo "✅ Changes pushed to GitHub."
                    } else {
                        echo "⚠️ No changes to commit."
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Successfully updated Helm charts for services: ${params.SERVICE_NAMES}"
        }
        failure {
            echo "❌ Failed to update Helm charts"
        }
    }
}
