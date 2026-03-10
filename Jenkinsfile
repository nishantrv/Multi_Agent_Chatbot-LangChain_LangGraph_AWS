// pipeline {
//     agent any

//     environment {
//         SONAR_PROJECT_KEY  = 'llmops'
//         SONAR_SCANNER_HOME = tool 'sonarcube-scan'
//         AWS_REGION         = 'us-east-1'
//         ECR_REPO           = 'my-repo'
//         IMAGE_TAG          = 'latest'
//     }

//     stages {

//         stage('Cloning Github repo to Jenkins') {
//             steps {
//                 script {
//                     echo 'Cloning Github repo to Jenkins............'
//                     checkout scmGit(
//                         branches: [[name: '*/main']],
//                         extensions: [[$class: 'CleanBeforeCheckout']],  // ✅ wipes workspace before clone
//                         userRemoteConfigs: [[
//                             credentialsId: 'github-mulit-agent-token',
//                             url: 'https://github.com/nishantrv/Multi_Agent_Chatbot-LangChain_LangGraph_AWSECR_FARGATE_JENKINS_TAVILYSEARCH-' 
//                         ]]
//                     )
//                 }
//             }
//         }

//         stage('SonarQube Analysis') {
//             steps {
//                 withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
//                     withSonarQubeEnv('sonarqube') {
//                         sh """
//                         ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
//                         -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
//                         -Dsonar.sources=. \
//                         -Dsonar.host.url=http://sonarqube-dind:9000 \
//                         -Dsonar.login=${SONAR_TOKEN}
//                         """
//                     }
//                 }
//             }
//         }


//         stage('Build and Push Docker Image to ECR') {
//             steps {
//                 withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-token']]) {
//                     script {
//                         def accountId = sh(
//                             script: "aws sts get-caller-identity --query Account --output text",
//                             returnStdout: true
//                         ).trim()
//                         def ecrUrl = "${accountId}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO}"

//                         sh """
//                         aws ecr get-login-password --region ${AWS_REGION} | \
//                             docker login --username AWS --password-stdin ${ecrUrl}
//                         docker build -t ${env.ECR_REPO}:${IMAGE_TAG} .
//                         docker tag ${env.ECR_REPO}:${IMAGE_TAG} ${ecrUrl}:${IMAGE_TAG}
//                         docker push ${ecrUrl}:${IMAGE_TAG}
//                         """
//                     }
//                 }
//             }
//         }

//         stage('Deploy to ECS Fargate') {
//             steps {
//                 withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-token']]) {
//                     script {
//                         sh """
//                         aws ecs update-service \
//                           --cluster able-bird-emufac \
//                           --service multi-ai-agent-101-service-czlo7vxp \
//                           --force-new-deployment \
//                           --region ${AWS_REGION}
//                         """
//                     }
//                 }
//             }
//         }
//     }

//     post {  // ✅ added - always clean up workspace after run
//         always {
//             cleanWs()
//         }
//         success {
//             echo 'Pipeline completed successfully!'
//         }
//         failure {
//             echo 'Pipeline failed — check logs above.'
//         }
//     }
// }


pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        SONAR_PROJECT_KEY  = 'llmops'
        SONAR_SCANNER_HOME = tool 'sonarcube-scan'

        AWS_REGION   = 'us-east-1'
        ECR_REPO     = 'my-repo'
        IMAGE_TAG    = 'latest'

        ECS_CLUSTER  = 'able-bird-emufac'
        ECS_SERVICE  = 'multi-ai-agent-101-service-czlo7vxp'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Cloning Github repo to Jenkins...'
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [[$class: 'CleanBeforeCheckout']],
                    userRemoteConfigs: [[
                        credentialsId: 'github-mulit-agent-token',
                        url: 'https://github.com/nishantrv/Multi_Agent_Chatbot-LangChain_LangGraph_AWSECR_FARGATE_JENKINS_TAVILYSEARCH-'
                    ]]
                )
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            set -eu
                            "${SONAR_SCANNER_HOME}/bin/sonar-scanner" \
                              -Dsonar.projectKey="${SONAR_PROJECT_KEY}" \
                              -Dsonar.sources=. \
                              -Dsonar.host.url="http://sonarqube-dind:9000" \
                              -Dsonar.token="${SONAR_TOKEN}"
                        '''
                    }
                }
            }
        }

        stage('Build and Push Docker Image to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-token']]) {
                    sh '''
                        set -eu

                        ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
                        ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        ECR_URI="${ECR_REGISTRY}/${ECR_REPO}"

                        ECR_PASSWORD="$(aws ecr get-login-password --region "${AWS_REGION}")"
                        printf '%s' "${ECR_PASSWORD}" | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

                        docker build -t "${ECR_REPO}:${IMAGE_TAG}" .
                        docker tag "${ECR_REPO}:${IMAGE_TAG}" "${ECR_URI}:${IMAGE_TAG}"
                        docker push "${ECR_URI}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Deploy to ECS Fargate') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-token']]) {
                    sh '''
                        set -eu

                        aws ecs update-service \
                          --cluster "${ECS_CLUSTER}" \
                          --service "${ECS_SERVICE}" \
                          --force-new-deployment \
                          --region "${AWS_REGION}"

                        aws ecs wait services-stable \
                          --cluster "${ECS_CLUSTER}" \
                          --services "${ECS_SERVICE}" \
                          --region "${AWS_REGION}"

                        aws ecs describe-services \
                          --cluster "${ECS_CLUSTER}" \
                          --services "${ECS_SERVICE}" \
                          --region "${AWS_REGION}" \
                          --query 'services[0].{service:serviceName,status:status,running:runningCount,desired:desiredCount,taskDefinition:taskDefinition}' \
                          --output table
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed — check logs above.'
        }
    }
}