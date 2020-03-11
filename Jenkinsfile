#!groovy
def git_coomit_hash = ""
def git_branch = ""
def deploy_branch = "origin/pipeline-practice"
def deploy_project = "meeting-development"
def build_config_name = "django-psql-persistent"
def app_image = "docker-registry.default.svc:5000/${deploy_project}/${build_config_name}"

pipeline {
  // pipelineを実行するagentの設定。yamlファイルで設定を渡せる
  // 可能な限りJenkinsfileにagentの設定をもたせたほうが自動化とGit管理が進むためおすすめ。
  agent {
    kubernetes {
      cloud 'openshift'
      yamlFile 'openshift/templates/jenkins-slave-pod.yaml'
    }
  }

  stages {
    stage('Checkout Source') {
      steps {
        checkout scm
      }
    }

    stage('Setup') {
      steps {
        sh 'pip install -r requirements.txt --user'
        sh './manage.py migrate'
      }
    }

    stage('application test') {
      parallel {
        stage('Code analysis') {
          steps {
            echo "Exec static analysis"
            //sh ''
          }
        }

        stage('unit test') {
          steps {
            echo "Exec unit test"
            sh './manage.py test'
          }
        }
      }
    }

    stage('Build and Tag OpenShift Image') {
      when {
        expression {
          return env.GIT_BRANCH == "${deploy_branch}" || params.FORCE_FULL_BUILD
        }
      }

      steps {
        echo "Building OpenShift container image"
        script {
          openshift.withCluster() {
            openshift.withProject("${deploy_project}") {
              // update build config
              //sh "oc process -f openshift/templates/application-build.yaml | oc apply -n ${deploy_project} -f -"
              openshift.apply(openshift.process('-f', 'openshift/templates/application-build.yaml'))

              openshift.selector("bc", "${build_config_name}").startBuild("--wait=true")
              openshift.tag("${build_config_name}:latest", "${build_config_name}:${env.GIT_COMMIT}")
            }
          }
        }
      }

    }

    stage('deploy') {
      when {
        expression {
          return env.GIT_BRANCH == "${deploy_branch}" || params.FORCE_FULL_BUILD
        }
      }

      steps {
        echo "deploy"
        script {
          openshift.withCluster() {
            openshift.withProject("${deploy_project}") {
              //sh "oc process -f openshift/templates/application-deploy.yaml -p APP_IMAGE=${app_image} APP_IMAGE_TAG=${env.GIT_COMMIT} | oc apply -n ${deploy_project} -f -"
              openshift.apply(openshift.process('-f', 'openshift/templates/application-deploy.yaml', '-p', "APP_IMAGE=${app_image}", "-p", "APP_IMAGE_TAG=${env.GIT_COMMIT}"))
            }
          }
        }
      }
    }
  }
}
