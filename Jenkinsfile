pipeline {
  
  agent {
    node {
      label ''
    }
  }
  
  options {
    timeout(time: 20, unit: 'MINUTES')
  }
  
  stages {

    stage('preamble') {
      steps {
        script {
          String lockName = "${JOB_NAME}-${BUILD_NUMBER}" as String
          openshift.setLockName(lockName)

          openshift.withCluster() {
            openshift.withProject() {
              echo "Using project: ${openshift.project()}"
             
              sh "echo -n `date -u +%Y%m%d%H%M%S%N` > build-timestamp.txt"
              VERSION = readFile 'build-timestamp.txt'
              
              echo "Building version: ${VERSION}"
              echo sh(returnStdout: true, script: 'env')
            }
          }
        }
      }
    }
    
    stage('clone repo') {
      steps {
        script {
          git url: 'https://github.com/mberndtgen/time-service.git'
        }
      }
    }

    stage('build') {
      steps {
        script {
          sh "./gradlew clean bootJar"
        }
      }
    }

    stage('unittest') {
      steps {
        script {
          sh "./gradlew test"
        }
      }
    }

    stage('update os artefacts') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject() {
              def models = openshift.process("openshift//ergo-openjdk18-binary-s2i", "--param-file=openshift/template-parameters.txt")
              def created = openshift.apply(models)
              //openshift.apply("-l=app=time-service", "--dry-run=true", "-f ./openshift")
              //openshift.apply("-l=app=time-service", "--dry-run=false", "-f ./openshift")
            }
          }
        }
      }
    }

    stage('os build & deployment') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject() {
              openshift.selector("bc", "time-service").startBuild("--from-dir=build/libs", "--wait=true")
            }
          }
        }
      }
    }
    
    stage('epilogue') {
      parallel {
        stage('tag image') {
          steps {
            script {
              openshift.withCluster() {
                openshift.withProject() {
                  openshift.tag("${openshift.project()}/time-service:latest", "${openshift.project()}/time-service:${VERSION}")
                }
              }
            }
          }
        }

        stage('tag repo') {
          steps {
            script {
              withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github', usernameVariable: 'githubUser', passwordVariable: 'githubPwd']]) {

                def fullGitUrl = ''
                //def fullGitUrl = "https://${githubUser}:${githubPwd}@github.com/torstenatgithub/time-service.git"

                lock(inversePrecedence: true, resource: 'time-service-regex') {
                  def gitUrl = sh returnStdout: true, script: 'git config remote.origin.url'
                  def gitUrlMatcher = gitUrl =~ '(.+://)(.+)'

                  if (gitUrlMatcher) {
                    def gitProtocol = gitUrlMatcher[0][1]
                    def gitHostAndPath = gitUrlMatcher[0][2]
                    def gitUrlMatcher2 = gitHostAndPath =~ '@(.+)'
                    if (gitUrlMatcher2) {
                      gitHostAndPath = gitUrlMatcher2[0][1]
                    }
                    fullGitUrl = "${gitProtocol}${githubUser}:${githubPwd}@${gitHostAndPath}"
                  } else {
                    error ("Unable to parse Git url ${gitUrl}")
                  }
                }

                sh("git config user.name \"CI/CD pipeline\"")
                sh("git config user.email \"cicd@no.reply\"")
                sh("git add build-timestamp.txt")
                sh("git commit -am \"Version ${VERSION}\"")
                sh("git tag -am \"Tag ${VERSION}\" ${VERSION}")
                sh("git push ${fullGitUrl} --tags --quiet")
              }
            }
          }
        }

        stage('verify deployment') {
          steps {
            script {
              openshift.withCluster() {
                openshift.withProject() {
                  def latestDeploymentVersion = openshift.selector("dc", "time-service").object().status.latestVersion
                  def rc = openshift.selector("rc", "time-service-${latestDeploymentVersion}")
                  rc.untilEach(1) {
                    def rcMap = it.object()
                    return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
