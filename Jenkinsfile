pipeline {
  agent any
    tools {
        maven "MAVEN"
        jdk "JDK"
    }
 triggers {
       pollSCM('*/5 * * * *')
    }
  
 options { disableConcurrentBuilds() }
  
  stages {  
    stage('Initialize'){
            steps{
                echo "PATH = ${M2_HOME}/bin:${PATH}"
                echo "M2_HOME = /opt/maven"
            }
      } 
    stage('Test App') {
      steps {
        sh "mvn clean test"
      }
    }
    stage('Quality Check :: Sonarqube & JaCoCo') {
      steps {
        sh "mvn sonar:sonar -Dsonar.host.url=https://sonar-cicd-app.apps.rosa.oliclu3.oafh.p3.openshiftapps.com -Dsonar.login=admin -Dsonar.password=P@ssw0rd"
      }
    }
    stage('Insatll App') {
      steps {
        sh "mvn install"
      }
    }
    stage('Create Image Builder') {
      when {
        expression {
          openshift.withCluster() {
            return !openshift.selector("bc", "springbootapp").exists();
          }
        }
      }
      steps {
        script {
          openshift.withCluster() {
            openshift.newBuild("--name=springbootapp","--image-stream=openjdk18-openshift:1.1", "--binary")
          }
        }
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.selector("bc", "springbootapp").startBuild("--from-file=target/spring-example-0.0.1-SNAPSHOT.jar", "--wait")
          }
        }
      }
    }
    stage('Promote to UAT') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("springbootapp:latest", "springbootapp:uat")
          }
        }
      }
    }
    stage('Create UAT') {
      steps {
        script {
           openshift.withCluster() {
           openshift.withProject("cicd-app") {
          def deployment = openshift.selector("dc", "springbootapp-uat")
                    
          if(!deployment.exists()){
            openshift.newApp("springbootapp:latest", "--name=springbootapp-uat").narrow('svc').expose()
          }
          timeout(5) { 
                openshift.selector("dc", "springbootapp-uat").related('pods').untilEach(1) {
                  return (it.object().status.phase == "Running")
                  }          
              }
          }
        }
      }
      }
    }
    stage('Promote PROD') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("springbootapp:uat", "springbootapp:prod")
          }
        }
      }
    }
    stage('Create PROD') {          
      steps {
        script {
           openshift.withCluster() {
           openshift.withProject("cicd-app") {
          def deployment = openshift.selector("dc", "springbootapp-prod")
                    
          if(!deployment.exists()){
            openshift.newApp("springbootapp:prod", "--name=springbootapp-prod").narrow('svc').expose()
          }
          timeout(5) { 
                openshift.selector("dc", "springbootapp-prod").related('pods').untilEach(1) {
                  return (it.object().status.phase == "Running")
                  }          
              }
          }
        }
        }
      }
    }
  }
}
