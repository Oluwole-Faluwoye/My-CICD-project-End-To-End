def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any

    environment {
        WORKSPACE = "${env.WORKSPACE}"
        NEXUS_CREDENTIAL_ID = 'Nexus-Credential'
        JENKINS_SSH_KEY_PATH = '/home/ansibleadmin/.ssh/id_rsa'
        JENKINS_SSH_PUB_PATH = '/home/ansibleadmin/.ssh/id_rsa.pub'
        BOOTSTRAP_USER = 'ec2-user'
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    stages {

        // ------------------- BUILD & TEST -------------------
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    echo 'Archiving artifact...'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps { sh 'mvn test' }
        }

        stage('Integration Test') {
            steps { sh 'mvn verify -DskipUnitTests' }
        }

        stage('Checkstyle Code Analysis') {
            steps { sh 'mvn checkstyle:checkstyle' }
            post { success { echo 'Generated Checkstyle report.' } }
        }

        // ------------------- SONARQUBE -------------------
        stage('SonarQube Inspection') {
            steps {
                script {
                    env.JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
                    env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
                    echo "Using JAVA_HOME = ${env.JAVA_HOME}"
                    sh 'java -version'

                    withEnv(['MAVEN_OPTS=--add-opens java.base/java.lang=ALL-UNNAMED ' +
                             '--add-opens java.base/java.io=ALL-UNNAMED ' +
                             '--add-opens java.base/java.util=ALL-UNNAMED ' +
                             '--add-opens java.base/java.lang.reflect=ALL-UNNAMED']) {
                        withSonarQubeEnv('SonarQube') {
                            withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                                sh """
                                mvn sonar:sonar \
                                -Dsonar.projectKey=Java-WebApp-Project \
                                -Dsonar.host.url=http://172.31.1.89:9000 \
                                -Dsonar.login=${SONAR_TOKEN}
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('SonarQube GateKeeper') {
            steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } }
        }

        // ------------------- NEXUS -------------------
        stage('Nexus Artifact Uploader') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '172.31.2.104:8081',
                    groupId: 'webapp',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'maven-project-releases',
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [[
                        artifactId: 'webapp',
                        classifier: '',
                        file: "${WORKSPACE}/webapp/target/webapp.war",
                        type: 'war'
                    ]]
                )
            }
        }

        // ------------------- POPULATE KNOWN_HOSTS -------------------
        stage('Populate known_hosts') {
            steps {
                withCredentials([file(credentialsId: 'Bootstrap-Key', variable: 'BOOTSTRAP_KEY_FILE')]) {
                    sh """
                    # Ensure Jenkins .ssh directory exists
                    mkdir -p /var/lib/jenkins/.ssh
                    chmod 700 /var/lib/jenkins/.ssh

                    # Ensure known_hosts file exists
                    touch /var/lib/jenkins/.ssh/known_hosts
                    chmod 600 /var/lib/jenkins/.ssh/known_hosts

                    # Dynamically add all EC2 hosts from inventory to known_hosts
                    for host in \$(ansible-inventory -i ${WORKSPACE}/ansible-config/aws_ec2.yaml --list | jq -r '.. | .ansible_host? // empty'); do
                        echo "Adding \$host to known_hosts"
                        ssh-keyscan -H \$host >> /var/lib/jenkins/.ssh/known_hosts 2>/dev/null
                    done
                    """
                }
            }
        }

        // ------------------- DEPLOY TO DEVELOPMENT -------------------
        stage('Deploy to Development Env') {
            environment { HOSTS = 'dev' }
            steps {
                withCredentials([file(credentialsId: 'Bootstrap-Key', variable: 'BOOTSTRAP_KEY_FILE')]) {
                    sh """
                    # Bootstrap Jenkins key
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
                      ${WORKSPACE}/ansible-config/deploy_tomcat.yaml \
                      -e environment=${HOSTS} \
                      -e workspace_path=${WORKSPACE} \
                      -e jenkins_pub_key=${JENKINS_SSH_PUB_PATH} \
                      -e jenkins_priv_key=${JENKINS_SSH_KEY_PATH} \
                      -e bootstrap_user=${BOOTSTRAP_USER} \
                      -e ansible_user=${BOOTSTRAP_USER} \
                      -e ansible_ssh_private_key_file=${BOOTSTRAP_KEY_FILE} \
                      --tags bootstrap

                    # Deploy WAR using Jenkins key
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
                      ${WORKSPACE}/ansible-config/deploy_tomcat.yaml \
                      -e environment=${HOSTS} \
                      -e workspace_path=${WORKSPACE} \
                      -e jenkins_pub_key=${JENKINS_SSH_PUB_PATH} \
                      -e jenkins_priv_key=${JENKINS_SSH_KEY_PATH} \
                      -e ansible_user=ansibleadmin \
                      -e ansible_ssh_private_key_file=${JENKINS_SSH_KEY_PATH} \
                      --tags deploy
                    """
                }
            }
        }

        stage('Deploy to Staging Env') {
            environment { HOSTS = 'stage' }
            steps {
                withCredentials([file(credentialsId: 'Bootstrap-Key', variable: 'BOOTSTRAP_KEY_FILE')]) {
                    sh """
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
                      ${WORKSPACE}/ansible-config/deploy_tomcat.yaml \
                      -e environment=${HOSTS} \
                      -e workspace_path=${WORKSPACE} \
                      -e jenkins_pub_key=${JENKINS_SSH_PUB_PATH} \
                      -e jenkins_priv_key=${JENKINS_SSH_KEY_PATH} \
                      -e bootstrap_user=${BOOTSTRAP_USER} \
                      -e ansible_user=${BOOTSTRAP_USER} \
                      -e ansible_ssh_private_key_file=${BOOTSTRAP_KEY_FILE} \
                      --tags bootstrap

                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
                      ${WORKSPACE}/ansible-config/deploy_tomcat.yaml \
                      -e environment=${HOSTS} \
                      -e workspace_path=${WORKSPACE} \
                      -e jenkins_pub_key=${JENKINS_SSH_PUB_PATH} \
                      -e jenkins_priv_key=${JENKINS_SSH_KEY_PATH} \
                      -e ansible_user=ansibleadmin \
                      -e ansible_ssh_private_key_file=${JENKINS_SSH_KEY_PATH} \
                      --tags deploy
                    """
                }
            }
        }

        stage('Quality Assurance Approval') {
            steps { input('Do you want to proceed to Production?') }
        }

        stage('Deploy to Production Env') {
            environment { HOSTS = 'prod' }
            steps {
                withCredentials([file(credentialsId: 'Bootstrap-Key', variable: 'BOOTSTRAP_KEY_FILE')]) {
                    sh """
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
                      ${WORKSPACE}/ansible-config/deploy_tomcat.yaml \
                      -e environment=${HOSTS} \
                      -e workspace_path=${WORKSPACE} \
                      -e jenkins_pub_key=${JENKINS_SSH_PUB_PATH} \
                      -e jenkins_priv_key=${JENKINS_SSH_KEY_PATH} \
                      -e bootstrap_user=${BOOTSTRAP_USER} \
                      -e ansible_user=${BOOTSTRAP_USER} \
                      -e ansible_ssh_private_key_file=${BOOTSTRAP_KEY_FILE} \
                      --tags bootstrap

                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
                      ${WORKSPACE}/ansible-config/deploy_tomcat.yaml \
                      -e environment=${HOSTS} \
                      -e workspace_path=${WORKSPACE} \
                      -e jenkins_pub_key=${JENKINS_SSH_PUB_PATH} \
                      -e jenkins_priv_key=${JENKINS_SSH_KEY_PATH} \
                      -e ansible_user=ansibleadmin \
                      -e ansible_ssh_private_key_file=${JENKINS_SSH_KEY_PATH} \
                      --tags deploy
                    """
                }
            }
        }
    }

    post {
        always {
            slackSend(
                channel: '#af-cicd-pipeline-2',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \nBuild Timestamp: ${env.BUILD_TIMESTAMP} \nWorkspace: ${env.WORKSPACE} \nMore info: ${env.BUILD_URL}"
            )
        }
    }
}
