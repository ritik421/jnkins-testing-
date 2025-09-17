pipeline {
    agent { label 'Worker-1' }

    options {
        // Prevent multiple builds from piling up
        disableConcurrentBuilds(abortPrevious: true)
    }

    parameters {
        choice(
            name: 'BRANCH_NAME',
            choices: ['bpt/stage', 'bpt/master'], // first one = default
            description: 'Choose branch to run tests on'
        )
    }

    stages {
        stage('Select Branch') {
            steps {
                script {
                    env.BRANCH_NAME = params.BRANCH_NAME

                    if (!(env.BRANCH_NAME in ['bpt/stage', 'bpt/master'])) {
                        error "❌ Build skipped. Only allowed on bpt/stage or bpt/master, got ${env.BRANCH_NAME}"
                    }

                    echo "⚡ Final branch to run pipeline: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Checkout Feathers Repo') {
            steps {
                git branch: "${env.BRANCH_NAME}",
                    url: 'https://github.com/attentive-fx/feathers',
                    credentialsId: 'd8025270-629b-4058-8293-9b8d4cc40022'
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    // Run pytest and capture exit code instead of failing immediately
                    def exitCode = sh(script: '''
                        #!/bin/bash
                        set -euo pipefail

                        sudo bash /home/vikas/py.sh
                        export PATH="$HOME/.local/bin:$PATH"
                        cd /home/vikas/workspace/Test-Cases/Test-Cases-Execution

                        # Install dependencies
                        poetry lock
                        poetry install

                        # Ensure pytest-html is installed
                        poetry run pip install pytest-html

                        # Setup secrets
                        sudo gcloud secrets versions access latest \
                          --secret=beam-qa-test \
                          --project=aai-network-test > nexus/.env
                        sudo cp /home/vikas/.test/firebase_credentials.json nexus/

                        # Reports
                        REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                        mkdir -p "$REPORT_DIR"

                        # Run pytest
                        poetry run pytest nexus/ --tb=short -v \
                          --junitxml="$REPORT_DIR/report.xml" \
                          --html="$REPORT_DIR/report.html" \
                          --self-contained-html
                        ''', returnStatus: true)

                    // Mark build as unstable if tests failed
                    if (exitCode != 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo "⚠️ Tests failed. Marking build as UNSTABLE."
                    }
                }
            }
        }
    }

    post {
        always {
            // ✅ Collect JUnit XML results
            junit 'reports/**/report.xml'

            // ✅ Publish Pytest HTML report
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

