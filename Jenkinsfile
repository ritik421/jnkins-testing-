pipeline {
    agent { label 'Worker-1' }

    options {
        // Prevent multiple builds piling up, keep only the latest
        disableConcurrentBuilds(abortPrevious: true)
    }

    stages {
        stage('Checkout Jenkinsfile Repo') {
            steps {
                checkout scm
            }
        }

        stage('Checkout Feathers Repo') {
            steps {
                git branch: 'bpt/stage',
                    url: 'https://github.com/attentive-fx/feathers',
                    credentialsId: 'd8025270-629b-4058-8293-9b8d4cc40022'
            }
        }

        stage('Setup Python') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail
                export PATH="$HOME/.local/bin:$PATH"
                poetry env use /usr/local/bin/python3.10
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail
                poetry install --no-interaction --no-root
                poetry run pip install pytest-html
                '''
            }
        }

        stage('Prepare Reports Directory') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail
                REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                mkdir -p "$REPORT_DIR"
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail
                cd ${WORKSPACE}

                # Explicitly tell Django where settings are + add root to PYTHONPATH
                export DJANGO_SETTINGS_MODULE=nexus.settings
                export PYTHONPATH="${PYTHONPATH:-}:$(pwd)"

                poetry run pytest nexus/ \
                  --tb=short -v \
                  --junitxml="reports/${BUILD_NUMBER}_feather/report.xml" \
                  --html="reports/${BUILD_NUMBER}_feather/report.html" \
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

