pipeline {
    agent { label 'worker' }

    stages {
        stage('Setup') {
            steps {
                script {
                    git branch: 'main', credentialsId: 'Github', url: 'https://github.com/sharara99/DEPI-Final-Project.git'
                }
            }
        }

        stage('Build Infrastructure') {
            steps {
                script {
                    sh '''
                        cd terraform
                        terraform init
                        terraform plan -out=tfplan

                        # Check if there are changes to be applied
                        if terraform show -json tfplan | jq .resource_changes | grep -q '"change"'; then
                            echo "Changes detected, applying infrastructure changes..."
                            terraform apply -auto-approve tfplan
                        else
                            echo "No changes to infrastructure, skipping apply."
                        fi
                    '''
                }
            }
        }

        stage('Ansible for Configuration and Management') {
            steps {
                script {
                    sh '''
                        ls -la  # Check contents of the Ansible main directory
                        ansible --version
                        ansible-playbook -i inventory.ini ansible-playbook.yml -e build_number=${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh '''
                            docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                            ansible-playbook -i inventory.ini ansible-playbook.yml -e build_number=${BUILD_NUMBER}
                        '''
                    }
                }
            }
        }

        stage('Deploy ArgoCD with Helm') {
            steps {
                script {
                    echo "Deploying ArgoCD using Helm..."
                    sh '''
                        ls -la
                        if [ -d "k8s/helm/ArgoCD" ]; then
                            cd k8s/helm/ArgoCD
                            ./deploy-argocd-minikube.sh
                        else
                            echo "Directory k8s/helm/ArgoCD does not exist!"
                            exit 1
                        fi
                    '''
                }
            }
        }

        stage('Create ArgoCD Application') {
            steps {
                script {
                    echo "Creating ArgoCD Application..."
                    sh '''
                        # Apply the ArgoCD application configuration
                        cd k8s/helm/ArgoCD
                        kubectl apply -f argocd-app.yaml

                        # Wait for the deployment to be ready
                        kubectl rollout status deployment/to-do-app-helm -n to-do-app

                        # Update the Kubernetes deployment with the new Docker image (rolling update)
                        kubectl set image deployment/to-do-app-helm to-do-app=sharara99/to-do-app:${BUILD_NUMBER} --record -n to-do-app
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Please check the logs for errors.'
        }
    }
}
