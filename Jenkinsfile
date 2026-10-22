pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-amazon-corretto.x86_64'
        PATH = "${JAVA_HOME}/bin:${PATH}"
        SONAR_HOST_URL = 'http://sonar.leancloudandevops.online:9000'
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                withCredentials([
                    string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {
                    sh '''
                        mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.projectKey=DevOps-Calculator-Web-Lakshmanan \
                          -Dsonar.projectName="DevOps Calculator Web Lakshmanan" \
                          -Dsonar.host.url="$SONAR_HOST_URL" \
                          -Dsonar.token="$SONAR_TOKEN" \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -Dsonar.coverage.exclusions=**/CalculatorServlet.java
                          
                    '''
                }
            }
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

        stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}

        stage('Deploy to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexususer',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        SETTINGS_FILE="$(mktemp)"
                        trap 'rm -f "$SETTINGS_FILE"' EXIT

                        cat > "$SETTINGS_FILE" <<EOF
<settings>
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>${NEXUS_USERNAME}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
    </servers>
</settings>
EOF

                        chmod 600 "$SETTINGS_FILE"

                        mvn -B -s "$SETTINGS_FILE" deploy
                    '''
                }
            }
        }


       
    
stage('Deploy to Tomcat via Ansible') {
    steps {
        withCredentials([
            sshUserPrivateKey(
                credentialsId: 'ansible-target-ssh',
                keyFileVariable: 'ANSIBLE_SSH_KEY',
                usernameVariable: 'ANSIBLE_SSH_USER'
            ),
            string(
                credentialsId: 'ansible-vault-password',
                variable: 'ANSIBLE_VAULT_PASSWORD'
            )
        ]) {
            sh '''
                set +x
                set -eu

                echo "=========================================="
                echo "Step 1: Determine application version"
                echo "=========================================="

                APP_VERSION="$(mvn -q -DforceStdout \
                    help:evaluate \
                    -Dexpression=project.version)"

                if [ -z "$APP_VERSION" ]; then
                    echo "ERROR: Unable to determine Maven project version."
                    exit 1
                fi

                echo "Application version: $APP_VERSION"

                echo "=========================================="
                echo "Step 2: Verify Ansible SSH credential"
                echo "=========================================="

                test -r "$ANSIBLE_SSH_KEY"
                ssh-keygen -y -f "$ANSIBLE_SSH_KEY" >/dev/null

                echo "SSH private key is valid."

                echo "=========================================="
                echo "Step 3: Test Ansible connectivity"
                echo "=========================================="

                ansible calculator \
                    -i /opt/ansible-calculator/inventory.ini \
                    -m ping \
                    -e "ansible_user=$ANSIBLE_SSH_USER" \
                    -e "ansible_ssh_private_key_file=$ANSIBLE_SSH_KEY"

                echo "=========================================="
                echo "Step 4: Prepare Ansible Vault password"
                echo "=========================================="

                VAULT_PASS_FILE="$(mktemp)"
                trap 'rm -f "$VAULT_PASS_FILE"' EXIT
                chmod 600 "$VAULT_PASS_FILE"
                printf '%s' "$ANSIBLE_VAULT_PASSWORD" > "$VAULT_PASS_FILE"
                unset ANSIBLE_VAULT_PASSWORD

                echo "=========================================="
                echo "Step 5: Deploy version $APP_VERSION"
                echo "=========================================="

                ansible-playbook \
                    -i /opt/ansible-calculator/inventory.ini \
                    /opt/ansible-calculator/deploy-calculator-web.yml \
                    --vault-password-file "$VAULT_PASS_FILE" \
                    -e "ansible_user=$ANSIBLE_SSH_USER" \
                    -e "ansible_ssh_private_key_file=$ANSIBLE_SSH_KEY" \
                    -e "artifact_version=$APP_VERSION"

                echo "=========================================="
                echo "Deployment of version $APP_VERSION succeeded"
                echo "=========================================="
            '''
        }
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
            echo 'Build, tests, SonarQube analysis, WAR verification, Nexus deployment, and artifact archiving succeeded.'
        }

        failure {
            echo 'Pipeline failed. Review the Jenkins console output.'
        }
    }
}