pipeline {
    agent any

    tools {
        jdk 'jdk21'
        maven 'maven'
    }

environment {
    SCANNER_HOME = tool 'sonarqube-scanner'
    NEXUS_VERSION = "nexus3"
    NEXUS_URL = "172.20.10.5:8081"        
    NEXUS_PROTOCOL = "https"
    NEXUS_REPOSITORY = "Board-game-artifacts"
    NEXUS_CREDENTIAL_ID = 'nexus-creds'
    IMAGE_NAME = "boardgame-image"
    AWS_REGION = "ap-south-1"                             
    ECR_ACCOUNT_ID = "764512389001"                       
    ECR_REPO = "game-repo/boardgame-app"
    EKS_CLUSTER = "Workshop-Cluster"
    ECR_REGISTR = "764512389001.dkr.ecr.ap-south-1.amazonaws.com"
}


    stages {

        stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']],
                userRemoteConfigs: [[url: 'https://github.com/iambuckky007/boardgame',
                ]]])
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {  
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Boardgame \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage("Push to Nexus") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml"
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    artifactPath = filesByGlob[0].path
                    artifactExists = fileExists artifactPath
                    if (artifactExists) {
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }

        stage('Docker Build & Trivy Scan') {
            steps {
                script {
                    sh """
                        docker build -t ${IMAGE_NAME}:${env.BUILD_NUMBER} .
                        trivy image --format json --output trivy-report-${env.BUILD_NUMBER}.json ${IMAGE_NAME}:${env.BUILD_NUMBER} || true
                    """
                }
            }
        }

        stage('Convert report to CSV') {
            steps {
                sh """
                jq -r '.Results[]?.Vulnerabilities[]? | [
                        .VulnerabilityID,
                        .PkgName,
                        .InstalledVersion,
                        .FixedVersion,
                        .Severity,
                        .Title
                ] | @csv' trivy-report-${env.BUILD_NUMBER}.json > trivy-report-${env.BUILD_NUMBER}.csv || true
                """
            }
            post {
                success {
                    emailext (
                        subject: "Trivy Scan Report Build #${BUILD_NUMBER}",
                        body: "Trivy vulnerability scan report for build #${BUILD_NUMBER}.",
                        to: "kunalkk694@gmail.com",
                        attachmentsPattern: "trivy-report-${BUILD_NUMBER}.csv"
                    )
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh """
                        # Tag image with Jenkins BUILD_ID
                        docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${env.BUILD_NUMBER}
                        
                        # Login to ECR using EC2 IAM role
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        # Push image
                        docker push ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${env.BUILD_NUMBER}
                    """
                }
            }
            post {
                success {
                    sh """
                        docker rmi ${IMAGE_NAME}:${env.BUILD_NUMBER} || true
                    """
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                       sed -e "s|ECR_REGISTRY|${ECR_REGISTR}|g" \
                           -e "s|ECR_REPO|${ECR_REPO}|g" \
                           -e "s|BUILD_ID|${env.BUILD_NUMBER}|g" \
                           deployment-service.yaml > deployment.yaml
                    """
                }
                script {
                    sh 'cat deployment.yaml'
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
                    sh 'kubectl apply -f deployment.yaml'
                }
            }
        }
    }
    post {
        failure {
            emailext(
                to: 'kunalkk694@gmail.com',
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Build failed for job: ${env.JOB_NAME}
    Build number: ${env.BUILD_NUMBER}

    Check logs: ${env.BUILD_URL}
    Blue Ocean: ${env.RUN_DISPLAY_URL}
    """
            )
        }
        success {
            emailext(
                to: 'kunalkk694@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Build successful for job: ${env.JOB_NAME}
    Build number: ${env.BUILD_NUMBER}

    Check logs: ${env.BUILD_URL}
    Blue Ocean: ${env.RUN_DISPLAY_URL}
    """
            )
        }
    }
}
