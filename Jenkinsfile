pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "2022bcd0017/wine-quality-api:latest"
        CONTAINER_NAME = "Lightning_macqueen"
        API_PORT = "8030"
        API_BASE = "http://host.docker.internal:8030"
        TIMEOUT_SECONDS = "30"
    }

    stages {

        // ─────────────────────────────────────────────
        // STAGE 1: Pull Image
        // ─────────────────────────────────────────────
        stage('Pull Image') {
            steps {
                echo "Pulling Docker image: ${DOCKER_IMAGE}"
                sh "docker pull ${DOCKER_IMAGE}"
                sh "docker image inspect ${DOCKER_IMAGE} > /dev/null 2>&1 && echo 'Image verified successfully'"
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 2: Run Container
        // ─────────────────────────────────────────────
        stage('Run Container') {
            steps {
                echo "Starting container: ${CONTAINER_NAME}"
                sh """
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${API_PORT}:8030 \
                        ${DOCKER_IMAGE}
                """
                echo "Container started successfully"
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 3: Wait for Service Readiness
        // ─────────────────────────────────────────────
        stage('Wait for Service Readiness') {
            steps {
                echo "Waiting for API to become ready..."
                script {
                    def ready = false
                    def elapsed = 0
                    def interval = 3

                    while (!ready && elapsed < TIMEOUT_SECONDS.toInteger()) {
                        try {
                            def response = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' ${API_BASE}/health",
                                returnStdout: true
                            ).trim()

                            if (response == "200") {
                                echo "API is ready! (took ${elapsed}s)"
                                ready = true
                            } else {
                                echo "API returned status ${response}, retrying in ${interval}s..."
                                sleep(interval)
                                elapsed += interval
                            }
                        } catch (Exception e) {
                            echo "API not yet available, retrying in ${interval}s..."
                            sleep(interval)
                            elapsed += interval
                        }
                    }

                    if (!ready) {
                        error("API did not become ready within ${TIMEOUT_SECONDS} seconds")
                    }
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 4: Send Valid Inference Request
        // ─────────────────────────────────────────────
        stage('Send Valid Inference Request') {
            steps {
                echo "Sending valid inference request..."
                script {
                    def validInput = readFile('test_inputs/valid_input.json').trim()

                    def response = sh(
                        script: """
                            curl -s -w "\\nHTTP_STATUS:%{http_code}" \
                                -X POST \
                                -H "Content-Type: application/json" \
                                -d '${validInput}' \
                                ${API_BASE}/predict
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Raw API Response: ${response}"

                    // Extract HTTP status code
                    def httpStatus = response.split("HTTP_STATUS:")[1].trim()
                    def responseBody = response.split("HTTP_STATUS:")[0].trim()

                    echo "HTTP Status: ${httpStatus}"
                    echo "Response Body: ${responseBody}"

                    // Validation 1: HTTP status must be 2xx
                    if (!httpStatus.startsWith("2")) {
                        error("VALIDATION FAILED: Expected 2xx status, got ${httpStatus}")
                    }
                    echo "HTTP status check passed (${httpStatus})"

                    // Validation 2: Response must contain 'wine_quality' field
                    if (!responseBody.contains("wine_quality")) {
                        error("VALIDATION FAILED: Response does not contain 'wine_quality' field. Got: ${responseBody}")
                    }
                    echo "'wine_quality' field exists in response"

                    // Validation 3: Prediction value must be numeric
                    def predMatch = responseBody =~ /"wine_quality"\s*:\s*([\d.eE+\-]+)/
                    if (!predMatch) {
                        error("VALIDATION FAILED: Prediction value is not numeric. Got: ${responseBody}")
                    }
                    echo "Prediction value is numeric: ${predMatch[0][1]}"
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 5: Send Invalid Request
        // ─────────────────────────────────────────────
        stage('Send Invalid Request') {
            steps {
                echo "Sending invalid/malformed inference request..."
                script {
                    def invalidInput = readFile('test_inputs/invalid_input.json').trim()

                    def response = sh(
                        script: """
                            curl -s -w "\\nHTTP_STATUS:%{http_code}" \
                                -X POST \
                                -H "Content-Type: application/json" \
                                -d '${invalidInput}' \
                                ${API_BASE}/predict
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Raw API Response: ${response}"

                    def httpStatus = response.split("HTTP_STATUS:")[1].trim()
                    def responseBody = response.split("HTTP_STATUS:")[0].trim()

                    echo "HTTP Status: ${httpStatus}"
                    echo "Response Body: ${responseBody}"

                    // Validation: API should return 4xx error for bad input
                    if (httpStatus.startsWith("2")) {
                        error("VALIDATION FAILED: Expected 4xx error for invalid input, but got ${httpStatus}")
                    }
                    echo "API correctly returned error status (${httpStatus}) for invalid input"

                    // Validation: Error response should contain a meaningful message
                    if (!responseBody.contains("detail") && !responseBody.contains("error")) {
                        error("VALIDATION FAILED: Error response is not meaningful. Got: ${responseBody}")
                    }
                    echo "Error response contains meaningful message"
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 6: Stop Container
        // ─────────────────────────────────────────────
        stage('Stop Container') {
            steps {
                echo "Stopping and removing container: ${CONTAINER_NAME}"
                sh "docker stop ${CONTAINER_NAME} || true"
                sh "docker rm ${CONTAINER_NAME} || true"
                echo "Container stopped and removed"

                // Verify no leftover containers
                script {
                    def running = sh(
                        script: "docker ps --filter name=${CONTAINER_NAME} -q",
                        returnStdout: true
                    ).trim()
                    if (running) {
                        error("Container is still running after stop attempt!")
                    }
                    echo "No leftover containers found"
                }
            }
        }
    }

    // ─────────────────────────────────────────────
    // STAGE 7: Pipeline Result (Post Actions)
    // ─────────────────────────────────────────────
    post {
        success {
            echo "============================================"
            echo "PIPELINE PASSED: All validation tests passed successfully!"
            echo "============================================"
        }
        failure {
            echo "============================================"
            echo "PIPELINE FAILED: One or more validation checks failed."
            echo "============================================"
            // Cleanup in case of failure
            sh "docker stop ${CONTAINER_NAME} || true"
            sh "docker rm ${CONTAINER_NAME} || true"
        }
        always {
            echo "Pipeline completed. Check logs above for details."
        }
    }
}