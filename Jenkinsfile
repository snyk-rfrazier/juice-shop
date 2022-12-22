pipeline {
    agent any

    // Requires a configured NodeJS installation via https://plugins.jenkins.io/nodejs/
    tools { nodejs "NodeJS 18.4.0" }

    stages {
        stage('git clone') {
            steps {
                git url: 'https://github.com/snyk-rfrazier/juice-shop.git'
            }
        }

        // Install the Snyk CLI with npm. For more information, check:
        // https://docs.snyk.io/snyk-cli/install-the-snyk-cli
        stage('Install snyk CLI') {
            steps {
                script {
                    sh 'npm install -g snyk snyk-to-html snyk-delta'
                }
            }
        }
        

        // Authorize the Snyk CLI
        stage('Authorize Snyk CLI') {
            steps {
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'snyk auth ${SNYK_TOKEN}'
                }
            }
        }

        stage('Build App') {
            steps {
                sh 'docker build -t juice-shop .'
            }
        }

        stage('Snyk') {
            parallel {
                stage('Snyk Open Source') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'snyk test --fail-on=all --sarif-file-output=results-open-source.sarif'
                        }
                        recordIssues tool: sarif(name: 'Snyk Open Source', id: 'snyk-open-source', pattern: 'results-open-source.sarif')
                    }
                }
                stage('Snyk Code') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'snyk code test --sarif-file-output=results-code.sarif'
                        }
                        sh 'snyk-to-html -i results-code.sarif -o results-code.html'
                        recordIssues  tool: sarif(name: 'Snyk Code', id: 'snyk-code', pattern: 'results-code.sarif')
                    }
                }
                stage('Snyk Container') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'snyk container test ../juice-shop --file=Dockerfile --sarif-file-output=results-container.sarif'
                        }
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'snyk container monitor --file=Dockerfile juice-shop:latest'
                        }
                        recordIssues tool: sarif(name: 'Snyk Container', id: 'snyk-container', pattern: 'results-container.sarif')
                    }
                }
                stage('Snyk IaC') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'snyk iac test --report --sarif-file-output=results-iac.sarif'
                        }
                        recordIssues tool: sarif(name: 'Snyk IaC', id: 'snyk-iac', pattern: 'results-iac.sarif')
                    }
                }
                stage('Snyk Delta') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'snyk test --json --print-deps | snyk-delta --baselineOrg f84edda3-a197-4214-90b7-0c3f6823925f --baselineProject affd22e5-f3ac-4d49-91b4-938f2ad527cd'
                        }
                    }
                }
            }
        }
    }
} 