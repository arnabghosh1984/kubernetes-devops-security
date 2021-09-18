@Library('slack') _


pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "arnabghosh1984/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://devsecops.eastus.cloudapp.azure.com"
    applicationURI = "compare/99"
  }

  stages {

    stage('Build Artifact - Maven') {
      steps {
        sh "mvn clean package -DskipTests=true"
        archive 'target/*.jar'
      }
    }

    stage('Unit Tests - JUnit and JaCoCo') {
      steps {
        sh "mvn test"
      }
     
    }

//     stage('Mutation Tests - PIT') {
//      steps {
//        sh "mvn org.pitest:pitest-maven:mutationCoverage"
//      }
//      post {
//        always {
//          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
//        }
//      }
//    } 

stage('SonarQube - SAST') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh "mvn sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.host.url=http://devsecops.eastus.cloudapp.azure.com:9000 -Dsonar.login=a6e256be6dcd99c1841c150738a27bc0c204978b"
        }
        timeout(time: 2, unit: 'MINUTES') {
          script {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }

stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
          "Dependency Scan": {
            sh "mvn dependency-check:check"
          },
          "Trivy Scan": {
            sh "bash trivy-docker-image-scan.sh"
          },
          "OPA Conftest": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
          }
        )
      }
    }


    stage('Docker Build and Push') {
      steps {
        withDockerRegistry([credentialsId: "docker-hub", url: "https://index.docker.io/v1/"]) {
          sh 'printenv'
          sh 'sudo docker build -t arnabghosh1984/numeric-app:""$GIT_COMMIT"" .'
          sh ' docker push arnabghosh1984/numeric-app:""$GIT_COMMIT""'
        }
      }
    }


   stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
          "OPA Scan": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "bash kubesec-scan.sh"
          },
          "Trivy Scan": {
            sh "bash trivy-k8s-scan.sh"
          }
        )
      }
    }


 //   stage('Kubernetes Deployment - DEV') {
 //     steps {
 //       withKubeConfig([credentialsId: 'kubeconfig']) {
 //         sh "sed -i 's#replace#arnabghosh1984/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
 //         sh "kubectl apply -f k8s_deployment_service.yaml"
 //       }
 //     }
 //   }

 // }

 stage('K8S Deployment - DEV') {
      steps {
        parallel(
          "Deployment": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment.sh"
            }
          },
          "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment-rollout-status.sh"
            }
          }
        )
      }
    }



  stage('Integration Tests - DEV') {
      steps {
        script {
          try {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash integration-test.sh"
            }
          } catch (e) {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "kubectl -n default rollout undo deploy ${deploymentName}"
            }
            throw e
          }
        }
      }
    }
    
    stage('OWASP ZAP - DAST') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh 'bash zap.sh'
        }
      }
    }

    stage('Prompte to PROD?') {
      steps {
          timeout(time: 2, unit: 'DAYS') {
          input 'Do you want to Approve the Deployment to Production Environment/Namespace?'
    }
  }
}

stages {
    stage('Testing Slack') {
      steps {
        sh 'exit 0'
      }
    }

  }



  }
    
    
  
  

    post {
    always {
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML Report', reportTitles: 'OWASP ZAP HTML Report'])
    
// Use sendNotifications.groovy from shared library and provide current build result as parameter    
      sendNotification currentBuild.result

    }

    // success {

    // }

    // failure {

    // }
  }

}
