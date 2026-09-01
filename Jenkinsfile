// Jenkins pipeline for the ReqRes API automation suite.
//
// Approach:
//   The official postman/newman image sets ENTRYPOINT ["newman"], which does
//   not play well with Jenkins' `agent { docker }` model (the plugin tries to
//   keep the container alive with `cat`, producing `newman cat`). So instead of
//   running the pipeline *inside* the Newman container, we run on the default
//   agent and invoke Newman via explicit `docker build` + `docker run` steps.
//   This keeps entrypoint behaviour intact and mirrors how you'd actually run
//   Newman in a container in production.
//
// Assumes a Docker-in-Docker setup (DOCKER_HOST -> tcp://docker:2376), i.e. the
// same environment used by the Selenium pipeline in this portfolio.

pipeline {
    agent any

    environment {
        NEWMAN_IMAGE   = 'reqres-newman:ci'
        COLLECTION     = 'collections/reqres-api-tests.postman_collection.json'
        ENVIRONMENT    = 'environments/reqres.postman_environment.json'
        REPORT_DIR     = 'newman-reports'

        // ReqRes API key, injected from a Jenkins "Secret text" credential.
        // Create it under Manage Jenkins > Credentials with ID 'reqres-api-key'.
        // The committed environment file ships with an empty apiKey; the real
        // value is supplied here at run time via --env-var (see the run stage).
        REQRES_API_KEY = credentials('reqres-api-key')
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 15, unit: 'MINUTES')
    }

    triggers {
        // Poll SCM for new commits every 5 minutes.
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
                // Build the custom Newman image (Newman + htmlextra reporter).
                sh 'docker build -f Dockerfile.newman -t $NEWMAN_IMAGE .'
            }
        }

        stage('Run API tests') {
            steps {
                // Ensure the report directory exists on the workspace side and
                // is writable by the container's user.
                sh 'mkdir -p $REPORT_DIR && chmod 777 $REPORT_DIR'

                // Mount the workspace into /etc/newman (the image's WORKDIR) and
                // run the collection. Reporters:
                //   cli       -> human-readable console output
                //   junit     -> consumed by Jenkins' JUnit plugin (trends)
                //   htmlextra -> rich, shareable HTML report (archived below)
                //
                // `--entrypoint=""` is NOT needed here because we call `newman`'s
                // own subcommand `run` directly, which the entrypoint expects.
                // The API key is passed into the container as an env var and
                // then handed to Newman via --env-var, overriding the empty
                // apiKey in the committed environment file. The key is never
                // written to the workspace or the collection.
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
            // Publish JUnit results so Jenkins shows pass/fail trends per build.
            junit allowEmptyResults: true, testResults: "${REPORT_DIR}/junit-report.xml"

            // Archive the HTML report as a build artifact.
            archiveArtifacts artifacts: "${REPORT_DIR}/report.html", allowEmptyArchive: true, fingerprint: true
        }
        success {
            echo 'API test suite passed.'
        }
        failure {
            echo 'API test suite failed - check the JUnit results and the archived HTML report.'
        }
        cleanup {
            // Best-effort cleanup of the built image to keep the DinD daemon tidy.
            sh 'docker rmi $NEWMAN_IMAGE || true'
        }
    }
}
