pipeline {
    agent any

    environment {
        SONAR_SCANNER = tool 'sonar-scanner'
    }

    stages {
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${SONAR_SCANNER}/bin/sonar-scanner \
                          -Dsonar.projectKey=starbucks-kubernetes \
                          -Dsonar.projectName=starbucks-kubernetes \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL}
                    """
                }
            }
        }
    }
}