pipeline {
    agent any
    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-2', description: 'AWS region where the EKS cluster is running')
        string(name: 'EKS_CLUSTER_NAME', defaultValue: 'eks-store', description: 'EKS cluster name')
        string(name: 'K8S_NAMESPACE', defaultValue: 'default', description: 'Kubernetes namespace used for deploy')
    }
    environment {
        SERVICE = 'gateway'
        NAME = "iquenavarro/${env.SERVICE}"
        AWS_CREDENTIALS_ID = 'aws-eks-credentials'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${env.AWS_CREDENTIALS_ID}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY')])
                {
                    sh 'aws eks update-kubeconfig --region "$AWS_REGION" --name "$EKS_CLUSTER_NAME"'
                    sh 'kubectl apply -n "$K8S_NAMESPACE" -f k8s/'
                    sh 'kubectl set image -n "$K8S_NAMESPACE" deployment/"$SERVICE" "$SERVICE"="$NAME":"$BUILD_ID"'
                    sh 'kubectl rollout status -n "$K8S_NAMESPACE" deployment/"$SERVICE"'
                }
            }
        }
    }
}
