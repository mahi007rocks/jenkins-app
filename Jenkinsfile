pipeline {
    environment {
      DOCKER = credentials('docker-hub')
    }
  agent any
  stages { 
// Remote server login
    stage('REMOTE LOGIN') {
      steps {
        sh 'ssh -t jenkins@172.16.9.63:8000 'ssh root@172.16.10.128''  
 //       sh 'ssh root@172.16.10.128'
      }
    }
// Building your Test Images
    stage('BUILD') {
      parallel {
        stage('Express Image') {
          steps {
             
            sh 'docker build -f express-image/Dockerfile \
            -t nodeapp-dev:${BUILD_NUMBER} .'
          }
        }
        stage('Test-Unit Image') {
          steps {
            sh 'docker build -f test-image/Dockerfile \
            -t test-image:${BUILD_NUMBER} .'
          }
        }
      }
      post {
        failure {
            echo 'This build has failed. See logs for details.'
        }
      }
    }
// Performing Software Tests
    stage('TEST') {
      parallel {
        stage('Mocha Tests') {
          steps {
            sh 'docker run --name nodeapp-dev --network="bridge" -d \
            -p 9000:9000 nodeapp-dev:${BUILD_NUMBER}'
            sh 'docker run --name test-image -v $PWD:/JUnit --network="bridge" \
            --link=nodeapp-dev -d -p 9001:9000 \
            test-image:${BUILD_NUMBER}'
          }
        }
        stage('Quality Tests') {
          steps {
            sh 'docker login --username $DOCKER_USR --password $DOCKER_PSW'
            sh 'docker tag nodeapp-dev:${BUILD_NUMBER} $DOCKER_USR/nodeapp-dev:${BUILD_NUMBER}'
            sh 'docker push $DOCKER_USR/nodeapp-dev:${BUILD_NUMBER}'
          }
        }
      }
      post {
        success {
            echo 'Build succeeded.'
        }
        unstable {
            echo 'This build returned an unstable status.'
        }
        failure {
            echo 'This build has failed. See logs for details.'
        }
      }
    }
// Deploying your Software
    stage('DEPLOY') {
          when {
           branch 'master'  //only run these steps on the master branch
          }
            steps {
                    retry(3) {
                        timeout(time:10, unit: 'MINUTES') {
                            sh 'docker tag nodeapp-dev:${BUILD_NUMBER} $DOCKER_USR/nodeapp-prod:${BUILD_NUMBER}'
                            sh 'docker push $DOCKER_USR/nodeapp-prod:${BUILD_NUMBER}'
                            sh 'docker save $DOCKER_USR/nodeapp-prod:${BUILD_NUMBER} | gzip > nodeapp-prod-golden.tar.gz'
                        }
                    }

            }
            post {
                failure {
                    sh 'docker stop nodeapp-dev test-image'
                    sh 'docker system prune -f'
                    deleteDir()
                }
            }
    }
// JUnit reports and artifacts saving
    stage('REPORTS') {
      steps {
        junit 'reports.xml'
        archiveArtifacts(artifacts: 'reports.xml', allowEmptyArchive: true)
        archiveArtifacts(artifacts: 'nodeapp-prod-golden.tar.gz', allowEmptyArchive: true)
      }
    }
// Doing containers clean-up to avoid conflicts in future builds
//    stage('CLEAN-UP') {
//      steps {
//        sh 'docker stop nodeapp-dev test-image'
//        sh 'docker system prune -f'
//        deleteDir()
//      }
//    }
  }
}
