// CI frontend SamaFile : oxlint, npm audit, build Vite, image Docker, Trivy, Docker Hub.
// Job Jenkins : Pipeline script from SCM, script path Jenkinsfile.
// Credential : dockerhub-credentials (utilisateur + jeton Docker Hub).

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(name: 'PUBLISH', defaultValue: false, description: 'Publier l image sur Docker Hub après les scans')
    }

    environment {
        LOCAL_IMAGE = 'samafile-frontend:ci'
        CI_IMAGE = 'samafile-frontend-ci:build'
        TRIVY_IMAGE = 'aquasec/trivy:0.58.1'
        NODE_IMAGE = 'node:22-alpine'
    }

    stages {
        stage('Préparation') {
            steps {
                sh 'mkdir -p reports'
                sh 'docker build -f Dockerfile.ci -t "$CI_IMAGE" .'
            }
        }

        stage('Lint oxlint') {
            steps {
                sh '''
                set -eu
                docker create --name "samafile-lint-${BUILD_NUMBER}" "$CI_IMAGE" npm run lint
                set +e
                docker start -a "samafile-lint-${BUILD_NUMBER}"
                lint_code=$?
                set -e
                docker rm -f "samafile-lint-${BUILD_NUMBER}" >/dev/null
                exit "$lint_code"
                '''
            }
        }

        stage('Audit npm') {
            steps {
                sh '''
                set -eu
                docker create --name "samafile-audit-${BUILD_NUMBER}" "$CI_IMAGE" \
                    sh -c 'npm audit --audit-level=high --json > reports/npm-audit.json'
                set +e
                docker start -a "samafile-audit-${BUILD_NUMBER}"
                audit_code=$?
                set -e
                docker cp "samafile-audit-${BUILD_NUMBER}:/app/reports/npm-audit.json" reports/npm-audit.json || true
                docker rm -f "samafile-audit-${BUILD_NUMBER}" >/dev/null
                if [ "$audit_code" -ne 0 ]; then exit "$audit_code"; fi
                docker run --rm "$CI_IMAGE" npm audit --audit-level=high || true
                '''
            }
        }

        stage('Hadolint') {
            steps {
                sh '''
                    set -eu
                    docker run --rm -i hadolint/hadolint:v2.12.0-alpine hadolint --failure-threshold error - < Dockerfile
                '''
            }
        }

        stage('Image Docker') {
            steps {
                sh 'docker build -t "$LOCAL_IMAGE" .'
            }
        }

        stage('Trivy image et sources') {
            steps {
                sh '''
                    set -eu
                    run_trivy() {
                        docker run --rm "$@"
                    }
                    if run_trivy \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        "$TRIVY_IMAGE" image \
                        --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
                        --format table \
                        "$LOCAL_IMAGE" > reports/trivy-image.txt 2>&1; then
                        image_code=0
                    else
                        image_code=$?
                    fi
                    echo "----- Trivy image -----"
                    cat reports/trivy-image.txt || true
                    if run_trivy \
                        -v "$PWD:/src" \
                        "$TRIVY_IMAGE" fs \
                        --scanners vuln,secret,misconfig \
                        --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
                        --skip-dirs /src/node_modules --skip-dirs /src/dist --skip-dirs /src/.git \
                        --skip-files /src/Dockerfile.test \
                        --format table \
                        /src > reports/trivy-fs.txt 2>&1; then
                        fs_code=0
                    else
                        fs_code=$?
                    fi
                    echo "----- Trivy fs -----"
                    cat reports/trivy-fs.txt || true
                    test "$image_code" = 0
                    test "$fs_code" = 0
                '''
            }
        }

        stage('Publication Docker Hub') {
            when {
                expression { return params.PUBLISH }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set -eu
                        echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        image="${DOCKERHUB_USER}/samafile-frontend"
                        docker tag "$LOCAL_IMAGE" "${image}:${BUILD_NUMBER}"
                        docker push "${image}:${BUILD_NUMBER}"
                        sha="$(git rev-parse --short HEAD 2>/dev/null || true)"
                        if [ -n "$sha" ]; then
                            docker tag "$LOCAL_IMAGE" "${image}:${sha}"
                            docker push "${image}:${sha}"
                        fi
                        branch="${BRANCH_NAME:-${GIT_BRANCH:-}}"
                        if printf '%s' "$branch" | grep -Eq '(^|/)(main|master|develop)$'; then
                            docker tag "$LOCAL_IMAGE" "${image}:latest"
                            docker push "${image}:latest"
                        fi
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f "samafile-lint-${BUILD_NUMBER}" "samafile-audit-${BUILD_NUMBER}" >/dev/null 2>&1 || true
            '''
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
