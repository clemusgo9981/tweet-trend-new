pipeline {
    agent {
        node {
            label 'maven'
        }
    }

    stages {
        stage('Clone-code') {
            steps {
                git branch: 'main', credentialsId: 'github_cred', url: 'https://github.com/clemusgo9981/tweet-trend-new.git'
            }
        }
    }
}