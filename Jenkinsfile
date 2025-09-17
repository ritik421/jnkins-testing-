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

                    // Restrict only to stage/master
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
                sh '''#!/bin/bash
                set -euo pipefail
                cd /home/vikas/workspace/Test-Cases/Test-Cases-Execution

                # Pre-step
                sudo bash /home/vikas/py.sh
                export PATH="$HOME/.local/bin:$PATH"

                # Install dependencies
                poetry install --no-root

                # Setup secrets
                sudo gcloud secrets versions access latest \
                  --secret=beam-qa-test \
                  --project=aai-network-test > nexus/.env
                sudo cp /home/vikas/.test/firebase_credentials.json nexus/

                # Reports
                REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                mkdir -p "$REPORT_DIR"

                poetry run pytest nexus/ --tb=short -v \
                  --junitxml="$REPORT_DIR/report.xml" \
                  --html="$REPORT_DIR/report.html" \
                  --self-contained-html
                '''
            }
        }
    }

    post {
        always {
            junit 'reports/**/report.xml'
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

