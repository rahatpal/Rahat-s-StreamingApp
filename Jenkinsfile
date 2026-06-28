pipeline {
    agent any

    stages {
        stage('Docker Check') {
            steps {
                sh '''
                    set -x
                    whoami
                    pwd
                    docker --version
                    docker info
                '''
            }
        }
    }
}