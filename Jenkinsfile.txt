
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node19'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/yasreebsaad/Hotstar-Clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=myntra \
                    -Dsonar.projectKey=myntra'''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                 //dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                // dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                sh 'pwd'
            }
        }
        stage('Docker Scout FS') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       //sh 'docker-scout quickview fs://.'
                       //sh 'docker-scout cves fs://.'
                       sh 'docker --version'
                   }
                }   
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build -t myntra ."
                       sh "docker tag myntra yasreebakmal/myntra:latest "
                       sh "docker push yasreebakmal/myntra:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview yasreebakmal/myntra:latest'
                       sh 'docker-scout cves yasreebakmal/myntra:latest'
                       sh 'docker-scout recommendations yasreebakmal/myntra:latest'
                   }
                }   
            }
        }
          stage('TRIVY FS SCAN') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                      // sh 'docker-scout quickview yasreebakmal/myntra:latest'
                       //sh 'docker-scout cves yasreebakmal/myntra:latest'
                       //sh 'docker-scout recommendations yasreebakmal/myntra:latest'
                      
                       sh 'trivy image yasreebakmal/myntra:latest > trivyfs.json'
                   }
                }
                
            }
        }
        stage("Remove container"){
            steps{
                sh "docker stop myntra | true"
                sh "docker rm myntra | true"
            }
        }
        stage("deploy_docker"){
            steps{
                sh "docker run -d --name myntra -p 3000:3000 yasreebakmal/myntra:latest"
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('K8S') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }
    }
}
