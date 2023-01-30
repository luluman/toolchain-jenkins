//===-*- Groovy -*-===

pipeline {
    agent {
        docker {
            image 'sophgo/tpuc_dev:latest'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'echo $WORKSPACE'
                sh "echo $HOME"
            }
        }
    }
}
