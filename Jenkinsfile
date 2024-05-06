
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
                git branch: 'main', url: 'https://github.com/yasreebsaad/DVWA.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=dvwa \
                    -Dsonar.projectKey=dvwa'''
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
                       sh "docker build -t dvwa ."
                       sh "docker tag dvwa yasreebakmal/dvwa:latest "
                       sh "docker push yasreebakmal/dvwa:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview yasreebakmal/dvwa:latest'
                       sh 'docker-scout cves yasreebakmal/dvwa:latest'
                       sh 'docker-scout recommendations yasreebakmal/dvwa:latest'
                   }
                }   
            }
        }
          stage('TRIVY FS SCAN') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                      // sh 'docker-scout quickview yasreebakmal/dvwa:latest'
                       //sh 'docker-scout cves yasreebakmal/dvwa:latest'
                       //sh 'docker-scout recommendations yasreebakmal/dvwa:latest'
                      
                       sh 'trivy image yasreebakmal/dvwa:latest > trivyfs.json'
                   }
                }
                
            }
        }
        stage("Remove container"){
            steps{
                sh "docker stop dvwa | true"
                sh "docker rm dvwa | true"
            }
        }
        stage("deploy_docker"){
            steps{
                sh "docker run -d --name dvwa -p 3000:3000 yasreebakmal/dvwa:latest"
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
