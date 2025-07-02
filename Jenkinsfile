pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO = '682475225405.dkr.ecr.us-east-1.amazonaws.com/java-maven-app'
        GIT_CREDENTIALS_ID = 'git-ssh'
        ECR_CREDENTIALS_ID = 'aws-ecr-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    git (credentialsId: "${GIT_CREDENTIALS_ID}", 
                        url: 'git@github.com:aaleem123/CI-CD-Pipeline-with-EKS-and-AWS-ECR.git', 
                        branch: 'main')
                }
            }
        }

        stage('Increment Version') {
            steps {
                sshagent (credentials: ["${GIT_CREDENTIALS_ID}"]) {
                    script {
                        sh '''
                            mvn build-helper:parse-version versions:set \
                            -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.$((parsedVersion.incrementalVersion + 1)) \
                            versions:commit
                        '''
                        def pomfix = readFile('pom.xml')
                        def matcher = pomfix =~ '<version>(.+)</version>'
                        def version = matcher[0][1]
                        
                        env.IMAGE_NAME = "${ECR_REPO}:${version}-${BUILD_NUMBER}"

                        sh """
                            git config user.email "ci@example.com"
                            git config user.name "CI Pipeline"
                            git add pom.xml
                            git commit -m "Bump version to ${version}"
                            git push origin HEAD:main
                        """
                    }
                }
            }
        }

        stage('Build Spring Boot Jar') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${ECR_CREDENTIALS_ID}", usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region ${AWS_REGION}

                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                        docker push ${IMAGE_NAME}
                    '''
                }
            }
        }
    }
}
