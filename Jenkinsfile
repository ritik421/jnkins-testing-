pipeline {
    agent { label 'Worker-1' }

    options {
        // Prevent multiple builds from piling up, keep only the latest
        disableConcurrentBuilds(abortPrevious: true)
        // Show timestamps in logs
        timestamps()
    }

    stages {
        stage('Setup Environment') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail

                # Run your Python setup script (this ensures correct Python version)
                sudo bash /home/vikas/py.sh

                # Ensure poetry is in PATH
                export PATH="$HOME/.local/bin:$PATH"

                cd /home/vikas/workspace/QA-Scripts/jnkins-test

                # Install dependencies
                poetry lock
                poetry install

                # Fetch secrets
                sudo gcloud secrets versions access latest \
                  --secret=beam-qa-test \
                  --project=aai-network-test > nexus/.env

                # Copy Firebase credentials
                sudo cp /home/vikas/.test/firebase_credentials.json nexus/
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail
                cd /home/vikas/workspace/QA-Scripts/jnkins-test

                REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                mkdir -p "$REPORT_DIR"

                # Run pytest with XML + HTML reports
                poetry run pytest nexus/ \
                  --tb=short -v \
                  --junitxml="$REPORT_DIR/report.xml" \
                  --html="$REPORT_DIR/report.html" \
                  --self-contained-html
                '''
            }
        }
    }

    post {
        always {
            // Publish JUnit XML results
            junit 'reports/**/report.xml'

            // Publish HTML report
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: '**/report.html',
                reportName: 'Pytest HTML Report'
            ])
        }
    }
}

