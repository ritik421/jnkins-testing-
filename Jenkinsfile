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
                        error "❌ Only bpt/stage or bpt/master are allowed, got ${env.BRANCH_NAME}"
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
                        sh """#!/bin/bash
                            set -e

                            # Pre-step
                            sudo bash /home/vikas/py.sh
                            export PATH="\$HOME/.local/bin:\$PATH"
                            cd /home/vikas/workspace/Test-Cases/Test-Cases-Execution

                            # Install deps
                            poetry lock
                            poetry install
                            poetry run pip install pytest-html

                            # Secrets
                            sudo gcloud secrets versions access latest \
                              --secret=beam-qa-test \
                              --project=aai-network-test > nexus/.env
                            sudo cp /home/vikas/.test/firebase_credentials.json nexus/

                            # Reports dir
                            REPORT_DIR="reports/\${BUILD_NUMBER}_feather"
                            mkdir -p "\$REPORT_DIR"

                            # Run pytest (capture exit code without failing pipeline)
                            set +e
                            poetry run pytest nexus/ --tb=short -v \
                              --junitxml="\$REPORT_DIR/report.xml" \
                              --html="\$REPORT_DIR/report.html" \
                              --self-contained-html
                            exitCode=\$?
                            set -e

                            exit \$exitCode
                        """
                    } catch (err) {
                        // Distinguish between infra error and test failures
                        if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                            echo "⚠️ BUILD SUCCESS at runtime, but tests failed → marking build as UNSTABLE."
                            currentBuild.result = 'UNSTABLE'
                        } else {
                            echo "❌ Infra/setup error → failing pipeline."
                            throw err
                        }
                    }
                }
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
                reportName: 'Pytest HTML Report',
                reportTitles: 'Test Results',
                useWrapperFileDirectly: true   // 👈 makes HTML report the landing page
            ])
        }

        failure {
            echo "❌ Pipeline FAILED (infra/setup issue)."
        }

        unstable {
            echo "⚠️ Pipeline UNSTABLE (tests failed)."
        }

        success {
            echo "✅ Pipeline SUCCESS (tests passed)."
        }
    }
}

