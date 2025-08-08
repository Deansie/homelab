```groovy
pipeline {
    agent any
    environment {
        DOCKER_REGISTRY = 'example-ip:5000'
        APP_NAME = 'example-app'
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}-frontend:${env.BUILD_ID}"
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}-backend:${env.BUILD_ID}"
        FRONTEND_IMAGE_LATEST = "${DOCKER_REGISTRY}/${APP_NAME}-frontend:latest"
        BACKEND_IMAGE_LATEST = "${DOCKER_REGISTRY}/${APP_NAME}-backend:latest"
        GIT_REPO_URL = 'example-repo'
        GIT_BRANCH = 'main'
        SSH_CREDENTIALS_ID = 'example-credential'
        CONTROL_PLANE_IP = 'example-ip'
        SSH_USER = 'jenkins'
        K8S_MANIFEST_PATH = '/root/kubernetes/manifests/applications'
        K8S_MANIFEST_DIR = 'deansie-wordle'
        K8S_NAMESPACE = 'default'
        FRONTEND_PORT = 'example-port'
        BACKEND_PORT = 'example-port'
        SERVICE_PORT = '80'
        REPLICAS = '4'
        CPU_REQUEST = '100m'
        MEMORY_REQUEST = '256Mi'
        CPU_LIMIT = '500m'
        MEMORY_LIMIT = '512Mi'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${GIT_BRANCH}"]],
                    userRemoteConfigs: [[url: "${GIT_REPO_URL}"]]
                ])
            }
        }
        stage('Verify Dockerfiles') {
            steps {
                script {
                    if (!fileExists('frontend/Dockerfile') || !fileExists('backend/Dockerfile')) {
                        error 'Dockerfile(s) not found in repository'
                    }
                    echo 'Dockerfiles found, proceeding with build'
                }
            }
        }
        stage('Build Docker Images') {
            steps {
                sh '''
                    #!/bin/bash
                    cd frontend
                    docker build -t ${FRONTEND_IMAGE} --no-cache .
                    cd ..
                    cd backend
                    docker build -t ${BACKEND_IMAGE} --no-cache .
                    cd ..
                '''
            }
        }
        stage('Push Docker Images') {
            steps {
                sh '''
                    docker push ${FRONTEND_IMAGE}
                    docker push ${BACKEND_IMAGE}
                    docker tag ${FRONTEND_IMAGE} ${FRONTEND_IMAGE_LATEST}
                    docker tag ${BACKEND_IMAGE} ${BACKEND_IMAGE_LATEST}
                    docker push ${FRONTEND_IMAGE_LATEST}
                    docker push ${BACKEND_IMAGE_LATEST}
                '''
            }
        }
        stage('Generate Kubernetes Manifests') {
            steps {
                script {
                    writeFile file: "${K8S_MANIFEST_DIR}/deployments/frontend-deployment.yaml", text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-frontend
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: ${REPLICAS}
  selector:
    matchLabels:
      app: ${APP_NAME}-frontend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-frontend
    spec:
      containers:
      - name: ${APP_NAME}-frontend
        image: ${FRONTEND_IMAGE}
        imagePullPolicy: Always
        ports:
        - containerPort: ${FRONTEND_PORT}
        resources:
          requests:
            cpu: "${CPU_REQUEST}"
            memory: "${MEMORY_REQUEST}"
          limits:
            cpu: "${CPU_LIMIT}"
            memory: "${MEMORY_LIMIT}"
        readinessProbe:
          httpGet:
            path: /
            port: ${FRONTEND_PORT}
          initialDelaySeconds: 5
          periodSeconds: 10
"""
                    writeFile file: "${K8S_MANIFEST_DIR}/services/frontend-service.yaml", text: """
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-frontend-service
  namespace: ${K8S_NAMESPACE}
spec:
  selector:
    app: ${APP_NAME}-frontend
  ports:
    - protocol: TCP
      port: ${SERVICE_PORT}
      targetPort: ${FRONTEND_PORT}
  type: ClusterIP
"""
                    writeFile file: "${K8S_MANIFEST_DIR}/deployments/backend-deployment.yaml", text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-backend
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: ${REPLICAS}
  selector:
    matchLabels:
      app: ${APP_NAME}-backend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-backend
    spec:
      containers:
      - name: ${APP_NAME}-backend
        image: ${BACKEND_IMAGE}
        imagePullPolicy: Always
        ports:
        - containerPort: ${BACKEND_PORT}
        env:
        - name: PORT
          value: "${BACKEND_PORT}"
        resources:
          requests:
            cpu: "${CPU_REQUEST}"
            memory: "${MEMORY_REQUEST}"
          limits:
            cpu: "${CPU_LIMIT}"
            memory: "${MEMORY_LIMIT}"
        readinessProbe:
          tcpSocket:
            port: ${BACKEND_PORT}
          initialDelaySeconds: 5
          periodSeconds: 10
"""
                    writeFile file: "${K8S_MANIFEST_DIR}/services/backend-service.yaml", text: """
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-backend-service
  namespace: ${K8S_NAMESPACE}
spec:
  selector:
    app: ${APP_NAME}-backend
  ports:
    - protocol: TCP
      port: ${SERVICE_PORT}
      targetPort: ${BACKEND_PORT}
  type: ClusterIP
"""
                    writeFile file: "${K8S_MANIFEST_DIR}/ingresses/ingress.yaml", text: """
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}-ingress
  namespace: ${K8S_NAMESPACE}
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: deansie-wordle.ekedala-services.se
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-frontend-service
            port:
              number: ${SERVICE_PORT}
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-backend-service
            port:
              number: ${SERVICE_PORT}
"""
                }
            }
        }
        stage('Deploy to Kubernetes via SSH') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY')]) {
                    sh """
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${CONTROL_PLANE_IP} "
                            mkdir -p ${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/{deployments,services,ingresses}
                        "
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no ${K8S_MANIFEST_DIR}/deployments/frontend-deployment.yaml ${SSH_USER}@${CONTROL_PLANE_IP}:${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/deployments/
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no ${K8S_MANIFEST_DIR}/services/frontend-service.yaml ${SSH_USER}@${CONTROL_PLANE_IP}:${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/services/
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no ${K8S_MANIFEST_DIR}/deployments/backend-deployment.yaml ${SSH_USER}@${CONTROL_PLANE_IP}:${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/deployments/
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no ${K8S_MANIFEST_DIR}/services/backend-service.yaml ${SSH_USER}@${CONTROL_PLANE_IP}:${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/services/
                        scp -i ${SSH_KEY} -o StrictHostKeyChecking=no ${K8S_MANIFEST_DIR}/ingresses/ingress.yaml ${SSH_USER}@${CONTROL_PLANE_IP}:${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/ingresses/
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${CONTROL_PLANE_IP} "
                            kubectl apply -f ${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/services/
                            kubectl apply -f ${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/deployments/
                            kubectl apply -f ${K8S_MANIFEST_PATH}/${K8S_MANIFEST_DIR}/ingresses/
                            kubectl rollout status deployment/${APP_NAME}-frontend -n ${K8S_NAMESPACE}
                            kubectl rollout status deployment/${APP_NAME}-backend -n ${K8S_NAMESPACE}
                        "
                    """
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
```
