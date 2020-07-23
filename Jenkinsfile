pipeline {
  agent any
  stages {
    stage('SCM') {
      steps {
        checkout scm
      }
    }

    stage('Build PHP') {
      parallel {
        stage('Compile') {
          agent {
            docker {
              reuseNode true
              image 'docker/compose'
            }

          }
          steps {
            sh 'printenv'
            sh 'echo $GIT_BRANCH'
            sh 'echo $GIT_COMMIT'
            echo 'Install non-dev composer packages and test a symfony cache clear'
            sh 'docker-compose -f build.yml up --exit-code-from fpm_build --remove-orphans fpm_build'
            echo 'Building the docker images with the current git commit'
            sh 'docker build -f Dockerfile-php-production -t localhost:5000/symfony_project_fpm:$BUILD_TAG .'
          }
        }

      }
    }

    stage('PHP Unit Tests') {
      steps {
        echo 'PHP Unit tests'
      }
    }

    stage('SonarQube') {
      agent any
      steps {
        sh 'echo "placeholder for the code analysis"'
      }
    }

    stage('Push') {
      when {
        branch 'develop'
      }
      steps {
        echo 'Deploying docker images'
        sh 'docker tag localhost:5000/symfony_project_fpm:$BUILD_TAG localhost:5000/symfony_project_fpm:latest'
        sh 'docker push localhost:5000/symfony_project_fpm:$BUILD_TAG'
        sh 'docker push localhost:5000/symfony_project_fpm:latest'
      }
    }

    stage('Deploy to Test Servers') {
      agent {
        docker {
          image 'ahmed24khaled/ansible-management'
          reuseNode true
        }

      }
      when {
        anyOf {
          branch 'develop'
        }

      }
      steps {
        sh 'ansible-playbook -i ./ansible_provisioning/hosts --extra-vars "host=local" ./ansible_provisioning/playbook.yml'
      }
    }

  }
  environment {
    SONARQUBE_URL = 'http://192.168.0.30'
    SONARQUBE_PORT = '9000'
  }
  options {
    skipDefaultCheckout()
  }
}
