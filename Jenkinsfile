pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                credentialsId: 'github-cred',
                url: 'https://github.com/creativesenthil/Boardgame.git'
            }
        }
        stage('Compile') {
            steps { sh 'mvn compile' }
        }
        stage('Test') {
            steps { sh 'mvn test' }
        }
        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false
            }
        }
        stage('Build') {
            steps { sh 'mvn package -DskipTests=true' }
        }
        stage('Docker Build & Tag') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh "docker build -t senthilkumarsoundararajan/boardgame:latest ."
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html senthilkumarsoundararajan/boardgame:latest"
            }
        }
        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh "docker push senthilkumarsoundararajan/boardgame:latest"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'kube-config',
                    clusterName: 'boardgame-cluster',
                    namespace: 'webapps') {
                        sh 'kubectl create namespace webapps --dry-run=client -o yaml | kubectl apply -f -'
                        sh 'kubectl apply -f deployment-service.yaml'
                        sh 'kubectl get pods -n webapps'
                        sh 'kubectl get svc -n webapps'
                }
            }
        }
    }
    post {
        always {
            emailext(
                subject: "Pipeline: ${currentBuild.currentResult} - ${env.JOB_NAME}",
                body: "Build ${env.BUILD_NUMBER} status: ${currentBuild.currentResult}",
                to: 'your-email@gmail.com',
                attachmentsPattern: 'trivy-*.html'
            )
        }
    }
}
