pipeline {
  agent any
  environment {
  // Specify your environment variables.
  APP_VERSION = '1'
  // This can be nexus3 or nexus2
  NEXUS_VERSION = "nexus3"
  // This can be http or https
  NEXUS_PROTOCOL = "http"
  // Where your Nexus is running. In my case:
  NEXUS_URL = "192.168.0.30:8081"
  // Repository where we will upload the artifact
  NEXUS_REPOSITORY = "maven-snapshots"
  // Jenkins credential id to authenticate to Nexus OSS
  NEXUS_CREDENTIAL_ID = "nexus-credentials"
  /* 
    Windows: set the ip address of docker host. In my case 192.168.99.100.
    to obtains this address : $ docker-machine ip
    Linux: set localhost to SONARQUBE_URL
  */
  SONARQUBE_URL = "http://192.168.0.30"
  SONARQUBE_PORT = "9000"
    }
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
            sh 'docker build -f Dockerfile-php-production -t 192.168.0.30:8081/symfony_project_fpm:$BUILD_TAG .'
          }
        }

      }
    }

    stage('Test') {
      steps {
        echo 'PHP Unit tests'
//        sh 'docker-compose -f test.yml up -d --build --remove-orphans'
//        sh 'sleep 5'
//       sh 'docker-compose -f test.yml exec -T fpm_test bash build/php_unit.sh'
      }
    }


    stage('SonarQube') {
      agent {
        docker {
          image 'maven:3.6.0-jdk-8-alpine'
          args '-v /root/.m2/repository:/root/.m2/repository'
          reuseNode true
        }

      }
      steps {
        sh " mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL:$SONARQUBE_PORT"
      }
    }

    stage('Push') {
    when {
        branch 'develop'
    }
    steps {
    echo 'Deploying docker images'
    sh 'docker tag 192.168.0.30:8081/symfony_project_fpm:$BUILD_TAG 192.168.0.30:8081/symfony_project_fpm:$APP_VERSION'
    sh 'docker tag 192.168.0.30:8081/symfony_project_fpm:$BUILD_TAG 192.168.0.30:8081/symfony_project_fpm:latest'
    sh 'docker push 192.168.0.30:8081/symfony_project_fpm:$APP_VERSION'
    sh 'docker push 192.168.0.30:8081/symfony_project_fpm:latest'
    sh 'docker tag 192.168.0.30:8081/symfony_project_nginx:$BUILD_TAG 192.168.0.30:8081/symfony_project_nginx:$APP_VERSION'
    sh 'docker tag 192.168.0.30:8081/symfony_project_nginx:$BUILD_TAG 192.168.0.30:8081/symfony_project_nginx:latest'
    sh 'docker push 192.168.0.30:8081/symfony_project_nginx:$APP_VERSION'
    sh 'docker push 192.168.0.30:8081/symfony_project_nginx:latest'
    sh 'docker tag 192.168.0.30:8081/symfony_project_db:$BUILD_TAG 192.168.0.30:8081/symfony_project_db:$APP_VERSION'
    sh 'docker tag 192.168.0.30:8081/symfony_project_db:$BUILD_TAG 192.168.0.30:8081/symfony_project_db:latest'
    sh 'docker push 192.168.0.30:8081/symfony_project_db:$APP_VERSION'
    sh 'docker push 192.168.0.30:8081/symfony_project_db:latest'
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
      branch 'master'
      branch 'develop'
    }

  }
  steps {
  sh 'ansible-playbook -v -i ./ansible_provisioning/hosts --extra-vars "host=staging" ./ansible_provisioning/playbook.yml'
    }

  }
}

}
environment {
NEXUS_VERSION = 'nexus3'
NEXUS_PROTOCOL = 'http'
NEXUS_URL = 'http://192.168.0.30:8081'
NEXUS_REPOSITORY = 'maven-snapshots'
NEXUS_CREDENTIAL_ID = 'nexus-credentials'
SONARQUBE_URL = 'http://http://192.168.0.30/'
SONARQUBE_PORT = '9000'
}
options {
skipDefaultCheckout()
}
}
