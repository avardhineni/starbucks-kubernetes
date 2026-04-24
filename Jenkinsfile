pipeline {
    agent any

    tools {
        nodejs 'node17'
    }

    environment {
        SONAR_SCANNER = tool 'sonar-scanner'
        DOCKER_IMAGE = "avardhineni7/starbucks-kubernetes"
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_APP_NAME = "starbucks-app"
        K8S_NAMESPACE = "default"
        KUBECTL = "/var/lib/minikube/binaries/v1.35.1/kubectl --kubeconfig=/var/lib/minikube/kubeconfig"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/avardhineni/starbucks-kubernetes.git'
            }
        }

        stage('Test Tools') {
            steps {
                sh 'node -v'
                sh 'npm -v'
                sh 'docker ps'
                sh "docker exec minikube sh -c '${KUBECTL} get nodes'"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build React App') {
            steps {
                sh 'CI=false npm run build'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${SONAR_SCANNER}/bin/sonar-scanner \
                          -Dsonar.projectKey=starbucks-kubernetes \
                          -Dsonar.projectName=starbucks-kubernetes \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Create Kubernetes Manifests') {
            steps {
                sh """
                    mkdir -p k8s-generated

                    cat > k8s-generated/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${K8S_APP_NAME}
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${K8S_APP_NAME}
  template:
    metadata:
      labels:
        app: ${K8S_APP_NAME}
    spec:
      containers:
      - name: ${K8S_APP_NAME}
        image: ${DOCKER_IMAGE}:${IMAGE_TAG}
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
EOF

                    cat > k8s-generated/service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ${K8S_APP_NAME}-svc
  namespace: ${K8S_NAMESPACE}
spec:
  type: NodePort
  selector:
    app: ${K8S_APP_NAME}
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30080
EOF

                    cat > k8s-generated/ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${K8S_APP_NAME}-ingress
  namespace: ${K8S_NAMESPACE}
spec:
  rules:
  - host: starbucks.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${K8S_APP_NAME}-svc
            port:
              number: 3000
EOF
                """
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh """
                    echo "Checking generated files:"
                    ls -la k8s-generated

                    docker exec minikube rm -rf /tmp/starbucks-k8s
                    docker exec minikube mkdir -p /tmp/starbucks-k8s

                    docker cp k8s-generated/deployment.yaml minikube:/tmp/starbucks-k8s/deployment.yaml
                    docker cp k8s-generated/service.yaml minikube:/tmp/starbucks-k8s/service.yaml
                    docker cp k8s-generated/ingress.yaml minikube:/tmp/starbucks-k8s/ingress.yaml

                    echo "Files copied to Minikube:"
                    docker exec minikube sh -c "ls -la /tmp/starbucks-k8s"

                    docker exec minikube sh -c '${KUBECTL} apply -f /tmp/starbucks-k8s/deployment.yaml'
                    docker exec minikube sh -c '${KUBECTL} apply -f /tmp/starbucks-k8s/service.yaml'
                    docker exec minikube sh -c '${KUBECTL} apply -f /tmp/starbucks-k8s/ingress.yaml'

                    docker exec minikube sh -c '${KUBECTL} rollout status deployment/${K8S_APP_NAME} -n ${K8S_NAMESPACE}'
                    docker exec minikube sh -c '${KUBECTL} get pods,svc,ingress -n ${K8S_NAMESPACE}'
                """
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
    }
}