
	node {

		def APP_NAME = 'open-api-import'
		def PROJECT_NAME = 'open-api'
		def WORKING_DIR = 'open-api-import'
		def SERVICE_ACCOUNT = "'cicduser:5MWkA!vf)Wp?MS8K1'"
		def SERVICE_ACCOUNT_ESC = "cicduser:5MWkA%21vf%29Wp%3FMS8K1"
		
		def GIT_REPO = "https://${SERVICE_ACCOUNT_ESC}@tasktrack.telekom.at/bitbucket/scm/md/${APP_NAME}.git"
		def IMAGE = 'lvpmkrhsat.onevip.mk:5000/one_vip-ocp311-redhat-openjdk-18_openjdk18-openshift'
		def ENV_VARIABLES = ' "OPENAPI_IMPORTAPI_NAME=Import App" ' + 
                            '"OPENAPI_IMPORTAPI_BASE_PATH=importapi/" ' +
                            '"OPENAPI_IMPORTAPI_VERSION=1.0" ' +
                            '"OPENAPI_IMPORTAPI_ENDPOINT=/importapi" ' + 
                            '"OPENAPI_IMPORTAPI_LIMIT=6" ' +
                            '"OPENAPI_DB_LOG_URL=http://open-api-db-log-open-api.osdev.onevip.mk" ' +
                            '"OPENAPI_DB_LOG_GET_TARIFF_INFO_PATH=getTariffInfo" ' +
                            '"OPENAPI_DB_ERROR_LOG_URL=http://open-api-db-error-log-open-api.osdev.onevip.mk/" '
		
		def TEST_ENV_VARIABLES = [
                            "OPENAPI_IMPORTAPI_NAME=Import App",
                            "OPENAPI_IMPORTAPI_BASE_PATH=importapi/",
                            "OPENAPI_IMPORTAPI_VERSION=1.0",
                            "OPENAPI_IMPORTAPI_ENDPOINT=/importapi",
                            "OPENAPI_IMPORTAPI_LIMIT=6",
                            "OPENAPI_DB_LOG_URL=http://open-api-db-log-open-api.osdev.onevip.mk",
                            "OPENAPI_DB_LOG_GET_TARIFF_INFO_PATH=getTariffInfo",
                            "OPENAPI_DB_ERROR_LOG_URL=http://open-api-db-error-log-open-api.osdev.onevip.mk/"
                            ]
		
		
		def EXTERNAL_DEPENDENCY_REPO = 'https://tasktrack.telekom.at/artifactory/a1mk-openshift-release-local/'
		
		def JOB_DIRECTORY = ''
		
		def CREATE_ROUTE = 'true'
		def CREATE_ENV_VARIABLES = 'true'
		def CREATE_AUTOSCALER = 'false'
		def CREATE_HEALTH_PROBES = 'true'
		def EXTERNAL_DEPENDENCY = 'true'
		
		def mvnHome = tool name: 'MVN3', type: 'maven'
		def mvnCmd = "${mvnHome}/bin/mvn -Dhttp.proxyHost=proxymk.win.vipnet.hr -Dhttp.proxyPort=8080 -Dhttps.proxyHost=proxymk.win.vipnet.hr -Dhttps.proxyPort=8080 -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true"
		
		// Checkout Source Code
		stage('Checkout Source') {
		
		  sh "rm -rf ${WORKING_DIR}"
		  sh "git clone ${GIT_REPO}"
		  
		  echo "Git clone finished successfully!"
		  
		}
		
		// Install external jar dependencies
		stage ('Install external dependecies') {
		
			if ("${EXTERNAL_DEPENDENCY}" == "true") {
			
				echo "Installing external dependencies"
				
                sh "curl -u${SERVICE_ACCOUNT} -O -w '%{http_code} ' -k '${EXTERNAL_DEPENDENCY_REPO}/lib/dblog-api-dto/1.0/dblog-api-dto-1.0.jar'"
			    sh "${mvnCmd} install:install-file -Dfile=dblog-api-dto-1.0.jar -DgroupId=com.devoteam -DartifactId=dblog-api-dto -Dversion=1.0 -Dpackaging=jar"
				
			} else {
				
				echo "No external dependencies found ... Skipping step."
				
			}
			
		}
		
		dir(WORKING_DIR){
			
			// Extract version and other properties from the pom.xml
			echo "Extract version, artifactId and groupId from pom.xml"
			
			def version = readMavenPom().getVersion()
			def artifactId = readMavenPom().getArtifactId()
			def groupId = readMavenPom().getGroupId()
			
			echo "Version: ${version}"
            echo "ArtifactId: ${artifactId}"
			echo "GroupId: ${groupId}"
			
			JOB_DIRECTORY = pwd()
			
			// Using Maven build the jar file, in this step tests are not run
			stage('Build Application Binary') {
			
				echo "Building jar file"
				sh "${mvnCmd} clean package -DskipTests=true"
				
			}
			
			// Using Maven run the unit tests
			withEnv(TEST_ENV_VARIABLES) {
				stage('Unit Tests') {
				
                    echo "Running Unit Tests"
					sh "${mvnCmd} test -DskipTests=false"
					
				}
			}
			
			// Publish the built jar file to JFrog as SNAPSHOT
			stage('Publish to JFrog') {
				
				echo "Publish to JFrog"
				
				// ## Example of how to publish to artifactory for storing generated jar files
				// sh "${mvnCmd} deploy:deploy-file -DgroupId=${groupId} -DartifactId=${artifactId} -Dversion=${devTag} -Dpackaging=jar -DrepositoryId=nexus -Durl=http://nexus3.cicd.svc.cluster.local:8081/repository/releases -Dfile=target/${artifactId}-${pomVersion}.jar -DskipTests=true -DpomFile=pom.xml"
			
			}
		
			// Create or replace Image builder artifacts
			stage('Create Image Builder') {
			
				echo "Creating Image Builder"
				sh "if oc get bc ${APP_NAME} --namespace=cicd; \
					then echo \"exist\"; \
					else oc new-build --binary=true --name=${APP_NAME} ${IMAGE} --labels=app=${APP_NAME} -n cicd;fi"
			
			}
			
			// Build the OpenShift Image in OpenShift.
			stage('Build and Tag OpenShift Image') {
			
				// Start Binary Build in OpenShift CICD project using the file we just published
				
				echo "Building OpenShift container image ${APP_NAME}"
				sh "oc start-build ${APP_NAME} --follow --from-file=target/${artifactId}-${version}.jar -n cicd"
				echo "oc start build complete."
			
			}
		}
		
		// Copy image to project
		stage('Copy Image to Project'){
			openshift.withCluster(){
			  
			  echo "Taging image so it can be accessed from another project"
			  sh "oc tag ${APP_NAME}:latest ${PROJECT_NAME}/${APP_NAME}:latest"
			  
			}
			
			// Deploy the built image.
			stage('Configure Deployment') {
				echo "Deploying container image"
				openshift.withCluster(){

					  // Switch to target project on remote cluster
					openshift.withProject( "${PROJECT_NAME}" ) {
					  
						echo "Using project: ${openshift.project()}"

						//Create application if it does not exist
						def deploymentSelector = openshift.selector( "dc", "${APP_NAME}")
						def deploymentExists = deploymentSelector.exists()
						if (!deploymentExists) {
						  echo "Deployment ${APP_NAME} does not exist"
						  sh "oc new-app ${PROJECT_NAME}/${APP_NAME}:latest --name=${APP_NAME} --namespace=${PROJECT_NAME} --allow-missing-imagestream-tags=true"
						  
						  // If error related to environment variables exists, return if statement for env variables inside this if statement
						  
						}
						
						echo "Deployment ${APP_NAME} exists"
						
						// Create env variables if required
						if ("${CREATE_ENV_VARIABLES}" == "true") {
							
							echo "Setting environment variables"
							sh 'oc set env -n ' + PROJECT_NAME + ' dc/' + APP_NAME + ENV_VARIABLES
							
						}
						
						// Create health probes if required
						if ("${CREATE_HEALTH_PROBES}" == "true") {
							
							//Set health probes
							echo "Creating liveness and readiness probes"
							openshift.raw("set","probe","dc/${APP_NAME}","--namespace=${PROJECT_NAME}","--liveness","--failure-threshold","5","--initial-delay-seconds","120","--get-url=http://:8080/actuator/health")
							openshift.raw("set","probe","dc/${APP_NAME}","--namespace=${PROJECT_NAME}","--readiness","--failure-threshold 5","--initial-delay-seconds","120","--get-url=http://:8080/actuator/health")
							
						}

						// Create an external facing route if required
						if ("${CREATE_ROUTE}" == "true") {
							if(openshift.selector("route", APP_NAME).exists()) {
								echo "Route already exposed."
							}else{
								sh 'oc expose service ' + APP_NAME + ' -n ' + PROJECT_NAME
							}
						}

						//Update the Image on the Development Deployment Config
						openshift.raw("set","image dc/${APP_NAME}","${APP_NAME}=${APP_NAME}:latest","--source=imagestreamtag",
								"-n ${PROJECT_NAME}")
						
						
						// Create autoscaler for container if required
						if ("${CREATE_AUTOSCALER}" == "true") {
							
							// Create autoscaler
							echo "Setting autoscaling for container"
							if(openshift.selector("hpa", APP_NAME).exists()) {
							
								echo "Autoscaler already exists for " + APP_NAME
								echo "Setting resources for autoscaler"
								sh 'oc set resources dc/' + APP_NAME + ' --limits cpu=150m --requests cpu=10m'
								
							}else{
								echo "Autoscaler doesnt exist for " + APP_NAME + ", creating new autoscaler"
								sh 'oc autoscale dc/' + APP_NAME + ' --namespace=' + PROJECT_NAME + ' --max 10 --cpu-percent=80'
								sh 'oc set resources dc/' + APP_NAME + ' --limits cpu=150m --requests cpu=10m'
								
							}
						}
					}
				}
			}
		}
		stage("Rollout to Developmemt"){
		
			openshift.withCluster(){
			
				openshift.withProject( "${PROJECT_NAME}" ) {
				
					if(openshift.selector("dc", APP_NAME).exists()){
					
						def deployment = openshift.selector('dc', "${APP_NAME}")

						timeout(time:10, unit:'MINUTES') {
						  deployment.rollout().status()
						}
					}
				}
			}
		}
		stage("Publish built JAR to JFrog artifactory"){
			openshift.withCluster(){
			
				openshift.withProject( "cicd" ) {
				
					dir(JOB_DIRECTORY){
						
						echo "Starting upload of JAR file"
						
						def version = readMavenPom().getVersion()
						def artifactId = readMavenPom().getArtifactId()
						
						sh "curl -u${SERVICE_ACCOUNT} -T target/${artifactId}-${version}.jar -O '${EXTERNAL_DEPENDENCY_REPO}/${APP_NAME}/${artifactId}-${version}.jar'"
						
						echo "Finished upload of JAR file"
						
					}
				}
			}
		}
		
	}