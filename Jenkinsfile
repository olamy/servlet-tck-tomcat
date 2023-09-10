#!groovy

pipeline {
  agent { node { label 'linux' } }

  options {
    buildDiscarder logRotator( numToKeepStr: '5' )
  }
  parameters {
    string( defaultValue: 'tckrefactor', description: 'GIT branch name to build TCK (master/tckrefactor)',
            name: 'TCK_BRANCH' )

    choice(
            description: 'TCK Github org',
            name: 'GITHUB_ORG_TCK',
            choices: ['jakartaee','olamy']
    )
    string( defaultValue: 'jdk17', description: 'JDK to run TCK', name: 'JDKBUILD' )
  }

  stages {

    stage("Checkout Build TCK Sources") {
      steps {
        ws('tck') {
          deleteDir()
          checkout([$class: 'GitSCM',
                    branches: [[name: "*/$TCK_BRANCH"]],
                    extensions: [[$class: 'CloneOption', depth: 1, noTags: true, shallow: true]],
                    userRemoteConfigs: [[url: 'https://github.com/${GITHUB_ORG_TCK}/servlet']]])
          timeout(time: 30, unit: 'MINUTES') {
            withEnv(["JAVA_HOME=${tool "$JDKBUILD"}",
                     "PATH+MAVEN=${env.JAVA_HOME}/bin:${tool 'maven3'}/bin",
                     "MAVEN_OPTS=-Xms2g -Xmx4g -Djava.awt.headless=true"]) {
              configFileProvider([configFile(fileId: 'oss-settings.xml', variable: 'GLOBAL_MVN_SETTINGS')]) {
                //sh "mvn -ntp install:install-file -Dfile=./lib/javatest.jar -DgroupId=javatest -DartifactId=javatest -Dversion=5.0 -Dpackaging=jar"
                sh "mvn -ntp -s $GLOBAL_MVN_SETTINGS -V -B clean install -e -Dmaven.build.cache.remote.url=dav:http://nginx-cache-service.jenkins.svc.cluster.local:80 -Dmaven.build.cache.remote.enabled=true -Dmaven.build.cache.remote.save.enabled=true -Dmaven.build.cache.remote.server.id=remote-build-cache-server"
              }
            }
          }
        }
      }
    }

    stage("Run TCK") {
      steps {
        timeout(time: 90, unit: 'MINUTES') {
          withEnv(["JAVA_HOME=${tool "$JDKBUILD"}",
                   "PATH+MAVEN=${env.JAVA_HOME}/bin:${tool "maven3"}/bin",
                   "MAVEN_OPTS=-Xms4g -Xmx8g -Djava.awt.headless=true"]) {
            configFileProvider([configFile(fileId: 'oss-settings.xml', variable: 'GLOBAL_MVN_SETTINGS')]) {
              sh "mvn -nsu -ntp -s $GLOBAL_MVN_SETTINGS -Dmaven.test.failure.ignore=true -V -B -U clean verify -e"
            }
          }
        }
      }
      post {
        always {
          junit testResults: '**/surefire-reports/TEST-**.xml'
        }
      }
    }
  }
}
