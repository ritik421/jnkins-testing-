pipeline {
    agent { label 'Worker-1' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Python') {
            steps {
                sh '''
                set -euo pipefail
                export PATH="$HOME/.local/bin:$PATH"
                poetry env use /usr/local/bin/python3.10
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                set -euo pipefail
                poetry install --no-interaction --no-root
                poetry run pip install pytest-html
                '''
            }
        }

        stage('Prepare Reports Directory') {
            steps {
                sh '''
                set -euo pipefail
                REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                mkdir -p "$REPORT_DIR"
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                set -euo pipefail
                cd ${WORKSPACE}
                export PYTHONPATH="${PYTHONPATH:-}:.:$(pwd)/nexus"
                export DJANGO_SETTINGS_MODULE=nexus.settings

                poetry run pytest \
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

