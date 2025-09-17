pipeline {
    agent { label 'Worker-1' }

    options {
        disableConcurrentBuilds(abortPrevious: true)
    }

    parameters {
        choice(
            name: 'BRANCH_NAME',
            choices: ['bpt/stage', 'bpt/master'],
            description: 'Choose branch to run tests on (default is bpt/stage)'
        )
    }

    stages {
        stage('Select Branch') {
            steps {
                script {
                    env.BRANCH_NAME = params.BRANCH_NAME ?: 'bpt/stage'

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
                    try {
                        sh '''#!/bin/bash
                        set -eu
                        cd /home/vikas/workspace/Test-Cases/Test-Cases-Execution

                        # Pre-step
                        sudo bash /home/vikas/py.sh
                        export PATH="$HOME/.local/bin:$PATH"

                        # Install dependencies
                        poetry lock
                        poetry install

                        # Ensure pytest-html exists (permanent)
                        poetry run pip install --quiet pytest-html

                        # Setup secrets
                        sudo gcloud secrets versions access latest \
                          --secret=beam-qa-test \
                          --project=aai-network-test > nexus/.env
                        sudo cp /home/vikas/.test/firebase_credentials.json nexus/

                        # Reports dir
                        REPORT_DIR="reports/${BUILD_NUMBER}_feather"
                        mkdir -p "$REPORT_DIR"

                        # Run tests
                        poetry run pytest nexus/ --tb=short -v \
                          --junitxml="$REPORT_DIR/report.xml" \
                          --html="$REPORT_DIR/report.html" \
                          --self-contained-html
                        '''
                    } catch (Exception e) {
                        echo "⚠️ Tests failed → marking build as UNSTABLE."
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }

    post {
        always {
            // Collect XML results
            junit 'reports/**/report.xml'

            // Publish HTML for this build
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "reports/${BUILD_NUMBER}_feather",
                reportFiles: 'report.html',
                reportName: "Pytest HTML Report"
            ])

            script {
                if (currentBuild.result == 'SUCCESS') {
                    echo "✅ Build Success → Reports published."
                } else if (currentBuild.result == 'UNSTABLE') {
                    echo "⚠️ Build Unstable (tests failed) → Reports published."
                } else {
                    echo "❌ Build Failed → Reports may be incomplete."
                }
            }
        }
    }
}

