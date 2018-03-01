def version, mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"

pipeline {
  agent {
    node {
        label 'maven'
    } //node
  } //agent
  stages {
    stage('Build App') {
      steps {
        git branch: 'eap-7', url: 'http://gogs:3000/gogs/openshift-tasks.git'
        script {
            def pom = readMavenPom file: 'pom.xml'
            version = pom.version
        } //script
        sh "${mvnCmd} install -DskipTests=true"
      } //steps
    } //stage
    stage('Test') {
      steps {
        sh "${mvnCmd} test"
        step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
      } //steps
    } //stage
    stage('Code Analysis') {
      steps {
        script {
          if (env.WITH_SONAR.toBoolean()) {
            sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
          } else {
            sh "${mvnCmd} site -DskipTests=true"

            step([$class: 'CheckStylePublisher', unstableTotalAll:'300'])
            step([$class: 'PmdPublisher', unstableTotalAll:'20'])
            step([$class: 'FindBugsPublisher', pattern: '**/findbugsXml.xml', unstableTotalAll:'20'])
            step([$class: 'JacocoPublisher'])
            publishHTML (target: [keepAll: true, reportDir: 'target/site', reportFiles: 'project-info.html', reportName: "Site Report"])
          } //if
        } //script
      } //steps
    } //stage
    stage('Archive App') {
      steps {
        sh "${mvnCmd} deploy -DskipTests=true -P nexus3"
      } //steps
    } //stage
    stage('Create Image Builder') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              return !openshift.selector("bc", "${env.APP_NAME}").exists();
            } //openshift.withproject
          } //openshift.withcluster
        } //expression
      } //when
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.newBuild("--name=${env.APP_NAME}", "--image-stream=jboss-eap70-openshift:1.5", "--binary=true")
            } //openshift.withproject
          } //openshift.withcluster
        } //script
      } //steps
    } //stage
    stage('Build Image') {
      steps {
        sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp target/openshift-tasks.war oc-build/deployments/ROOT.war"

        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("bc", "${env.APP_NAME}").startBuild("--from-dir=oc-build", "--wait=true")
            } //openshift.withproject
          } //openshift.withcluster
        } //script
      } //steps
    } //stage
    stage('Create DEV') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              return !openshift.selector('dc', "${env.APP_NAME}").exists()
            } //openshift.withproject
          } //openshift.withcluster
        } //expression
      } //when
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              def app = openshift.newApp("${env.APP_NAME}:latest")
              app.narrow("svc").expose();

              def dc = openshift.selector("dc", "${env.APP_NAME}")
              while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                  sleep 10
              } //while
              openshift.set("triggers", "dc/${env.APP_NAME}", "--remove-all")
            } //openshift.withproject
          } //openshift.withcluster
        } //script
      } //steps
    } //stage
    stage('Deploy DEV') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("dc", "${env.APP_NAME}").rollout().latest();
            } //openshift.withproject
          } //openshift.withcluster
        } //script
      } //steps
    } //stage
    stage('Promote to Test?') {
      steps {
        timeout(time:7, unit:'DAYS') {
            input message: "Promote to Test?", ok: "Promote"
        } //timeout
      } //steps
    } //stage
} //stages

//  node {
//    label 'jenkins-slave-image-mgmt'
//  } //node

//  stages {
    stage ('tag and push') {
      container('jenkins-slave-image-mgmt')
        script {
            sh """
            set +x
            imageRegistry=\$(${env.OC_CMD} get is ${env.APP_NAME} --template='{{ .status.dockerImageRepository }}' -n ${env.DEV_PROJECT} | cut -d/ -f1)

            echo "Promoting ${imageRegistry}/${env.DEV_PROJECT}/${env.APP_NAME} -> ${imageRegistry}/${env.TEST_PROJECT}/${env.APP_NAME}"

            skopeo --tls-verify=false copy --remove-signatures --src-creds ${env.DEV_REGISTRY_USER}:${env.DEV_REGISTRY_TOKEN} --dest-creds ${env.TEST_REGISTRY_USER}:${env.TEST_REGISTRY_TOKEN} docker://${env.DEV_REGISTRY}/${env.DEV_PROJECT}/${env.APP_NAME} docker://${env.TEST_REGISTRY}/${env.STAGE_PROJECT}/${env.APP_NAME}
            """
            } //script
    } //stage

     stage('Deploy Test') {
        steps {
          script {
            openshift.withCluster('test') {
              openshift.withProject(env.STAGE_PROJECT) {
                if (openshift.selector('dc', 'tasks').exists()) {
                  openshift.selector('dc', 'tasks').delete()
                  openshift.selector('svc', 'tasks').delete()
                  openshift.selector('route', 'tasks').delete()
                } //if

                openshift.newApp("tasks:${version}").narrow("svc").expose()
              } //openshift.withProject
            } // openshift.withCluster
          } //script
        } //steps
    } //stage
} //pipeline
