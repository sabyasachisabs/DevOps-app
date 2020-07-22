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
            sh 'docker build -f Dockerfile-php-production -t 192.168.0.30:5000/symfony_project_fpm:$BUILD_TAG .'
          }
        }

      }
    }

    stage('Test') {
      steps {
        echo 'PHP Unit tests'
      }
    }

    stage('Push') {
      when {
        branch 'develop'
      }
      steps {
        sh 'echo "{ "insecure-registries":["192.168.0.30:5000"]} {}" > /etc/docker/daemon.json'
        sh 'docker tag 192.168.0.30:5000/symfony_project_fpm:$BUILD_TAG 192.168.0.30:5000/symfony_project_fpm:$APP_VERSION'
        sh 'docker tag 192.168.0.30:5000/symfony_project_fpm:$BUILD_TAG 192.168.0.30:5000/symfony_project_fpm:latest'
        sh 'docker push 192.168.0.30:5000/symfony_project_fpm:$APP_VERSION'
        sh 'docker push 192.168.0.30:5000/symfony_project_fpm:latest'
        sh 'docker tag 192.168.0.30:5000/symfony_project_nginx:$BUILD_TAG 192.168.0.30:5000/symfony_project_nginx:$APP_VERSION'
        sh 'docker tag 192.168.0.30:5000/symfony_project_nginx:$BUILD_TAG 192.168.0.30:5000/symfony_project_nginx:latest'
        sh 'docker push 192.168.0.30:5000/symfony_project_nginx:$APP_VERSION'
        sh 'docker push 192.168.0.30:5000/symfony_project_nginx:latest'
        sh 'docker tag 192.168.0.30:5000/symfony_project_db:$BUILD_TAG 192.168.0.30:5000/symfony_project_db:$APP_VERSION'
        sh 'docker tag 192.168.0.30:5000/symfony_project_db:$BUILD_TAG 192.168.0.30:5000/symfony_project_db:latest'
        sh 'docker push 192.168.0.30:5000/symfony_project_db:$APP_VERSION'
        sh 'docker push 192.168.0.30:5000/symfony_project_db:latest'
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
      post {
        always {
          recordIssues(aggregatingResults: true, tools: [javaDoc(), checkStyle(pattern: '**/target/checkstyle-result.xml'), findBugs(pattern: '**/target/findbugsXml.xml', useRankAsPriority: true), pmdParser(pattern: '**/target/pmd.xml')])
        }

      }
      steps {
        sh " mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL:$SONARQUBE_PORT"
      }
    }

    stage('Deploy Artifact To Nexus') {
      when {
        anyOf {
          branch 'master'
          branch 'develop'
        }

      }
      steps {
        script {
          unstash 'pom'
          unstash 'artifact'
          // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
          pom = readMavenPom file: "pom.xml";
          // Find built artifact under target folder
          filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
          // P} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
          // Extract the path from the File found
          artifactPath = filesByGlob[0].path;
          // Assign to a boolean response verifying If the artifact name exists
          artifactExists = fileExists artifactPath;
          if (artifactExists) {
            nexusArtifactUploader(
              nexusVersion: NEXUS_VERSION,
              protocol: NEXUS_PROTOCOL,
              nexusUrl: NEXUS_URL,
              groupId: pom.groupId,
              version: pom.version,
              repository: NEXUS_REPOSITORY,
              credentialsId: NEXUS_CREDENTIAL_ID,
              artifacts: [
                // Artifact generated such as .jar, .ear and .war files.
                [artifactId: pom.artifactId,
                classifier: '',
                file: artifactPath,
                type: pom.packaging
              ],
              // Lets upload the pom.xml file for additional information for Transitive dependencies
              [artifactId: pom.artifactId,
              classifier: '',
              file: "pom.xml",
              type: "pom"
            ]
          ]
        )
      } else {
        error "*** File: ${artifactPath}, could not be found";
      }
    }

  }
}

stage('Deploy to Staging Servers') {
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
    script {
      pom = readMavenPom file: "pom.xml"
      repoPath = "${pom.groupId}".replace(".", "/") + "/${pom.artifactId}"
      version = pom.version
      artifactId = pom.artifactId
      withEnv(["ANSIBLE_HOST_KEY_CHECKING=False", "APP_NAME=${artifactId}", "repoPath=${repoPath}", "version=${version}"]) {
        sh '''

curl --silent "http://$NEXUS_URL/repository/maven-snapshots/${repoPath}/${version}/maven-metadata.xml" > tmp &&
egrep '<value>+([0-9\\-\\.]*)' tmp > tmp2 &&
tail -n 1 tmp2 > tmp3 &&
tr -d "</value>[:space:]" < tmp3 > tmp4 &&
REPO_VERSION=$(cat tmp4) &&

export APP_SRC_URL="http://${NEXUS_URL}/repository/maven-snapshots/${repoPath}/${version}/${APP_NAME}-${REPO_VERSION}.war" &&
ansible-playbook -v -i ./ansible_provisioning/hosts --extra-vars "host=staging" ./ansible_provisioning/playbook.yml

'''
      }
    }

  }
}

stage('Deploy to Production Servers') {
  agent {
    docker {
      image 'ahmed24khaled/ansible-management'
      reuseNode true
    }

  }
  when {
    branch 'master'
  }
  steps {
    script {
      pom = readMavenPom file: "pom.xml"
      repoPath = "${pom.groupId}".replace(".", "/") + "/${pom.artifactId}"
      version = pom.version
      artifactId = pom.artifactId
      withEnv(["ANSIBLE_HOST_KEY_CHECKING=False", "APP_NAME=${artifactId}", "repoPath=${repoPath}", "version=${version}"]) {
        sh '''

curl --silent "$NEXUS_URL/repository/maven-snapshots/${repoPath}/${version}/maven-metadata.xml" > tmp &&
egrep '<value>+([0-9\\-\\.]*)' tmp > tmp2 &&
tail -n 1 tmp2 > tmp3 &&
tr -d "</value>[:space:]" < tmp3 > tmp4 &&
REPO_VERSION=$(cat tmp4) &&

export APP_SRC_URL="http://${NEXUS_URL}/repository/maven-snapshots/${repoPath}/${version}/${APP_NAME}-${REPO_VERSION}.war" &&
ansible-playbook -v -i ./ansible_provisioning/hosts --extra-vars "host=production" ./ansible_provisioning/playbook.yml

'''
      }
    }

  }
}

}
environment {
APP_VERSION = '1'
NEXUS_VERSION = 'nexus3'
NEXUS_PROTOCOL = 'http'
NEXUS_URL = '192.168.0.30:8081'
NEXUS_REPOSITORY = 'maven-snapshots'
NEXUS_CREDENTIAL_ID = 'nexus-credentials'
SONARQUBE_URL = 'http://192.168.0.30'
SONARQUBE_PORT = '9000'
}
options {
skipDefaultCheckout()
}
}
