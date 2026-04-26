pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Pipeline is working!'
            }
        }
    }
    post {
        always {
            echo 'Pipeline finished!'
        }
    }
}