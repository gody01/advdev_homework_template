#!groovy
podTemplate(
  label: "skopeo-pod",
  cloud: "openshift",
  inheritFrom: "maven",
  containers: [
    containerTemplate(
      name: "jnlp",
      image: "docker-registry.default.svc:5000/${GUID}-jenkins/jenkins-agent-appdev",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi",
      resourceRequestCpu: "1",
      resourceLimitCpu: "2"
    )
  ]
) {
  node('skopeo-pod') {
    // Define Maven Command to point to the correct
    // settings for our Nexus installation
    def mvnCmd = "mvn -s ../nexus_settings.xml"

    // Checkout Source Code.
    stage('Checkout Source') {
      checkout scm
    }

    // Build the Tasks Service
    dir('openshift-tasks') {
      // The following variables need to be defined at the top level
      // and not inside the scope of a stage - otherwise they would not
      // be accessible from other stages.
      // Extract version from the pom.xml
      def version = getVersionFromPom("pom.xml")

      // TBD Set the tag for the development image: version + build number
      def devTag  = "${version}-${BUILD_NUMBER}"
      // Set the tag for the production image: version
      def prodTag = "${version}"

      // Using Maven build the war file
      // Do not run tests in this step
      stage('Build war') {
        echo "Building version ${devTag}"
	sh "${mvnCmd} clean package -DskipTests"
      }


      // TBD: The next two stages should run in parallel
      parallel (

      	// Using Maven run the unit tests
      	stage('Unit Tests') {
      	  echo "Running Unit Tests"
	  sh "${mvnCmd} test"
      	}

      	// Using Maven to call SonarQube for Code Analysis
      	stage('Code Analysis') {
          echo "Running Code Analysis"
	  sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.gpte-hw-cicd.svc.cluster.local:9000 -Dsonar.projectName=${JOB_BASE_NAME}-${devTag} -Dsonar.projectVersion=${devTag}"
	}
      )

      // Publish the built war file to Nexus
      stage('Publish to Nexus') {
        echo "Publish to Nexus"
	sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.gpte-hw-cicd.svc.cluster.local:8081/repository/all-maven-public"
      }

      // Build the OpenShift Image in OpenShift and tag it.
      stage('Build and Tag OpenShift Image') {
        echo "Building OpenShift container image tasks:${devTag}"

        // TBD: Build Image, tag Image

        sh "oc start-build tasks --follow --from-file=http://nexus3.gpte-hw-cicd.svc.cluster.local:8081/repository/all-maven-public/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war -n mg-tasks-dev"
        openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: 'mg-tasks-dev', namespace: 'mg-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'

      }

      // Deploy the built image to the Development Environment.
      stage('Deploy to Dev') {
        echo "Deploying container image to Development Project"

        // TBD: Deploy to development Project
        //      Set Image, Set VERSION
        //      Make sure the application is running and ready before proceeding
        sh "oc set image dc/tasks tasks=docker-registry.default.svc:5000/mg-tasks-dev/tasks:${devTag} -n mg-tasks-dev"

	sh "oc set env  dc/tasks VERSION='${devTag} (tasks-dev)'"

        openshiftDeploy depCfg: 'tasks', namespace: 'mg-tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'

        openshiftVerifyDeployment depCfg: 'tasks', namespace: 'mg-tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
        
        openshiftVerifyService namespace: 'mg-tasks-dev', svcName: 'tasks', verbose: 'false'

//	sh "curl -s -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks-mg-tasks-dev.apps.{$GUID}.openshift.opentlc.com/ws/tasks/integration_test_1 | grep \"HTTP/1.1 201 Created\""
//	sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks-mg-tasks-dev.apps.{$GUID}.openshift.opentlc.com/ws/tasks/1 | grep \"<title>integration_test_1</title>\""
//	sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks-mg-tasks-dev.apps.{$GUID}.openshift.opentlc.com/ws/tasks/1 | grep \"HTTP/1.1 204 No Content\""
      }

      // Copy Image to Nexus container registry
      stage('Copy Image to Nexus container registry') {
        echo "Copy image to Nexus container registry"

        // TBD: Copy image to Nexus container registry
    	sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:redhat docker://docker-registry.default.svc.cluster.local:5000/mg-tasks-dev/tasks:${devTag} docker://nexus-registry.mg-nexus.svc.cluster.local:5000/tasks:${devTag}"

        // TBD: Tag the built image with the production tag.
      }

      // Blue/Green Deployment into Production
      // -------------------------------------
      def destApp   = "tasks-green"
      def activeApp = ""

      stage('Blue/Green Production Deployment') {
        // TBD: Determine which application is active
        //      Set Image, Set VERSION
        //      Deploy into the other application
        //      Make sure the application is running and ready before proceeding
	sh "oc set env  dc/tasks VERSION='${prodTag} (tasks-blue)'"
      }

      stage('Switch over to new Version') {
        echo "Switching Production application to ${destApp}."
        // TBD: Execute switch
	sh "oc set env  dc/tasks VERSION='${prodTag} (tasks-green)'"
      }
    }
  }
}

// Convenience Functions to read version from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}