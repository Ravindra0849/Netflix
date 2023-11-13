pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
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
                git 'https://github.com/Ravindra0849/Netflix.git'
            }
        }

        stage('sonarqube Analysis') {
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }

        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Jenkins-sonar' 
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
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'Dockerhub', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=f4ba7296dd271837249197da6420f34e -t netflix ."
                       sh "docker tag netflix ravisree900/netflix:'${env.BUILD_NUMBER}'"
                       sh "docker push ravisree900/netflix:'${env.BUILD_NUMBER}'"
                    }
                }
            }
        }
        
        stage("TRIVY"){
            steps{
                sh "trivy image ravisree900/netflix:'${env.BUILD_NUMBER}' > trivyimage.txt" 
            }
        }
        
        stage('Deploy to container'){
            steps{
                sh "docker run -d --name netflix -p 8081:80 ravisree900/netflix:'${env.BUILD_NUMBER}'"
            }
        }

        stage('Deploy to kubernets'){
            steps
            {
                script
                {
                    sh 'ssh ubuntu@172.31.1.48 kubectl apply -f deployment.yml'
                    sh 'ssh ubuntu@172.31.1.48 kubectl apply -f service.yml'
                }
            }
        }
    }
     post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'ravisree900@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}