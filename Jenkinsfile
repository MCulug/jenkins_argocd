pipeline {
    agent {
        kubernetes {
            label 'argocd-installer'
            defaultContainer 'kubectl'
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: argocd-installer
spec:
  containers:
    - name: kubectl
      image: bitnami/kubectl:latest
      command:
        - cat
      tty: true
      securityContext:
        runAsUser: 0
'''
        }
    }

    environment {
        CLUSTER_FILE = "cluster.yaml"
        PATH = "/usr/local/bin:${env.PATH}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Repo Files') {
            steps {
                container('kubectl') {
                    sh '''
                        set -e

                        echo "Workspace:"
                        pwd
                        ls -la

                        if [ ! -f "${CLUSTER_FILE}" ]; then
                          echo "ERROR: ${CLUSTER_FILE} not found"
                          exit 1
                        fi

                        echo "cluster.yaml found."
                        cat "${CLUSTER_FILE}"
                    '''
                }
            }
        }

        stage('Load Cluster Config') {
            steps {
                container('kubectl') {
                    script {
                        env.CLUSTER_NAME = sh(
                            script: "grep '^cluster_name:' ${CLUSTER_FILE} | awk '{print \$2}'",
                            returnStdout: true
                        ).trim()

                        env.KUBECONFIG_CREDENTIAL_ID = sh(
                            script: "grep -A5 '^kubeconfig:' ${CLUSTER_FILE} | awk '/credential_id:/ {print \$2; exit}'",
                            returnStdout: true
                        ).trim()

                        env.ARGOCD_NAMESPACE = sh(
                            script: "grep -A10 '^argocd:' ${CLUSTER_FILE} | awk '/namespace:/ {print \$2; exit}'",
                            returnStdout: true
                        ).trim()

                        env.ARGOCD_MANIFEST_URL = sh(
                            script: "grep -A10 '^argocd:' ${CLUSTER_FILE} | awk '/install_manifest_url:/ {print \$2; exit}'",
                            returnStdout: true
                        ).trim()

                        env.ARGOCD_SERVICE_TYPE = sh(
                            script: "grep -A10 '^argocd:' ${CLUSTER_FILE} | awk '/server_service_type:/ {print \$2; exit}'",
                            returnStdout: true
                        ).trim()

                        env.TIMEOUT_SECONDS = sh(
                            script: "grep -A5 '^checks:' ${CLUSTER_FILE} | awk '/timeout_seconds:/ {print \$2; exit}'",
                            returnStdout: true
                        ).trim()
                    }

                    echo "Cluster name: ${CLUSTER_NAME}"
                    echo "Kubeconfig credential ID: ${KUBECONFIG_CREDENTIAL_ID}"
                    echo "Argo CD namespace: ${ARGOCD_NAMESPACE}"
                    echo "Argo CD manifest: ${ARGOCD_MANIFEST_URL}"
                    echo "Argo CD service type: ${ARGOCD_SERVICE_TYPE}"
                    echo "Timeout seconds: ${TIMEOUT_SECONDS}"
                }
            }
        }

        stage('Prepare Kubeconfig') {
            steps {
                container('kubectl') {
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh '''
                            set -e

                            mkdir -p .kube
                            cp "${KUBECONFIG_FILE}" .kube/config
                            chmod 600 .kube/config

                            echo "Kubeconfig injected from Jenkins credential."
                            kubectl --kubeconfig=.kube/config config current-context || true
                            kubectl --kubeconfig=.kube/config config view --minify
                        '''
                    }
                }
            }
        }

        stage('Check Cluster Access') {
            steps {
                container('kubectl') {
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh '''
                            set -e

                            mkdir -p .kube
                            cp "${KUBECONFIG_FILE}" .kube/config
                            chmod 600 .kube/config

                            echo "Checking cluster access..."
                            kubectl --kubeconfig=.kube/config cluster-info
                            kubectl --kubeconfig=.kube/config get nodes -o wide

                            echo "Cluster access check passed."
                        '''
                    }
                }
            }
        }

        stage('Install Argo CD') {
            steps {
                container('kubectl') {
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh '''
                            set -e

                            mkdir -p .kube
                            cp "${KUBECONFIG_FILE}" .kube/config
                            chmod 600 .kube/config

                            if [ -z "${ARGOCD_NAMESPACE}" ]; then
                              ARGOCD_NAMESPACE="argocd"
                            fi

                            if [ -z "${ARGOCD_MANIFEST_URL}" ]; then
                              ARGOCD_MANIFEST_URL="https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                            fi

                            if [ -z "${ARGOCD_SERVICE_TYPE}" ]; then
                              ARGOCD_SERVICE_TYPE="ClusterIP"
                            fi

                            echo "Installing Argo CD..."
                            echo "Namespace: ${ARGOCD_NAMESPACE}"
                            echo "Manifest: ${ARGOCD_MANIFEST_URL}"
                            echo "Service type: ${ARGOCD_SERVICE_TYPE}"

                            kubectl --kubeconfig=.kube/config get namespace "${ARGOCD_NAMESPACE}" >/dev/null 2>&1 || \
                              kubectl --kubeconfig=.kube/config create namespace "${ARGOCD_NAMESPACE}"

                            kubectl --kubeconfig=.kube/config apply \
                              -n "${ARGOCD_NAMESPACE}" \
                              --server-side \
                              --force-conflicts \
                              -f "${ARGOCD_MANIFEST_URL}"

                            if [ "${ARGOCD_SERVICE_TYPE}" != "ClusterIP" ]; then
                              kubectl --kubeconfig=.kube/config \
                                -n "${ARGOCD_NAMESPACE}" \
                                patch svc argocd-server \
                                -p "{\\"spec\\": {\\"type\\": \\"${ARGOCD_SERVICE_TYPE}\\"}}"
                            fi

                            echo "Argo CD manifest applied."
                        '''
                    }
                }
            }
        }

        stage('Wait for Argo CD') {
            steps {
                container('kubectl') {
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh '''
                            set -e

                            mkdir -p .kube
                            cp "${KUBECONFIG_FILE}" .kube/config
                            chmod 600 .kube/config

                            if [ -z "${ARGOCD_NAMESPACE}" ]; then
                              ARGOCD_NAMESPACE="argocd"
                            fi

                            if [ -z "${TIMEOUT_SECONDS}" ]; then
                              TIMEOUT_SECONDS="600"
                            fi

                            echo "Waiting for Argo CD deployments..."

                            kubectl --kubeconfig=.kube/config \
                              -n "${ARGOCD_NAMESPACE}" \
                              wait --for=condition=Available deployment \
                              --all \
                              --timeout="${TIMEOUT_SECONDS}s"

                            echo "Waiting for Argo CD pods..."

                            kubectl --kubeconfig=.kube/config \
                              -n "${ARGOCD_NAMESPACE}" \
                              wait --for=condition=Ready pod \
                              --all \
                              --timeout="${TIMEOUT_SECONDS}s"

                            echo "Argo CD is ready."
                        '''
                    }
                }
            }
        }

        stage('Verify Argo CD') {
            steps {
                container('kubectl') {
                    withCredentials([
                        file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG_FILE')
                    ]) {
                        sh '''
                            set -e

                            mkdir -p .kube
                            cp "${KUBECONFIG_FILE}" .kube/config
                            chmod 600 .kube/config

                            if [ -z "${ARGOCD_NAMESPACE}" ]; then
                              ARGOCD_NAMESPACE="argocd"
                            fi

                            echo "Argo CD pods:"
                            kubectl --kubeconfig=.kube/config \
                              -n "${ARGOCD_NAMESPACE}" \
                              get pods -o wide

                            echo ""
                            echo "Argo CD services:"
                            kubectl --kubeconfig=.kube/config \
                              -n "${ARGOCD_NAMESPACE}" \
                              get svc -o wide

                            echo ""
                            echo "Argo CD initial admin secret:"
                            if kubectl --kubeconfig=.kube/config \
                              -n "${ARGOCD_NAMESPACE}" \
                              get secret argocd-initial-admin-secret >/dev/null 2>&1; then
                              echo "Initial admin secret exists."
                            else
                              echo "Initial admin secret not found."
                            fi

                            echo ""
                            echo "Argo CD installation verification completed."
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Argo CD installation completed successfully.'
        }

        failure {
            echo 'Argo CD installation failed. Check Jenkins console logs.'
        }

        always {
            container('kubectl') {
                sh '''
                    rm -rf .kube || true
                '''
            }
        }
    }
}