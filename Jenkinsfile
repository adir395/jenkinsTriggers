pipeline {
    agent any
    tools {
        maven "MAVEN3.9"
    }

/* */
    environment {
        registryCredential = 'ecr:eu-north-1:awscreds'
        appRegistry = "084375581933.dkr.ecr.eu-north-1.amazonaws.com/vprofileappimage"
        vprofileRegistry = "https://084375581933.dkr.ecr.eu-north-1.amazonaws.com"
        cluster = "jenkinsCluster"
        service = "vprofileappsvc"
    }

    stages {
        stage('Fetch code') {
            steps {
                git branch: 'docker', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn install -DskipTests'
            }
            post {
                success {
                    echo "archiving artifact"
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage('Unit test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('sonar code analysis') {

		  environment {
             scannerHome = tool 'sonar6.2'
          }

          steps {
            withSonarQubeEnv('sonarserver') {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }
          }
        }
        stage('sonarqube quility gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build App Image') {
          steps {

            script {
                dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
          }

        }
        stage('Upload App Image') {
          steps{
            script {
              docker.withRegistry( vprofileRegistry, registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }

        stage ("remove container image") {
            steps {
                sh 'docker rmi -f $(docker images -a -q)'
            }
        }
        stage('Deploy to ecs') {
          steps {
            withAWS(credentials: 'awscreds', region: 'eu-north-1') {
            sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
               }
          }
        }

        }
    post {
        always {
            echo 'slack notfications'
            slackSend channel: '#devopscicd',
            message: "Find Status of Pipeline:- ${currentBuild.currentResult} ${env.JOB_NAME} ${env.BUILD_NUMBER} ${BUILD_URL}"
        }
    }
}
