// Prerequisites this pipeline assumes are already in place:
//  - k8s secret "ghcr-regcred" (type kubernetes.io/dockerconfigjson) in the jenkins
//    namespace, scoped to ghcr.io, used by Kaniko/crane to push images
//  - Jenkins credential "github-pat"    (Username with password) - PAT with repo scope,
//    used to push the image-tag bump to the kube-prom-stack gitops repo
//  - Jenkins credential "cosign-key"      (Secret file)   - cosign.key from `cosign generate-key-pair`
//  - Jenkins credential "cosign-password" (Secret text)   - password for that key
//  - manifest file manifests/demo-app/deployment.yaml in kube-prom-stack, containing a
//    line "image: ghcr.io/ebak555/demo-app:<tag>" for this pipeline to sed/bump

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.9-eclipse-temurin-21
      command: ["sleep"]
      args: ["99d"]
      resources:
        requests: { cpu: 250m, memory: 512Mi }
        limits:   { cpu: 1000m, memory: 1Gi }
    - name: trivy
      image: aquasec/trivy:0.55.0
      command: ["sleep"]
      args: ["99d"]
      resources:
        requests: { cpu: 100m, memory: 256Mi }
        limits:   { cpu: 500m, memory: 512Mi }
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.23.2-debug
      command: ["sleep"]
      args: ["99d"]
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
      resources:
        requests: { cpu: 250m, memory: 512Mi }
        limits:   { cpu: 1000m, memory: 1Gi }
    - name: crane
      image: gcr.io/go-containerregistry/crane:debug
      command: ["sleep"]
      args: ["99d"]
      env:
        - name: DOCKER_CONFIG
          value: /kaniko/.docker
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
      resources:
        requests: { cpu: 100m, memory: 128Mi }
        limits:   { cpu: 300m, memory: 256Mi }
    - name: syft
      image: anchore/syft:latest
      command: ["sleep"]
      args: ["99d"]
      resources:
        requests: { cpu: 100m, memory: 256Mi }
        limits:   { cpu: 500m, memory: 512Mi }
    - name: cosign
      image: ghcr.io/sigstore/cosign/cosign:v2.4.1
      command: ["sleep"]
      args: ["99d"]
      resources:
        requests: { cpu: 100m, memory: 128Mi }
        limits:   { cpu: 300m, memory: 256Mi }
    - name: git
      image: alpine/git:latest
      command: ["sleep"]
      args: ["99d"]
      resources:
        requests: { cpu: 50m, memory: 64Mi }
        limits:   { cpu: 200m, memory: 128Mi }
  volumes:
    - name: docker-config
      secret:
        secretName: ghcr-regcred
        items:
          - key: .dockerconfigjson
            path: config.json
'''
        }
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        REGISTRY       = 'ghcr.io'
        IMAGE          = 'ghcr.io/ebak555/demo-app'
        GITOPS_REPO    = 'github.com/ebak555/kube-prom-stack.git'
        GITOPS_MANIFEST = 'manifests/demo-app/deployment.yaml'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
                echo "Building ${env.IMAGE}:${env.GIT_SHA}"
            }
        }

        stage('Build & Unit Test') {
            steps {
                container('maven') {
                    sh 'mvn -B clean verify'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Dependency Scan (SCA)') {
            steps {
                container('trivy') {
                    sh '''
                        trivy fs \
                          --scanners vuln \
                          --severity CRITICAL,HIGH \
                          --exit-code 1 \
                          --format table \
                          -o trivy-fs-report.txt \
                          .
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Config & Secret Scan (SAST)') {
            steps {
                container('trivy') {
                    sh '''
                        trivy config \
                          --severity CRITICAL,HIGH \
                          --exit-code 1 \
                          --format table \
                          -o trivy-config-report.txt \
                          .
                        trivy fs \
                          --scanners secret \
                          --exit-code 1 \
                          --format table \
                          -o trivy-secret-report.txt \
                          .
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-config-report.txt,trivy-secret-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Build Image') {
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor \
                          --context "$(pwd)" \
                          --dockerfile Dockerfile \
                          --no-push \
                          --tarball=image.tar \
                          --destination="${IMAGE}:${GIT_SHA}"
                    '''
                }
            }
        }

        stage('Image Scan') {
            steps {
                container('trivy') {
                    sh '''
                        trivy image \
                          --input image.tar \
                          --severity CRITICAL,HIGH \
                          --exit-code 1 \
                          --format table \
                          -o trivy-image-report.txt \
                          .
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Generate SBOM') {
            steps {
                container('syft') {
                    sh 'syft docker-archive:image.tar -o cyclonedx-json=sbom.json'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'sbom.json', allowEmptyArchive: true
                }
            }
        }

        stage('Push Image') {
            steps {
                container('crane') {
                    sh '''
                        crane push image.tar "${IMAGE}:${GIT_SHA}"
                        crane tag "${IMAGE}:${GIT_SHA}" latest
                    '''
                    script {
                        env.IMAGE_DIGEST = sh(
                            script: 'crane digest "${IMAGE}:${GIT_SHA}"',
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }

        stage('Sign Image') {
            steps {
                container('cosign') {
                    withCredentials([
                        file(credentialsId: 'cosign-key', variable: 'COSIGN_KEY_PATH'),
                        string(credentialsId: 'cosign-password', variable: 'COSIGN_PASSWORD')
                    ]) {
                        sh '''
                            cosign sign --yes \
                              --key "$COSIGN_KEY_PATH" \
                              -a git-sha="${GIT_SHA}" \
                              -a build-url="${BUILD_URL}" \
                              "${IMAGE}@${IMAGE_DIGEST}"
                        '''
                    }
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                container('git') {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-pat',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh '''
                            rm -rf gitops
                            git clone "https://${GIT_USER}:${GIT_TOKEN}@${GITOPS_REPO}" gitops
                            cd gitops

                            sed -i -E "s#(image:\\s*${IMAGE}:).*#\\1${GIT_SHA}#" "${GITOPS_MANIFEST}"

                            git config user.name "jenkins-ci"
                            git config user.email "jenkins-ci@example.com"
                            git add "${GITOPS_MANIFEST}"
                            git commit -m "deploy demo-app@${GIT_SHA}"
                            git push origin HEAD:main
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline succeeded: ${IMAGE}:${GIT_SHA} built, scanned, signed, and deployed via GitOps."
        }
        failure {
            echo "Pipeline failed - check the archived Trivy reports for the failing gate."
        }
    }
}
