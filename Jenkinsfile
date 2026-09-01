pipeline {
    agent any

    environment {
        NEWMAN_IMAGE   = 'reqres-newman:ci'
        COLLECTION     = 'collections/reqres-api-tests.postman_collection.json'
        ENVIRONMENT    = 'environments/reqres.postman_environment.json'
        REPORT_DIR     = 'newman-reports'
        REQRES_API_KEY = credentials('reqres-api-key')
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 15, unit: 'MINUTES')
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Newman image') {
            steps {
                sh 'docker build -f Dockerfile.newman -t $NEWMAN_IMAGE .'
            }
        }

        stage('Run API tests') {
            steps {
                sh 'mkdir -p $REPORT_DIR && chmod 777 $REPORT_DIR'
                sh '''
                    docker run --rm \
                        -v "$WORKSPACE:/etc/newman" \
                        -e REQRES_API_KEY \
                        $NEWMAN_IMAGE \
                        run "$COLLECTION" \
                        -e "$ENVIRONMENT" \
                        --env-var "apiKey=$REQRES_API_KEY" \
                        -r cli,junit,htmlextra \
                        --reporter-junit-export "$REPORT_DIR/junit-report.xml" \
                        --reporter-htmlextra-export "$REPORT_DIR/report.html" \
                        --color off
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: "${REPORT_DIR}/junit-report.xml"
            archiveArtifacts artifacts: "${REPORT_DIR}/report.html", allowEmptyArchive: true, fingerprint: true
        }
        success {
            echo 'API test suite passed.'
        }
        failure {
            echo 'API test suite failed - check the JUnit results and the archived HTML report.'
        }
        cleanup {
            sh 'docker rmi $NEWMAN_IMAGE || true'
        }
    }
}
