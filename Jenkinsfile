pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
        TMPDIR = '/var/tmp'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
         stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Shencko/petshop_app.git'
            }
        }
         stage('Maven Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
         stage('Maven Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('SonarQube FS Scan') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petshop \
                        -Dsonar.java.binaries=. -Dsonar.projectKey=Petshop'''
                }
            }
        }
        stage('Quality Gate check') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }
        stage('Build war file') {
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML', odcInstallation: 'dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build,Tag and Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh 'docker build -t petshop .'
                        sh 'docker tag petshop shencko/petshop:latest'
                        sh 'docker push shencko/petshop:latest'
                    }
                }
            }
        }
         stage('Trviy FS Scan') {
            steps {
                sh ''' mkdir -p $TMPDIR
                        trivy fs . '''
            }
        }
        stage('Deploy via container') {
            steps {
                sh 'docker run -d --name pet1 -p 8080:8080 shencko/petshop:latest'
            }
        }
        stage('Deploy to K8S') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'gke-cluster', contextName: '', credentialsId: 'k8s-cred', namespace: 'petshop', restrictKubeConfigAccess: false, serverUrl: 'https://104.197.218.22') {
                        sh 'kubectl apply -f deployment.yaml -n petshop'
                        sh 'kubectl get pod -n petshop'
                        sh 'kubectl get svc -n petshop'
                        sleep 20
                    }
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
                to: 'pel28emman@gmail.com'
        }
    }
}
