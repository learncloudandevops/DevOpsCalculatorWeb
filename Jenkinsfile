
pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-amazon-corretto.x86_64'
        PATH = "${JAVA_HOME}/bin:${PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('Verify WAR') {
            steps {
                sh '''
                    test -f target/calculator.war
                    ls -lh target/calculator.war
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'target/*.war',
                             fingerprint: true
        }

        success {
            echo 'Build and tests succeeded. WAR artifact archived.'
        }

        failure {
            echo 'Build failed. Review the Jenkins console output.'
        }
    }
}
