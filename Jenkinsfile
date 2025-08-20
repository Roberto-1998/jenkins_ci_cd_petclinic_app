def COLOR_MAP =[
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
    }

/*  post {
        always {
            echo 'Slack Notifications'
            slackSend channel: '#demo-app-ci-pipeline',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    } */
}