pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.8-openjdk-17
                    command:
                    - cat
                    tty: true
                  - name: docker
                    image: docker:20.10.7-dind
                    securityContext:
                      privileged: true
                    volumeMounts:
                    - name: docker-graph-storage
                      mountPath: /var/lib/docker
                  - name: kubectl
                    image: bitnami/kubectl:latest
                    command:
                    - /bin/sh
                    - -c
                    - cat
                    tty: true
                  volumes:
                  - name: docker-graph-storage
                    emptyDir: {}        
            '''
        }
    }
    environment {
        DOCKER_CREDENTIALS_ID = '68d22450-bfc7-40d0-ae3f-153ccc8f9e4f'
        DOCKER_IMAGE = "linuxmanl/petclinic:${env.BUILD_ID}"
    }
    stages {
        stage('Checkout') {
            steps {
                container('maven') {
                    sh 'git clone https://github.com/spring-projects/spring-petclinic.git'
                }
            }
        }
        
        stage('Build with Maven Wrapper') {
            steps {
                container('maven') {
                    dir('spring-petclinic') {
                        sh '''
                        chmod +x mvnw
                        ./mvnw package
                        '''
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    dir('spring-petclinic') {
                        script {
                            sh '''
                            cat <<EOT > Dockerfile
FROM openjdk:17-jdk-slim
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
EOT
                            '''
                            sh "docker build -t ${DOCKER_IMAGE} ."
                        }
                    }
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([string(credentialsId: '68d22450-bfc7-40d0-ae3f-153ccc8f9e4f', variable: 'DOCKER_ACCESS_TOKEN')]) {
                        sh """
                        echo \$DOCKER_ACCESS_TOKEN | docker login -u linuxmanl --password-stdin
                        docker push ${DOCKER_IMAGE}
                        """
                    }
                }
            }
        }   
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'd392ab2c-0372-46de-b19a-6a6a5f40928f', serverUrl: 'https://kubernetes.default.svc']) {
                        script {
                            sh '''
                            kubectl version --client
                            kubectl get nodes
                            '''
                            sh """
                            cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-petclinic
  template:
    metadata:
      labels:
        app: spring-petclinic
    spec:
      containers:
      - name: spring-petclinic
        image: ${DOCKER_IMAGE}
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: spring-petclinic
spec:
  selector:
    app: spring-petclinic
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
                            """
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl get deployments'
                            sh 'kubectl get services'
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            container('maven') {
                cleanWs()
            }
        }
    }
}
