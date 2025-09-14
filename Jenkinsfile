pipeline {
    agent { label 'Worker-1' }

    triggers {
        // Requires webhook on original repo
        githubPullRequest()
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${env.CHANGE_BRANCH ?: env.BRANCH_NAME}",
                    url: 'https://github.com/attentive-fx/feathers',
                    credentialsId: 'd8025270-629b-4058-8293-9b8d4cc40022'
            }
        }

        stage('Verify Branch') {
            when {
                not {
                    anyOf {
                        branch 'bpt/stage'
                        branch 'bpt/master'
                    }
                }
            }
            steps {
                echo "Skipping build because branch is not bpt/stage or bpt/master"
                script { currentBuild.result = 'SUCCESS' }
            }
        }

        stage('Setup Environment') {
            when {
                anyOf {
                    branch 'bpt/stage'
                    branch 'bpt/master'
                }
            }
            steps {
                sh '''
                set -euo pipefail
                export PATH="$HOME/.local/bin:$PATH"
                poetry env use /usr/local/bin/python3.10

                echo "Installing dependencies..."
                poetry install --no-interaction --no-root

                # Ensure pytest-html is installed
                poetry run pip install pytest-html
                '''
            }
        }

        stage('Run Tests') {
            when {
                anyOf {
                    branch 'bpt/stage'
                    branch 'bpt/master'
                }
            }
            steps {
                sh '''
                REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                mkdir -p "$REPORT_DIR"

                cd ${WORKSPACE}

                export PYTHONPATH="${PYTHONPATH:-}:.:$(pwd)/nexus"
                export DJANGO_SETTINGS_MODULE=nexus.settings

                echo "Running pytest with Poetry..."
                poetry run pytest \
                  --junitxml="$REPORT_DIR/report.xml" \
                  --html="$REPORT_DIR/report.html" \
                  --self-contained-html
                '''
            }
            post {
                always {
                    junit 'reports/**/report.xml'
                    publishHTML([
                        reportDir: 'reports',
                        reportFiles: '**/report.html',
                        reportName: 'Pytest HTML Report'
                    ])
                }
            }
        }
    }
}

