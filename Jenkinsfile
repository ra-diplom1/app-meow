pipeline {
  agent none
  stages {
    //Build container image
    stage('Build') {
      agent {
        kubernetes {
          inheritFrom 'jenkinsrun'
          defaultContainer 'dind'
          yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:18.05-dind
    securityContext:
      privileged: true
    volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
  volumes:
    - name: dind-storage
      emptyDir: {}
"""
        }
      }
      steps {
        sh "printenv"
        container('dind') {
          script {
            docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-auth') {
              def pathTag = "raleonid/app-meow:${JOB_BASE_NAME}"
              if(env.TAG_NAME == null || env.TAG_NAME.length() == 0) {
                pathTag = "${pathTag}-${BUILD_ID}"
              }
              //build the image
              def customImage = docker.build(pathTag)
              //upload it to the registry
              customImage.push()
              println "test7"
            }
          }
        }
      }
    }
    stage('Deploy') {
      agent {

        kubernetes {
          inheritFrom 'helm-pod'
          yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: helm
    image: wardviaene/helm-s3
    command:
    - cat
    tty: true
"""
         }
      }
      when {
          // Only say hello if a "greeting" is requested
          expression { env.TAG_NAME != null && env.TAG_NAME.length() > 0 }
      }
      steps {
        container('helm') {
          withKubeConfig([credentialsId: 'token-k8s-sa', namespace: "stage"]) {
            sh "helm template deploy -n stage --set image.tag=${TAG_NAME} > deployment.yaml"
            sh "readlink -f deployment.yaml"
            sh "helm upgrade --install app-meow app --create-namespace -n stage"
          }
          //sh "helm upgrade --install app-meow deploy --create-namespace -n stage"
        }
      }
    }
    stage('Deploy2') {
      agent {

        kubernetes {
          inheritFrom 'jenkins-jenkins-agent'
        }
      }
      when {
          // Only say hello if a "greeting" is requested
          expression { true || env.TAG_NAME != null && env.TAG_NAME.length() > 0 }
      }
      steps {
        // script {
        //       kubernetes.image().withName("raleonid/app-meow:${JOB_BASE_NAME}").build().fromPath(".")
        // }
        // kubernetesDeploy(configs: "deployment.yaml")
        // withKubeConfig([namespace: "stage"]) {
        //   sh 'kubectl apply -f deployment.yaml'
        // }
        // sh "readlink -f deployment.yaml"
        withKubeConfig([credentialsId: 'token-k8s-sa', namespace: "stage"]) {
            sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'
            sh 'chmod u+x ./kubectl'
            sh './kubectl get pods -n stage'
        }
      }
    }
  }
}