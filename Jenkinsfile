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

        stage('Security Scanning (SAST)') {
            parallel {
                
                // --- Stage 2a: Agnostic Security (Semgrep via Docker) ---
                stage('Semgrep SAST') {
                    steps {
                        echo "Executing Semgrep SAST scan via Docker..."
                        // We mount the Jenkins workspace into the Semgrep container
                        sh '''
                        docker run --rm \
                          -v $(pwd):/src \
                          -w /src \
                          returntocorp/semgrep:latest \
                          semgrep scan --json --output sast-results.json --config="p/javascript" --config="p/owasp-top-ten" --exclude="codeql-db" --exclude="node_modules" --exclude="codeql-results.sarif" .
                        '''
                    }
                    post {
                        always {
                            // Save the JSON artifact to the Jenkins UI
                            archiveArtifacts artifacts: 'sast-results.json', allowEmptyArchive: true
                        }
                    }
                }

                // --- Stage 2b: Proprietary Security (CodeQL CLI) ---
                stage('CodeQL SAST') {
                    steps {
                        echo "Executing CodeQL CLI Scan..."
                        // Using the memory-optimized script from our GitLab learnings!
                        sh '''
                        echo "Step 1: Downloading CodeQL CLI Bundle..."
                        wget -q https://github.com/github/codeql-action/releases/latest/download/codeql-bundle-linux64.tar.gz
                        tar -xzf codeql-bundle-linux64.tar.gz
                        export PATH=$PATH:$(pwd)/codeql
                        
                        echo "Step 2: Creating CodeQL Database..."
                        codeql database create codeql-db --language=javascript-typescript --overwrite
                        
                        echo "Step 3: Running CodeQL Analysis (Memory Optimized)..."
                        codeql database analyze codeql-db javascript-security-extended.qls \
                          --format=sarif-latest \
                          --output=codeql-results.sarif \
                          --ram=12000 \
                          --threads=8
                        '''
                    }
                    post {
                        always {
                            // Save the SARIF artifact to the Jenkins UI
                            archiveArtifacts artifacts: 'codeql-results.sarif', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "DevSecOps Pipeline execution complete."
        }
    }
}