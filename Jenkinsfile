pipeline {
    agent any

    tools {
        maven 'maven1'      // Maven name configured in Jenkins Global Tool Configuration         // JDK configured in Jenkins
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }
    }

    post {
        success {
            echo 'Build Successful'
        }
        failure {
            echo 'Build Failed'
        }
    }
}
