pipeline {
    agent { label 'docker-gradle-linux' }

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timestamps()
        ansiColor('xterm')
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
    }

    parameters {
        string(
            name: 'GIT_URL',
            defaultValue: 'https://github.com/champanu/testrepo.git',
            description: 'GitHub repository URL'
        )

        string(
            name: 'GIT_REF',
            defaultValue: 'main',
            description: 'Branch, tag, or commit'
        )

        string(
            name: 'ECR_REPOSITORY',
            defaultValue: 'example-service',
            description: 'Amazon ECR repository name'
        )

        string(
            name: 'AWS_REGION',
            defaultValue: 'ap-south-1',
            description: 'AWS region'
        )

        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Optional image tag. Defaults to Git commit SHA'
        )

        booleanParam(
            name: 'PUSH_IMAGE',
            defaultValue: true,
            description: 'Push image to ECR after all security gates pass'
        )
    }

    environment {
        SONARQUBE_SERVER = 'sonarqube-production'

        GITHUB_CREDENTIALS_ID = 'github-readonly'
        AWS_CREDENTIALS_ID = 'aws-ecr-production'
        DEPENDENCY_TRACK_CREDENTIALS_ID = 'dependency-track-api-key'

        DEPENDENCY_TRACK_URL = 'https://dependency-track.example.internal'
        DEPENDENCY_TRACK_PROJECT_NAME = 'example-service'

        // One or more High/Critical findings fails the build
        DT_FAILED_TOTAL_CRITICAL = '1'
        DT_FAILED_TOTAL_HIGH = '1'

        TRIVY_VERSION = '0.59.1'
        DOCKER_BUILDKIT = '1'
        GIT_TERMINAL_PROMPT = '0'
    }

    stages {
        stage('Checkout GitHub Code') {
            steps {
                deleteDir()

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.GIT_REF}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [
                        [
                            $class: 'CloneOption',
                            depth: 0,
                            noTags: false,
                            shallow: false,
                            timeout: 20
                        ]
                    ],
                    userRemoteConfigs: [
                        [
                            url: params.GIT_URL,
                            credentialsId: GITHUB_CREDENTIALS_ID
                        ]
                    ]
                ])

                sh '''
                    #!/usr/bin/env bash
                    set -Eeuo pipefail

                    test -f gradlew || {
                        echo "ERROR: gradlew was not found"
                        exit 1
                    }

                    test -f Dockerfile || {
                        echo "ERROR: Dockerfile was not found"
                        exit 1
                    }

                    chmod +x gradlew

                    git rev-parse HEAD > .git-commit
                    git status --short
                '''

                script {
                    env.GIT_COMMIT_SHA = readFile('.git-commit').trim()

                    env.IMAGE_TAG_VALUE =
                        params.IMAGE_TAG?.trim()
                            ? params.IMAGE_TAG.trim()
                            : env.GIT_COMMIT_SHA

                    env.IMAGE_URI =
                        "${params.AWS_REGION}.dkr.ecr.amazonaws.com/" +
                        "${params.ECR_REPOSITORY}:${env.IMAGE_TAG_VALUE}"

                    env.DEPENDENCY_TRACK_PROJECT_VERSION =
                        "${env.JOB_NAME}-${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Gradle Build and Unit Tests') {
            steps {
                sh '''
                    #!/usr/bin/env bash
                    set -Eeuo pipefail

                    ./gradlew \
                        --no-daemon \
                        --stacktrace \
                        clean \
                        test \
                        build
                '''
            }

            post {
                always {
                    junit(
                        testResults: '**/build/test-results/test/*.xml',
                        allowEmptyResults: true
                    )

                    archiveArtifacts(
                        artifacts: '**/build/libs/*.jar',
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
            }
        }

        stage('SonarQube Code Scan') {
            steps {
                withSonarQubeEnv(SONARQUBE_SERVER) {
                    sh '''
                        #!/usr/bin/env bash
                        set -Eeuo pipefail

                        ./gradlew \
                            --no-daemon \
                            --stacktrace \
                            sonarqube \
                            -Dsonar.qualitygate.wait=false \
                            -Dsonar.projectVersion="${GIT_COMMIT_SHA}"
                    '''
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Generate Dependency SBOM') {
            steps {
                sh '''
                    #!/usr/bin/env bash
                    set -Eeuo pipefail

                    ./gradlew \
                        --no-daemon \
                        --stacktrace \
                        cyclonedxBom

                    test -s build/reports/cyclonedx/bom.xml
                '''

                archiveArtifacts(
                    artifacts: 'build/reports/cyclonedx/bom.xml,build/reports/cyclonedx/bom.json',
                    allowEmptyArchive: false,
                    fingerprint: true
                )
            }
        }

        stage('Dependency-Track Vulnerability Gate') {
            steps {
                withCredentials([
                    string(
                        credentialsId: DEPENDENCY_TRACK_CREDENTIALS_ID,
                        variable: 'DT_API_KEY'
                    )
                ]) {
                    script {
                        dependencyTrackPublisher(
                            artifact: 'build/reports/cyclonedx/bom.xml',
                            projectName: DEPENDENCY_TRACK_PROJECT_NAME,
                            projectVersion: DEPENDENCY_TRACK_PROJECT_VERSION,
                            synchronous: true,
                            dependencyTrackUrl: DEPENDENCY_TRACK_URL,
                            dependencyTrackApiKey: DT_API_KEY,
                            autoCreateProjects: true,
                            failOnViolationFail: true,

                            // Fail when total Critical findings >= 1
                            failedTotalCritical: DT_FAILED_TOTAL_CRITICAL.toInteger(),

                            // Fail when total High findings >= 1
                            failedTotalHigh: DT_FAILED_TOTAL_HIGH.toInteger()
                        )
                    }
                }
            }
        }

        stage('Install and Scan Source with Trivy') {
            steps {
                sh '''
                    #!/usr/bin/env bash
                    set -Eeuo pipefail

                    mkdir -p .tools

                    if ! command -v trivy >/dev/null 2>&1; then
                        if [ ! -x .tools/trivy ]; then
                            curl \
                                --fail \
                                --silent \
                                --show-error \
                                --location \
                                "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" \
                                | tar -xz -C .tools trivy
                        fi

                        TRIVY=".tools/trivy"
                    else
                        TRIVY="$(command -v trivy)"
                    fi

                    "${TRIVY}" --version

                    "${TRIVY}" fs \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --scanners vuln,secret,misconfig \
                        .
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    #!/usr/bin/env bash
                    set -Eeuo pipefail

                    docker build \
                        --pull \
                        --label "org.opencontainers.image.revision=${GIT_COMMIT_SHA}" \
                        --label "org.opencontainers.image.source=${GIT_URL}" \
                        --tag "${IMAGE_URI}" \
                        .
                '''
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh '''
                    #!/usr/bin/env bash
                    set -Eeuo pipefail

                    if command -v trivy >/dev/null 2>&1; then
                        TRIVY="$(command -v trivy)"
                    else
                        TRIVY=".tools/trivy"
                    fi

                    "${TRIVY}" image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --scanners vuln,secret,misconfig \
                        "${IMAGE_URI}"
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            when {
                expression {
                    return params.PUSH_IMAGE
                }
            }

            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: AWS_CREDENTIALS_ID
                    ]
                ]) {
                    sh '''
                        #!/usr/bin/env bash
                        set -Eeuo pipefail

                        ACCOUNT_ID="$(
                            aws sts get-caller-identity \
                                --query Account \
                                --output text
                        )"

                        REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        FULL_IMAGE_URI="${REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG_VALUE}"

                        aws ecr describe-repositories \
                            --repository-names "${ECR_REPOSITORY}" \
                            --region "${AWS_REGION}" \
                            >/dev/null

                        aws ecr get-login-password \
                            --region "${AWS_REGION}" \
                            | docker login \
                                --username AWS \
                                --password-stdin "${REGISTRY}"

                        docker tag \
                            "${IMAGE_URI}" \
                            "${FULL_IMAGE_URI}"

                        docker push "${FULL_IMAGE_URI}"

                        docker image inspect "${FULL_IMAGE_URI}" \
                            --format '{{index .RepoDigests 0}}' \
                            | tee ecr-image-digest.txt
                    '''

                    archiveArtifacts(
                        artifacts: 'ecr-image-digest.txt',
                        allowEmptyArchive: false,
                        fingerprint: true
                    )
                }
            }
        }
    }

    post {
        always {
            sh '''
                #!/usr/bin/env bash
                set +e

                docker image rm "${IMAGE_URI}" >/dev/null 2>&1 || true

                rm -rf .tools >/dev/null 2>&1 || true
            '''

            archiveArtifacts(
                artifacts: '**/build/reports/**,**/build/test-results/**',
                allowEmptyArchive: true
            )

            deleteDir()
        }

        success {
            echo '''
            Pipeline passed successfully.

            Completed:
            - Gradle build and tests
            - SonarQube analysis
            - SonarQube quality gate
            - Dependency-Track SBOM vulnerability gate
            - Trivy source scan
            - Docker image build
            - Trivy image scan
            - ECR image push
            '''
        }

        failure {
            echo '''
            Pipeline failed.

            Docker image push is blocked when any required build or security
            gate fails.
            '''
        }
    }
}
