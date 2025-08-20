def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning',
    'ABORTED': '#CCCCCC'
]

pipeline {
    agent any

    tools {
        maven 'MAVEN3.9'
        jdk 'JDK21'
    }

    environment {
        CENTRAL_REPO = 'petclinic-app-maven-central'
        RELEASE_REPO = 'petclinic-app-release'
        NEXUS_HOST = '172.31.37.76'
        NEXUS_PORT = '8081'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        NEXUS_LOGIN = 'nexuslogin'
        APP_DIR = 'spring-petclinic'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        PROJECT_NAME = 'petclinic-app'
        REGISTRY_CREDENTIALS = 'ecr:us-east-1:awscreds'
        APP_REGISTRY = '585008040165.dkr.ecr.us-east-1.amazonaws.com/petclinic-app-image'
        PETCLINIC_REGISTRY = 'https://585008040165.dkr.ecr.us-east-1.amazonaws.com'
        ECS_CLUSTER = 'petclinic-staging'
        ECS_SERVICE = 'petclinic-task-service-7me3dpdu'
    }

    stages {
        stage('Build') {
            steps {
                dir(env.APP_DIR) {
                    sh 'mvn -s ../settings.xml -DskipTests clean package'
                }
            }
            post {
                success {
                    dir(env.APP_DIR) {
                        echo 'Great! Now archiving the artifact.'
                        archiveArtifacts artifacts: '**/target/*.jar'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                dir(env.APP_DIR) {
                    sh 'mvn -s ../settings.xml test'
                    sh 'mvn -s ../settings.xml verify'
                }
            }
        }
        stage('Code Quality') {
            steps {
                dir(env.APP_DIR) {
                    sh 'mvn  -s ../settings.xml checkstyle:checkstyle'
                }
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                dir(env.APP_DIR) {
                    withSonarQubeEnv("${SONARSERVER}") {
                        sh """${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=${PROJECT_NAME} \
                        -Dsonar.projectName=${PROJECT_NAME} \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportsPaths=target/checkstyle-result.xml"""
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Upload Artifact') {
            steps {
                dir(env.APP_DIR) {
                    nexusArtifactUploader(
        nexusVersion: 'nexus3',
        protocol: 'http',
        nexusUrl: "${NEXUS_HOST}:${NEXUS_PORT}",
        groupId: 'QA',
        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
        repository: "${RELEASE_REPO}",
        credentialsId: "${NEXUS_LOGIN}",
        artifacts: [
            [artifactId: "${PROJECT_NAME}",
             classifier: '',
             file: 'target/spring-petclinic-3.5.0-SNAPSHOT.jar',
             type: 'jar']
        ]
     )
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    def jarName = "${PROJECT_NAME}-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.jar"

                    def downloadUrl = "http://${NEXUS_HOST}:${NEXUS_PORT}/repository/${RELEASE_REPO}/QA/${PROJECT_NAME}/${env.BUILD_ID}-${env.BUILD_TIMESTAMP}/${jarName}"

                    sh 'mkdir -p docker_build'
                    sh 'rm -rf docker_build/*'

                    sh "curl -u ${NEXUS_USER}:${NEXUS_PASS} -o docker_build/${jarName} ${downloadUrl}"

                    dockerImage = docker.build("${APP_REGISTRY}:${env.BUILD_NUMBER}", '-f Dockerfiles/Dockerfile docker_build')
                }
            }
        }
        stage('Upload App Image') {
            steps {
                script {
                    docker.withRegistry(PETCLINIC_REGISTRY, REGISTRY_CREDENTIALS) {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Staging ECS') {
            steps {
                withAWS(credentials: 'awscreds', region: 'us-east-1') {
                    sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --force-new-deployment"
                }
            }
        }

        stage('Cleanup') {
            steps {
                script {
                    sh "docker rmi ${APP_REGISTRY}:${env.BUILD_NUMBER} || true"
                    sh "docker rmi ${APP_REGISTRY}:latest || true"
                }
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications'
            slackSend channel: '#petclinic-app',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
