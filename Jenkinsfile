pipeline { 
    agent any
    
    tools {
        jdk 'Java17'
        maven 'Maven3'
        nodejs 'Node18'
    }

    environment {
        MYSQL_DB = 'database'
        MYSQL_PORT = '3306'
        APP_NAME = "portfolio-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "mariem360"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {
        stage('Start MySQL (no password)') {
            steps {
                sh 'docker rm -f mysql-test || true'
                sh '''
                docker run --name mysql-test \
                    -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
                    -e MYSQL_DATABASE=$MYSQL_DB \
                    -p $MYSQL_PORT:3306 \
                    -d mysql:8.0 \
                    --default-authentication-plugin=mysql_native_password

                echo "Waiting for MySQL to start..."
                sleep 20
                '''
            }
        }

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/mira33ch/MyPortfolio'
            }
        }

        stage("Build Application") {
            steps {
                script {
                    // Build Angular frontend
                    dir('portfolio-frontend') {
                        sh 'npm install'
                        sh 'npm run build --prod'
                    }

                    // Build Spring Boot backend
                    dir('demo1') {
                        sh 'mvn clean package -DskipTests'
                    }
                }
            }
        }

        stage("Test Application") {
            steps {
                dir('demo1') {
                    sh 'mvn test'
                }
            }
        }

      stage('SonarQube Analysis Backend') {
            steps {
                dir('demo1') {
                    script {
                        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                            sh 'mvn sonar:sonar'
                        }
                    }
                }
            }
        }
     stage('SonarQube Frontend Analysis') {
           steps {
               dir('portfolio-frontend') {
                   script {
                       withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                           sh 'npm install sonar-scanner' // or have it installed already
                           sh 'npx sonar-scanner -Dsonar.projectKey=frontend-project-key -Dsonar.sources=src'
                       }
                   }
               }
           }
     }

     stage('Quality Gate') {
          steps {
              script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
              }
          }
     }
     stage('Build & Push Docker Images') {
         steps {
             script {
                    // Build and push backend image
                 docker.withRegistry('', DOCKER_PASS) {
                     def backendImage = docker.build("${IMAGE_NAME}-backend:${IMAGE_TAG}", 'demo1/')
                     backendImage.push()
                     backendImage.push('latest')
                 }

                    // Build and push frontend image
                 docker.withRegistry('', DOCKER_PASS) {
                     def frontendImage = docker.build("${IMAGE_NAME}-frontend:${IMAGE_TAG}", 'portfolio-frontend/')
                     frontendImage.push()
                     frontendImage.push('latest')
                 }
             }
         }
     }
     stage("Trivy Scan") {
        steps {
            script {
	            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image elyeshub/portfolio-app-pipeline-frontend:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
                sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image elyeshub/portfolio-app-pipeline-backend:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
            }
        }
     }

        stage('Stop MySQL') {
            steps {
                sh '''
                docker stop mysql-test || true
                docker rm mysql-test || true
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished'
        }
    }
}
