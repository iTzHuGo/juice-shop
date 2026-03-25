pipeline {
    agent any

    // Name of the tool created in Jenkins Tools
    tools {
        nodejs 'Node22' 
    }

    stages {
        stage('Checkout') {
            steps {
                // Pull code from Git repository
                checkout scm
            }
        }
        
        stage('Build & Install Dependencies') {
            steps {
                echo "Building the application using Node 22..."
                sh 'npm install'
            }
        }

        stage('Functional Testing') {
            steps {
                echo "Running standard unit tests..."
                // catchError ensures the pipeline finishes and reports the state, 
                // even if Juice Shop's internal tests throw an error.
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'npm test'
                }
            }
        }
    }
    
    post {
        always {
            echo "Baseline Pipeline execution complete."
        }
    }
}