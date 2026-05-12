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
                          semgrep scan --json --output sast-results.json --config="p/javascript" --config="p/owasp-top-ten" --exclude="codeql-db" --exclude="node_modules" --exclude="codeql-results.sarif" --exclude="codeql" .
                          
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
                        # Create a unique temporary directory OUTSIDE the Jenkins workspace
                        export CQ_TMP="/tmp/codeql-thesis-${BUILD_NUMBER}"
                        mkdir -p $CQ_TMP
                        
                        echo "Step 1: Downloading CodeQL CLI Bundle..."
                        # Download and extract directly to the /tmp folder
                        wget -q https://github.com/github/codeql-action/releases/latest/download/codeql-bundle-linux64.tar.gz -O $CQ_TMP/codeql-bundle.tar.gz
                        tar -xzf $CQ_TMP/codeql-bundle.tar.gz -C $CQ_TMP
                        export PATH=$PATH:$CQ_TMP/codeql
                        
                        echo "Step 2: Creating CodeQL Database Out-of-Tree..."
                        # We build the DB in /tmp, but tell it to scan the current directory (.)
                        codeql database create $CQ_TMP/codeql-db --language=javascript-typescript --source-root .
                        
                        echo "Step 3: Running CodeQL Analysis (Unleashed!)..."
                        # Output the SARIF file back into the Jenkins workspace so we can archive it
                        codeql database analyze $CQ_TMP/codeql-db javascript-security-extended.qls \
                          --format=sarif-latest \
                          --output=$(pwd)/codeql-results.sarif \
                          --ram=12000 \
                          --threads=8
                          
                        echo "Step 4: Cleanup..."
                        # Nuke the temporary folder to save disk space
                        rm -rf $CQ_TMP
                        '''
                    }
                    post {
                        always {
                            // Save the SARIF artifact to the Jenkins UI
                            archiveArtifacts artifacts: 'codeql-results.sarif', allowEmptyArchive: true
                        }
                    }
                }

                // --- Stage 2c: SonarQube
                stage('SonarQube SAST') {
                    steps {
                        echo "Executing SonarQube Analysis..."
                        
                        // We MUST use a script block to declare variables in a Declarative Pipeline
                        script {
                            def scannerHome = tool 'SonarScanner'
                            
                            // Wraps the execution using the server credentials
                            withSonarQubeEnv('Sonar-VM') {
                                sh "${scannerHome}/bin/sonar-scanner \
                                    -Dsonar.projectKey=juice-shop-thesis \
                                    -Dsonar.sources=. \
                                    -Dsonar.exclusions=**/codeql/**,**/codeql-db/**,**/*.tar.gz,**/node_modules/**,**/sast-results.json,**/codeql-results.sarif \
                                    -Dsonar.host.url=http://10.17.0.250:9000 \
                                    -Dsonar.javascript.node.maxspace=3072"
                            }
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