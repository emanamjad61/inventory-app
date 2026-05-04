pipeline {
    agent any
    
    environment {
        APP_IMAGE = "inventory-app"
        TEST_IMAGE = "inventory-tester"
        TEST_REPO = "https://github.com/emanamjad61/inventory-tests.git"
    }

    stages {
        stage('Cleanup') {
            steps {
                echo 'Ensuring a clean environment...'
                sh 'docker rm -f app-container tester-container || true'
            }
        }

        stage('Checkout App') {
            steps {
                // Checkout current repo (Application)
                checkout scm
            }
        }

        stage('Checkout Tests') {
            steps {
                // Pull tests from the second repository
                dir('tests') {
                    git url: "${TEST_REPO}", branch: 'main'
                }
            }
        }

        stage('Build Images') {
            steps {
                sh "docker build -t ${APP_IMAGE} ."
                sh "docker build -t ${TEST_IMAGE} ./tests"
            }
        }

        stage('Bring App Up') {
            steps {
                // Start app in background on a specific network
                sh 'docker network create test-net || true'
                sh "docker run -d --name app-container --network test-net -p 5000:5000 ${APP_IMAGE}"
                // Wait for app to wake up
                sh 'sleep 10'
            }
        }

        stage('Execute Tests') {
            steps {
                // Run tests against the running container
                sh "docker run --rm --name tester-container --network test-net -e APP_URL=http://app-container:5000 ${TEST_IMAGE}"
            }
        }
    }

    post {
        always {
            script {
                // Get the email of the person who made the commit
                def committerEmail = sh(script: "git log -1 --pretty=format:'%ae'", returnStdout: true).trim()
                
                mail to: "${committerEmail}, qasimalik@gmail.com",
                     subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) - ${currentBuild.currentResult}",
                     body: "The test results are attached/available in the Jenkins console: ${env.BUILD_URL}"
            }
            // Cleanup after tests
            sh 'docker stop app-container || true'
        }
        success {
            echo 'Tests Passed! Keeping the latest version deployed.'
        }
        failure {
            echo 'Tests Failed! App has been shut down.'
            sh 'docker rm -f app-container || true'
        }
    }
}
