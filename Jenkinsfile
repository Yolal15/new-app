pipeline {
    agent any

    environment {
        // Docker Image Location
        DOCKER_IMAGE = "skmds/automated-maintenance-suite"
        DOCKER_TAG = "v1.0.0"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"
        COVERAGE_MIN = 80   // Minimum required coverage percent
        KUBECONFIG = "${HOME}/.kube/config"
        K8S_DEPLOY_FILE = "k8s/deployment.yaml"
        K8S_DEPLOYMENT_NAME = "automated-maintenance-suite"

    }

    options {
        timestamps()
        skipDefaultCheckout()
        ansiColor('xterm')
    }

    stages {

        stage('Clone') {
            steps {
                echo "=== Stage 1: Cloning repository ==="
                checkout scm
                sh 'ls -alh'
            }
        }

        stage('Build') {
            steps {
                echo "=== Stage 2: Installing dependencies and building app ==="
                sh 'npm install'
                sh 'npm run build'
            }
        }

        stage('Unit Test') {
            steps {
                echo "=== Stage 3: Running unit tests and checking coverage ==="
                // Run all tests and generate coverage reports
                sh 'npm run test -- --coverage --watchAll=false'
            }
            post {
                always {
                    // Publish JUnit test results, if you have them
                    junit allowEmptyResults: true, testResults: '**/junit.xml'
                    // Publish coverage report, if using Cobertura
                    cobertura coberturaReportFile: '**/cobertura-coverage.xml', onlyStable: false
                }
                success {
                    script {
                        // Parse JSON for coverage summary
                        def covReport = readJSON file: 'coverage/coverage-summary.json'
                        def totalCoverage = covReport.total.lines.pct
                        echo "Coverage: ${totalCoverage}%"
                        if (totalCoverage < env.COVERAGE_MIN.toInteger()) {
                            error "Unit Test Coverage ${totalCoverage}% BELOW minimum threshold (${COVERAGE_MIN}%)"
                        } else {
                            echo "Coverage threshold met: ${totalCoverage}% >= ${COVERAGE_MIN}%"
                        }
                    }
                }
            }
        }

        stage('Container Push') {
            steps {
                echo "=== Stage 4: Building and pushing Docker image ==="
                script {
                    docker.withRegistry('', env.DOCKER_CREDENTIALS_ID) {
                        def image = docker.build("${env.DOCKER_IMAGE}:${env.DOCKER_TAG}")
                        image.push()
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo "=== Stage 5: Deploying to local Minikube ==="
                // Update image tag in deployment file, if needed
                script {
                    sh """
                      kubectl --kubeconfig=${env.KUBECONFIG} set image deployment/${env.K8S_DEPLOYMENT_NAME} *=${env.DOCKER_IMAGE}:${env.DOCKER_TAG} --record || \\
                      kubectl --kubeconfig=${env.KUBECONFIG} apply -f ${env.K8S_DEPLOY_FILE}
                    """
                    sh "kubectl --kubeconfig=${env.KUBECONFIG} rollout status deployment/${env.K8S_DEPLOYMENT_NAME}"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed (success or failure). Cleaning up workspace..."
            cleanWs()
        }
        success {
            echo "All pipeline stages passed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check the stage logs above."
        }
    }
}

// === Helper groovy functions (optional) ===
def readJSON(args) {
    return new groovy.json.JsonSlurper().parse(new File(args.file))
}
