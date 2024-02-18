pipeline {
    agent any

    environment {
        AZURE_SUBSCRIPTION_ID = 'c8ce3edc-0522-48a3-b7e4-afe8e3d731d9'
        AZURE_TENANT_ID = '4ccd6048-181f-43a0-ba5a-7f48e8a4fa35'
        RESOURCE_GROUP = 'AKS'
        NAMESPACE = 'front'

        CONTAINER_REGISTRY = 'goodbirdacr.azurecr.io'
        REPO = 'metahospital/front'
        IMAGE_NAME = 'metahospital/front:latest'

        GIT_CREDENTIALS_ID = 'jenkins-git-access'

        NEW_IMAGE_TAG = "${env.BRANCH_NAME}.${env.BUILD_ID}"
        GIT_REPOSITORY = "rlozi99/MetaMedicine_front_deploy" 

        KUBECONFIG = '/home/azureuser/.kube/config'

        // TAG_VERSION = "v1.0.Beta"
        // TAG = "${TAG_VERSION}${env.BUILD_ID}"
    }

    stages{
        stage('Check BRANCH_NAME') {
            steps {
                script {
                    echo "Current BRANCH_NAME is ${env.BRANCH_NAME}"
                }
            }
        }
        stage('Initialize..') {
            steps {
                script {
                    // Multibranch Pipeline에서 제공하는 BRANCH_NAME 환경 변수를 사용합니다.
                    def branch = env.BRANCH_NAME
                    echo "Checked out branch: ${branch}"
                    
                    if (branch == 'dev') {
                        env.TAG = 'dev'
                        env.DIR_NAME = "development"
                    } else if (branch == 'stg') {
                        env.TAG = 'stg'
                        env.DIR_NAME = "staging"
                    } else if (branch == 'prod') {
                        env.TAG = 'latest'
                        env.DIR_NAME = "production"
                    } else {
                        env.TAG = 'unknown'
                        env.DIR_NAME = "unknown"
                    }
                    
                    echo "TAG is now set to ${env.TAG}"
                }
            }
        }  
        stage('Build and Push Docker Image to ACR') {
            steps {
                script {                 
                    withCredentials([usernamePassword(credentialsId: 'acr-credential-id', passwordVariable: 'ACR_PASSWORD', usernameVariable: 'ACR_USERNAME')]) {
                        // Log in to ACR
                        sh "az acr login --name $CONTAINER_REGISTRY --username $ACR_USERNAME --password $ACR_PASSWORD"

                        // Build and push Docker image to ACR
                        // 변경: 이미지 이름을 $CONTAINER_REGISTRY/$IMAGE_NAME으로 수정
                        sh "docker build -t $CONTAINER_REGISTRY/$REPO:$NEW_IMAGE_TAG ."
                        sh "docker push $CONTAINER_REGISTRY/$REPO:$NEW_IMAGE_TAG"
                    }
                }
            }
        }
        stage('Checkout GitOps') {
                    steps {
                        // 'front_gitops' 저장소에서 파일들을 체크아웃합니다.
                        git branch: BRANCH_NAME,
                            credentialsId: 'jenkins-git-access',
                            url: "https://github.com/${GIT_REPOSITORY}"
                    }
                }
        stage('Update Kubernetes Configuration') {
            steps {
                script {
                    sh "ls -la"
                    // Assuming the kubeconfig is set correctly on the Jenkins agent.
                    withKubeConfig([credentialsId: 'kubeconfig-credentials-id']) {
                        // Change directory to the location of your kustomization.yaml
                        sh "ls -la"

                        dir("overlays/${env.DIR_NAME}") {
                            sh "ls -la"
                            sh "kustomize edit set image $REPO:$NEW_IMAGE_TAG"
                            sh "kustomize build . | kubectl apply -f - -n ${NAMESPACE}"
                        }
                    }
                }
            }
        }
        stage('Commit and Push Changes to GitOps Repository') {
            steps {
                script {
                    dir("overlays/${env.DIR_NAME}") {
                        // GitOps 저장소로 변경 사항을 커밋하고 푸시합니다.
                        sh "git config user.email 'rlozi1999@gmail.com'"
                        sh "git config user.name 'rlozi99'"
                        sh "git add ."
                        sh "git commit -m 'Update image tag to $NEW_IMAGE_TAG'"
                        // Credential을 사용하여 GitHub에 push
                        withCredentials([usernamePassword(credentialsId: 'jenkins-git-access', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                            // GIT_USERNAME과 GIT_PASSWORD 환경변수를 사용하여 push
                            dir("overlays/${env.DIR_NAME}") {
                                sh("git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_REPOSITORY}.git ${env.BRANCH_NAME}")
                            }
                        }
                        // sh "git push origin ${env.BRANCH_NAME}"
                    }
                }
            }
        }
    }
}
// This assumes you have a 'withKubeConfig' shared library or function in Jenkins to handle kubeconfig.
def withKubeConfig(Map args, Closure body) {
    withCredentials([file(credentialsId: args.credentialsId, variable: 'KUBECONFIG')]) {
        body.call()
    }
}
