def registry = 'https://clemusgo.jfrog.io'
def imageName = 'clemusgo.jfrog.io/clemusgo-docker-local/ttrend'
def version   = '2.1.2'
pipeline {
    agent {
        node {
            label 'maven'
        }
    }

    environment {
        PATH = "/opt/apache-maven-3.9.4/bin:$PATH"
    }

    stages {
        stage("Build") {
            steps {
                echo "---------- Build Started ----------"
                sh 'mvn clean deploy'
                echo "---------- Build Completed ----------"
            }
        }
        stage("Test") {
            steps {
                echo "---------- Unit Test Started ----------"
                sh 'mvn surefire-report:report'
                echo "---------- Unit Test Completed ----------"
            }
        }
        stage('SonarQube analysis') {
            environment {
                scannerHome = tool 'clemusgo-sonar-scanner';
            }
            steps {
                withSonarQubeEnv('clemusgo-sonarqube-server') { // If you have configured more than one global server connection, you can specify its name
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
                        def qg = waitForQualityGate() // Reuse taskId previously collected by with SonarQubeEnv
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage("Jar Publish") {
            steps {
                script {
                    echo '<--------------- Jar Publish Started --------------->'
                    def server = Artifactory.newServer url:registry+"/artifactory" ,  credentialsId:"artifact-cred"
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "jarstaging/(*)",
                                "target": "libs-release-local/{1}",
                                "flat": "false",
                                "props" : "${properties}",
                                "exclusions": [ "*.sha1", "*.md5"]
                            }
                        ]
                    }"""
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    server.publishBuildInfo(buildInfo)
                    echo '<--------------- Jar Publish Ended --------------->'
                }
            }
        }

        stage(" Docker Build ") {
            steps {
                echo '<--------------- Docker Build Started --------------->'
                script {
                    app = docker.build("${imageName}:${version}")
                }
                echo '<--------------- Docker Build Ends --------------->'
            }
        }

        stage(" Docker Publish ") {
            steps {
                echo '<--------------- Docker Publish Started --------------->'
                script {
                    docker.withRegistry(registry, 'artifact-cred') {
                        app.push()
                    }
                }
                echo '<--------------- Docker Publish Ended --------------->'
            }
            post {
                always {
                    script {
                        echo '<--------------- Docker Clean Up Temporary Files --------------->'
                        sh 'rm -rf /home/ubuntu/jenkins/workspace/ttrend-multibranch_main@tmp/*'
                        echo '<--------------- Clean up operations completed --------------->'
                    }
                }
                failure {
                    echo 'Handle failures. Send notifications.'
                }
            }
        }
    }
}