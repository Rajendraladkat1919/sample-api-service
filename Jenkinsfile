pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    APP_NAME = "sample-api-service"
    IMAGE_REGISTRY = "rmkanda"
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh './mvnw install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets scanner') {
          steps {
            container('trufflehog') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh "trufflehog -x secrets-exclude.txt ${GIT_URL}"
              }
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh './mvnw package -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
        }
      }
    }

    stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh '''#!/bin/bash --login
                      /bin/bash --login
                      rvm use default
                      gem install license_finder
                      license_finder
                    '''
              }
            }
          }
        }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh './mvnw test'
            }
          }
        }
        stage('Dependency check ') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: false
              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
            }
          }
        }
      }
    }
    stage('Spot Bugs - Security') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw compile spotbugs:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/spotbugsXml.xml', fingerprint: true, onlyIfSuccessful: false
              recordIssues enabledForFailure: true, tool: spotBugs()
            }
          }
        }
    stage('Package') {
      steps {
        container('docker-tools') {
          sh "docker build . -t ${APP_NAME}"
        }
      }
    }
    stage('Artefact Analysis') {
      parallel {
        stage('Contianer Scan') {
          steps {
            container('docker-tools') {
              sh "grype ${APP_NAME}"
            }
          }
        }
        stage('Container Audit') {
          steps {
            container('docker-tools') {
              sh "dockle ${APP_NAME}"
            }
          }
        }
      }
    }

    stage('Publish') {
      steps {
        container('docker-tools') {
          echo "Publishing docker image"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Kubesec') {
      steps {
        container('docker-tools') {
          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE')
          echo "Scanning the kube yaml files"
          sh "kubesec scan k8s.yaml"
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        container('docker-tools') {
          echo "Deploying the app"
          // sh "kubectl apply -f k8s.yaml"
        }
      }
    }

       stage('DSAT') {
          steps {
            container('docker-tools') {
              sh "docker pull owasp/zap2docker-weekly"
              sh "docker run -t owasp/zap2docker-weekly zap-api-scan.py -t http://172.17.0.4:30001/v3/api-docs -f openapi"
            }
          }
        }

    stage('Promote to Prod') {
      steps {
        container('docker-tools') {
          echo "Promote to Prod"
        }
      }
    }
  }
}