def  appName = 'sample-app'
def  feSvcName = "${appName}-frontend"
def  ACRNAME = 'tugboatlabs'
def  imageTag = "${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

pipeline {
  agent {
    kubernetes {
      label 'sample-app'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
labels:
  component: ci
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: jenkins
  containers:
  - name: golang
    image: golang:1.10
    command:
    - cat
    tty: true
  - name: azurecli
    image: microsoft/azure-cli:latest
    command:
    - cat
    tty: true
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true
"""
}
  }
  stages {

    stage('gitpull') {
      steps {
        container('jnlp') {
          sh """
            git clone https://github.com/dstrebel/k8-jenkins
          """
          }
        }
      }
    stage('Test') {
      steps {
        container('golang') {
          sh """
            ln -s /home/jenkins/workspace/test-k8-deploy/k8-jenkins /go/src/k8-jenkins
            cd /go/src/k8-jenkins/
            go test
          """
        }
      }
    }
    stage('Build Container and Push to Registry') {
      steps {
        container('azurecli') {
        withCredentials([azureServicePrincipal('azure-cli')]) {
            sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID'
            sh 'az account set -s $AZURE_SUBSCRIPTION_ID'
            sh("az acr build -t ${imageTag} -r ${ACRNAME} /home/jenkins/workspace/test-k8-deploy/k8-jenkins")
        }
      }
    }
    }
    stage('Deploy Canary') {
      // Canary branch
      when { branch 'canary' }
      steps {
        container('kubectl') {
          // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#tugboatlabs.azurecr.io/${appName}:1.0.0#${imageTag}#' /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/canary/*.yaml")
          sh("kubectl --namespace=production apply -f /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/services/")
          sh("kubectl --namespace=production apply -f /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/canary/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        } 
      }
    }
    stage('Deploy Production') {
      // Production branch
      steps{
        container('kubectl') {
        // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#tugboatlabs.azurecr.io/${appName}:1.0.0#${imageTag}#' /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/production/*.yaml")
          sh("kubectl --namespace=production apply -f /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/services/")
          sh("kubectl --namespace=production apply -f /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/production/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        }
      }
    }
    stage('Deploy Dev') {
      // Developer Branches
      when { 
        not { branch 'master' } 
        not { branch 'canary' }
      } 
      steps {
        container('kubectl') {
          // Create namespace if it doesn't exist
          sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
          // Don't use public load balancing for development branches
          sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
          sh("sed -i.bak 's#tugboatlabs.azurecr.io/${appName}:1.0.0#${imageTag}#' /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/dev/*.yaml")
          sh("kubectl --namespace=${env.BRANCH_NAME} apply -f /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/services/")
          sh("kubectl --namespace=${env.BRANCH_NAME} apply -f /home/jenkins/workspace/test-k8-deploy/k8-jenkins/k8s/dev/")
          echo 'To access your environment run `kubectl proxy`'
          echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
        }
      }
    }
  }
}