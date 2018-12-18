#!groovy.
import groovy.json.JsonOutput
import groovy.transform.Field
import groovy.json.JsonSlurperClassic

// To be replaced as @Field def repo_credential_id = "value" for repo_credential_id, repo_base and repo_core
@Field def repo_credential_id = "jazz_repocreds"
@Field def repo_base = "18.235.7.234"
@Field def repo_core = "slf"
@Field def scm_type = "gitlab"

/**
 * The Service delete workflow for service types: API, function & website
*/

@Field def configModule
@Field def configLoader
@Field def scmModule
@Field def events
@Field def utilModule
@Field def serviceMetadataLoader
@Field def environmentMetadataLoader
@Field def sonarModule
@Field def lambdaEvents

@Field def g_base_url = ''
@Field def g_svc_admin_cred_ID = 'SVC_ADMIN'
@Field def auth_token = ''
@Field def service_config
@Field def context_map = [:]
@Field def is_service_deletion = true;
@Field def current_environment_id
node {
	echo "Starting delete service job with params : $params"

	jazzBuildModuleURL = getBuildModuleUrl()
	loadBuildModules(jazzBuildModuleURL)

	def jazz_prod_api_id = utilModule.getAPIIdForCore(configLoader.AWS.API["PROD"])
	g_base_url = "https://${jazz_prod_api_id}.execute-api.${configLoader.AWS.REGION}.amazonaws.com/prod"

	def version
	def repo_name
	def tracking_id
	def cloudfrontEnabled
	def flowType
	def service_id = params.db_service_id
	def environmentIds = []

	if (params.version) {
		version = params.version.trim()
	}

	if (params.tracking_id) {
		tracking_id = params.tracking_id.trim()
	}

	if (params.environment_id) {
		environment_id = params.environment_id
		environmentIds.push(environment_id.trim())
		is_service_deletion = false
	}


	auth_token = setCredentials()

	stage("Initialization") {

		if (params.domain && params.domain != "") {
			repo_name = params.domain + "_" + params.service_name
		}
		sh 'rm -rf ' + repo_name
		sh 'mkdir ' + repo_name

		checkoutSCM(repo_name)

		def configObj = dir(repo_name)
		{
			return LoadConfiguration()
		}

		if (configObj.service_id) {
			service_config = serviceMetadataLoader.loadServiceMetadata(configObj.service_id)
		} else {
			error "Service Id is not available."
		}

		if (!service_config) {
			error "Failed to fetch service metadata from catalog"
		}
		environmentMetadataLoader.initialize(service_config, configLoader, scmModule, null, env.BUILD_URL, env.BUILD_ID, g_base_url + "/jazz/environments", auth_token)
		def environmentList = environmentMetadataLoader.getEnvironmentLogicalIds()
		echo "Environment List: $environmentList"

		if (!events) { error "Can't load events module" } //Fail here
		events.initialize(configLoader, service_config, "SERVICE_DELETION", "", "", g_base_url + "/jazz/events")
		context_map = [created_by : service_config['created_by']]
		sonarModule.initialize(configLoader, service_config, "")

		if (environmentIds.size() > 0) {
			//validate each id in environment catalog
			for (_eId in environmentIds) {
				if (!environmentList.contains(_eId)) {
					events.sendFailureEvent('INITIALIZE_DELETE_WORKFLOW', "unable to find the environment id $_eId from environment catalog")
				}
			}
		} else if (environmentList.size() > 0) {
			for (_env in environmentList) {
				environmentIds.push(_env)
			}
		} else if (environmentList.size() == 0) {
			echo "$repo_name doesn't contain any active environment logical id in environment catalog!"
		}


		dir(repo_name){
			echo "loadServerlessConfig......."

			if (fileExists('build.api')) {
				flowType = "API"
				loadServerlessConfig()
				updateSwaggerConfig()
			} else if (fileExists('build.lambda')) {
				flowType = "LAMBDA"
				loadServerlessConfig()
			} else if (fileExists('build.website')) {
				flowType = "WEBSITE"
				if (service_config['create_cloudfront_url']) {
					cloudfrontEnabled = service_config['create_cloudfront_url'].trim()
				} else {
					cloudfrontEnabled = "false"
				}
			} else {
				error "Invalid project configuration"
			}
		}
	}


	dir(repo_name){
		switch (flowType) {
			case "API":
				stage('Undeploy Service') {
					updateServiceNameConfig()
					def path = getResourcePath()
					for (_envId in environmentIds) {
						current_environment_id = _envId
						try {
							def branch = environmentMetadataLoader.getEnvironmentBranchName(_envId)
							if (branch && branch != 'NA') {
								environmentMetadataLoader.setBranch(branch)
								events.setBranch(branch)
								sonarModule.setBranch(branch)
								branch = 'NA'
							}
							events.setEnvironment(_envId)
							events.sendStartedEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " started", null, _envId)
							echo 'undeploying service for environment: ' + _envId

							cleanUpApiGatewayResources(_envId, path)
							cleanUpApiDocs(_envId)
							unDeployService(_envId)
							if (configLoader.CODE_QUALITY.SONAR.ENABLE_SONAR == "true" && configLoader.CODE_QUALITY.SONAR.CLEANUP == "true") {
								cleanupCodeQualityReports()
							}
              archiveAssetDetails(_envId)
							events.sendCompletedEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " completed", null, _envId)
						} catch (ex) {
							echo "Environment Deletion failed for envId: $_envId . Error Details:" + ex
							events.sendFailureEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " failed! " + ex.getMessage(), null, _envId)
							error "Environment Deletion failed for envId: $_envId . Error Details:" + ex
						}
					}
				}
				break

			case "LAMBDA":
				stage('Undeploy Service') {
					updateServiceNameConfig()

          for (_envId in environmentIds) {
						current_environment_id = _envId

            try {
							def branch = environmentMetadataLoader.getEnvironmentBranchName(_envId)
							if (branch && branch != 'NA') {
								events.setBranch(branch)
								sonarModule.setBranch(branch)
								branch = 'NA'
							}
							events.setEnvironment(_envId)
							events.sendStartedEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " started", null, _envId)

							if (service_config['event_source_s3']) { // check service for S3 Bucket and make empty if not. //
								checkBucket(current_environment_id);
							}
							echo 'undeploying function for environment: ' + _envId

              cleanupEventSourceMapping(_envId)
              unDeployService(_envId)

							if (configLoader.CODE_QUALITY.SONAR.ENABLE_SONAR == "true" && configLoader.CODE_QUALITY.SONAR.CLEANUP == "true") {
								cleanupCodeQualityReports()
							}
              archiveAssetDetails(_envId)
							events.sendCompletedEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " completed", null, _envId)
						} catch (ex) {
							events.sendFailureEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " failed! " + ex.getMessage(), null, _envId)
							error "Environment cleanup for " + _envId + " failed! " + ex.getMessage()
						}
					}
				}
				break

			case "WEBSITE":
				stage('Undeploy Website') {
					for (_envId in environmentIds) {
						current_environment_id = _envId
						try {
							def branch = environmentMetadataLoader.getEnvironmentBranchName(_envId)
							echo "Branch ::: $branch"
							echo "ENVIRONMENT ::: $_envId"
							if (branch && branch != 'NA') {
								events.setBranch(branch)
								sonarModule.setBranch(branch)
								branch = 'NA'
							}
							events.setEnvironment(_envId)
							events.sendStartedEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " started", null, _envId)
							cloudfrontEnabled = "true"  //TODO: Remove when non-cloudFront websites feature is introduced.
							if (cloudfrontEnabled == "true") {
								cleanupCloudFrontDistribution(_envId)
							}
							unDeployWebsite(_envId)
              archiveAssetDetails(_envId)
							events.sendCompletedEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " completed", null, _envId)
						} catch (ex) {
							echo "error occured when deleting environment: " + ex
							events.sendFailureEvent('DELETE_ENVIRONMENT', "Environment cleanup for " + _envId + " failed! " + ex.getMessage(), null, _envId)
							error "Environment cleanup for " + _envId + " failed! " + ex.getMessage()
						}
					}
				}
				break
		}
	}

	if (is_service_deletion) {
		stage('Cleanup SCM') {
			cleanup(repo_name)
			events.sendCompletedEvent('DELETE_PROJECT', 'deletion completed', context_map)
		}
	}
}

def cleanupEventSourceMapping(env) {
  def lambda_arn = "arn:aws:lambda:${configLoader.AWS.REGION}:${configLoader.AWS.ACCOUNTID}:function:${configLoader.INSTANCE_PREFIX}-${service_config['domain']}-${service_config['service']}-${env}"
  def assets_api = g_base_url + "/jazz/assets"
  lambdaEvents.deleteEventSourceMapping(lambda_arn, assets_api, auth_token,service_config, env)
}

def archiveAssetDetails(env) {
  def assets_api = g_base_url + "/jazz/assets"
  def assets = utilModule.getAssets(assets_api, auth_token, service_config, env)
  def assetList = parseJsonMap(assets)
  for (asset in assetList.data.assets) {
    events.sendCompletedEvent('UPDATE_ASSET', "Environment cleanup for ${env} completed", utilModule.generateAssetMap("aws", asset.provider_id , asset.asset_type, service_config), env)
  }
}

/**
 * Check bucket existance and content in it. if yes then empty it for SLS delete
 * @param  stage
 * @return
 */
def checkBucket(stage) {
	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"

			// Empty bucket
			def targetBucket = service_config['event_source_s3'];
			if(stage != "prod"){
				targetBucket = service_config['event_source_s3'] + "-" + stage;
			}
      def s3Exists = true;
      try {
          sh "aws s3api head-bucket --bucket ${targetBucket} --output json"
      } catch (ex) {
          echo "Bucket does not exist"
          s3Exists = false
      }
     if(s3Exists && (!isBucketEmpty(targetBucket))) {
					sh "aws s3 rm s3://${targetBucket}/ --recursive --exclude '/'"
					echo "Removing items from bucket"
			}
		} catch (ex) {
			handleFailureEvent(ex.getMessage())
		}
	}
}

def cleanupCodeQualityReports(){
	try {
		sonarModule.cleanupCodeQualityReports()
	} catch (ex) {
		echo "error occured while deleting code quality reports: " + ex.getMessage()
	}
}

def handleFailureEvent(errorMessage){
	if (is_service_deletion) {
		events.sendFailureEvent('DELETE_PROJECT', errorMessage, context_map)
		send_status_email("FAILED")
		error "deleteProject failed: " + errorMessage
	} else {
		echo "error occured when Deleting  Environment: " + errorMessage
		send_status_email("FAILED")
		events.sendFailureEvent('DELETE_ENVIRONMENT', errorMessage, null, current_environment_id)
		error "Environment cleanup for " + current_environment_id + " failed! " + errorMessage
	}
}

/**
 * Calls the serverless remove to undeploy the lambda service
 * @param  stage
 * @return
 */
def unDeployService(stage) {
	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"

			def env_key
			if (stage.endsWith("-dev")) {
				env_key = "DEV"
			} else {
				env_key = "PROD"
			}
			def envBucketKey = "${env_key}${configLoader.JAZZ.S3_BUCKET_NAME_SUFFIX}"
			sh "serverless remove --stage ${stage} --verbose  --bucket ${configLoader.AWS.S3[envBucketKey]}"

			echo "Service undeployed"
		} catch (ex) {
			handleFailureEvent(ex.getMessage())
		}
	}
}

/**
 * Checkout Code
 * @param  repo_name
 * @return
 */
def checkoutSCM(repo_name) {
	dir(repo_name)
	{
		def repo_url = scmModule.getRepoCloneUrl(repo_name)
		try {
			checkout([$class: 'GitSCM', branches: [
				[name: '*/master']
			], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
					[credentialsId: configLoader.REPOSITORY.CREDENTIAL_ID, url: repo_url]
				]])
		} catch (ex) {
			error "checkout SCM failed."
		}
	}
}

/**
 * Update the service name in serverless service_config file
 * @param  domain
 * @param  serviceName
 * @return
 */
def updateServiceNameConfig() {
	try {
		sh "sed -i -- 's/\${file(deployment-env.yml):service}/${configLoader.INSTANCE_PREFIX}-${service_config['domain']}-${service_config['service']}/g' serverless.yml"
		sh "sed -i -- 's/name: \${self:service}/name: ${configLoader.INSTANCE_PREFIX}-${service_config['domain']}-${service_config['service']}/g' serverless.yml"
		writeServerlessFile()
	} catch (ex) {
		handleFailureEvent(ex.getMessage())
	}
}

def writeServerlessFile(){
	sh "pwd"
	sh "sed -i -- 's/\${file(deployment-env.yml):region}/${configLoader.AWS.REGION}/g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):domain, self:provider.domain}/${service_config['domain']}/g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):owner, self:provider.owner}/${service_config['created_by']}/g' serverless.yml"
	sh "sed -i -e 's|\${file(deployment-env.yml):iamRoleARN}|${service_config['iamRoleARN']}|g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):providerRuntime}/${service_config['providerRuntime']}/g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):providerMemorySize}/${service_config['providerMemorySize']}/g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):providerTimeout}/${service_config['providerTimeout']}/g' serverless.yml"
	sh "sed -i -- 's|\${file(deployment-env.yml):eventScheduleRate}|${service_config['eventScheduleRate']}|g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):eventScheduleEnable}/${service_config['eventScheduleEnable']}/g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):securityGroupIds}/${service_config['securityGroupIds']}/g' serverless.yml"
	sh "sed -i -- 's/\${file(deployment-env.yml):subnetIds}/${service_config['subnetIds']}/g' serverless.yml"

	if (service_config['artifact']) {
		sh "sed -i -- 's/\${file(deployment-env.yml):artifact}/${service_config['artifact']}/g' serverless.yml"
	}
	if (service_config['mainClass']) {
		sh "sed -i -- 's/\${file(deployment-env.yml):mainClass}/${service_config['mainClass']}/g' serverless.yml"
	}

  if (service_config['event_source_s3']) {
    sh "sed -i -- 's/{event_source_s3}/${service_config['event_source_s3']}/g' ./serverless.yml"
    sh "sed -i -- 's/{event_action_s3}/${service_config['event_action_s3']}/g' ./serverless.yml"
  }

  if (service_config['event_source_stream']) {
    sh "sed -i -- 's/resourcesDisabled/resources/g' ./serverless.yml"
    sh "sed -i -- 's/{event_source_stream}/${service_config['event_source_stream']}/g' ./serverless.yml"
  }

  if (service_config['event_source_dynamodb']) {
    sh "sed -i -- 's/resourcesDisabled/resources/g' ./serverless.yml"
    sh "sed -i -- 's|{event_source_dynamodb}|${service_config['event_source_dynamodb']}|g' serverless.yml"
    sh "sed -i -- 's|{event_source_dynamodb}|${service_config['event_source_dynamodb']}|g' policyFile.yml"
  }

  if ( service_config['event_source_dynamodb'] || service_config['event_source_sqs']){
    sh "sed -i -e 's|\${file(deployment-env.yml):iamRoleARN}|customEventRole|g' serverless.yml"
  }else{
    sh "sed -i -e 's|\${file(deployment-env.yml):iamRoleARN}|${service_config['iamRoleARN']}|g' serverless.yml"
  }
}

/**
 * Replace the service name & Domain place holders in swagger file.
 * @param  domain
 * @param  serviceName
 * @return
 */
def updateSwaggerConfig() {
	try {
		if (fileExists('swagger/swagger.json')) {
			sh "sed -i -- 's/{service_name}/${service_config['service']}/g' swagger/swagger.json"
			sh "sed -i -- 's/{domain}/${service_config['domain']}/g' swagger/swagger.json"

			def region = "${configLoader.AWS.REGION}"
			def role = "${configLoader.AWS.ROLEID}"
			def roleARN = role.replaceAll("/", "\\\\/")

			// TODO: the below couple of statements will be replaced with regular expression in very near future;
			def roleId = roleARN.substring(roleARN.indexOf("::") + 2, roleARN.lastIndexOf(":"))

			sh "sed -i -- 's/{conf-role}/${roleARN}/g' ./swagger/swagger.json"
			sh "sed -i -- 's/{conf-region}/${region}/g' ./swagger/swagger.json"
			sh "sed -i -- 's/{conf-accId}/${roleId}/g' ./swagger/swagger.json"
		}
	} catch (ex) {
		handleFailureEvent(ex.getMessage())
	}
}


/**
 * Cleans up the project repository from SCM as well as update the database and send corresponding status email
 * @param  repoName
 * @return
 */
def cleanup(repoName) {

	try {
		scmModule.deleteProject(repoName)
		send_status_email("COMPLETED")
	} catch (ex) {
		events.sendFailureEvent('DELETE_PROJECT', ex.getMessage(), context_map)
		send_status_email("FAILED")
		error "deleteProject failed. " + ex.getMessage()
	}
}

/**
 * Clean up the API gateway resource configurations specific to the service
 * @param  stage environment
 * @param  path the resource path
 * @return
 */
def cleanUpApiGatewayResources(stage, path) {
	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"

			def resource_id = null
			def resource_search_key
			def current_environment

			if (stage.endsWith("-dev")) {
				resource_search_key = "/" + stage
				current_environment = 'dev'
			} else {
				resource_search_key = path
				current_environment = 'prod'
			}
			echo "Resource search key for ApiGateway : $resource_search_key"
			def aws_api_id = getApiId(stage)

			try {
				def outputStr = sh(
					script: "aws apigateway get-resources --rest-api-id ${aws_api_id} --region ${configLoader.AWS.REGION} --output json",
					returnStdout: true
				).trim()

				def list = parseJsonMap(outputStr)
				for (items in list["items"]) {
					if (items["path"] == resource_search_key) {
						resource_id = items["id"]
					}
				}
			} catch (ex) {
				handleFailureEvent(ex.getMessage())
			}
			if (resource_id != null && resource_id != "") {
				def status_json = sh(
					script: "aws apigateway delete-resource --rest-api-id ${aws_api_id}  --region ${configLoader.AWS.REGION} --resource-id ${resource_id} --output json",
					returnStdout: true
				).trim()
				try {
					def deployment_status_json = sh(
						script: "aws apigateway create-deployment --rest-api-id ${aws_api_id}  --region ${configLoader.AWS.REGION} --stage-name ${current_environment} --description 'API deployment after resource clean up' --output json",
						returnStdout: true
					).trim()
				} catch (ex) {
				}
			} else {
				echo "Resource Id does not exist in API gateway."
			}
		} catch (ex) {
			handleFailureEvent(ex.getMessage())
		}
	}
}

/**
 * Get the API Id of the gateway specific to an environment. The values will be pulled ENV vars set
 * @param  stage the environment
 * @return  api Id
 */
def getServiceBucket(stage) {
	if (stage == 'dev') {
		return configLoader.JAZZ.S3.WEBSITE_DEV_BUCKET;
	} else if (stage == 'stg') {
		return configLoader.JAZZ.S3.WEBSITE_STG_BUCKET;
	} else if (stage == 'prod') {
		return configLoader.JAZZ.S3.WEBSITE_PROD_BUCKET;
	}
}


/**
 * Get the API Id of the gateway specific to an environment. The value will be retrieved from environments table and if not available will try to retrieve it from config
 * @param  stage the environment
 * @return  api Id
 */
def getApiId(stage) {
	echo "stage: $stage"

	def envInfo = environmentMetadataLoader.getEnvironmentInfo()
	def envMetadata = [:]

	if (envInfo && envInfo["metadata"]) {
		envMetadata = envInfo["metadata"]
	}

	if (envMetadata["AWS_API_ID"] != null) {
		return envMetadata["AWS_API_ID"]
	}

	if (stage.endsWith('-dev')) {
		return utilModule.getAPIIdForCore(configLoader.AWS.API["DEV"])
	} else if (stage == 'stg') {
		return utilModule.getAPIIdForCore(configLoader.AWS.API["STG"])
	} else if (stage == 'prod') {
		return utilModule.getAPIIdForCore(configLoader.AWS.API["PROD"])
	}
}

/**
 * Clean up the API documentation folder from S3 corresponding to the environment
 * @param  stage the environment
 * @return  api Id
 */
def cleanUpApiDocs(stage) {
	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"
			def apiRootFolder = getApiDocsFolder(stage)
			sh "aws s3 rm s3://${apiRootFolder}/${service_config['domain']}/${service_config['service']}/${stage} --recursive"
		} catch (ex) {
			handleFailureEvent(ex.getMessage())
		}
	}
}

/**
 * Get the API docs folder for environment
 * @param stage environment
 * @return  folder name
 */
def getApiDocsFolder(stage) {
	return configLoader.AWS.S3.API_DOC
}

/**
 * Get the environment key
 * @param stage environment
 * @return  environment key to be represented in the event
 */
def getEnvKey(stage) {
	if (stage == 'dev') {
		return "DEV"
	} else if (stage == 'stg') {
		return "STG"
	} else if (stage == 'prod') {
		return "PROD"
	}
}

/**
 * Get the resource Path from domain and service name.
 * @return  formed resource path string
 */
def getResourcePath() {
	def basePath
	def pathInfo
	def resourcepath
	try {
		dir("swagger") {
			def swaggerStr = readFile('swagger.json').trim()
			def swaggerJsonObj = parseJsonMap(swaggerStr)
			basePath = swaggerJsonObj.basePath
			def keys = swaggerJsonObj.paths.keySet()
			for (_p in keys) {
				pathInfo = _p
				break
			}
		}
		resourcepath = (basePath + "/" + pathInfo).replaceAll("//", "/")
		return resourcepath
	} catch (ex) {
		handleFailureEvent(ex.getMessage())
	}
}


def generateAssetInfo(environment){
	def assetInfo = [:]
	def s3Bucket
	def assetBase

	if (configLoader.JAZZ.BUCKET_PER_SERVICE == "true") {
		if (service_config['s3_bucket_name']) {
			s3Bucket = service_config['s3_bucket_name']
		} else {
			handleFailureEvent("Failed to get S3 bucket information from service catalog.")
		}
		assetBase = "${s3Bucket}"
		assetInfo['asset_name'] = "${s3Bucket}-${environment}"
	} else {
		if (environment.endsWith("-dev")) {
			s3Bucket = utilModule.getBucket('dev')
		} else {
			s3Bucket = utilModule.getBucket(environment)
		}
		assetBase = "${s3Bucket}/${service_config['domain']}-${service_config['service']}"
		assetInfo['asset_name'] = "${s3Bucket}-${service_config['domain']}-${service_config['service']}-${environment}"
	}

	assetInfo['s3Bucket'] = s3Bucket
	assetInfo['asset_info'] = "${assetBase}/${environment}"
	assetInfo['folder_name'] = "${assetBase}"

	return assetInfo
}

/**
 * Undeploy the website. Delete the web folder from S3 bucket
 * @param stage
 * @return
 */
def unDeployWebsite(stage) {
	echo "unDeployWebsite::${service_config['service']}::${service_config['domain']} ${stage}"

	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"
			sh "aws configure set preview.cloudfront true"

			def assetInfo = generateAssetInfo(stage)
			echo "BucketName :: ${assetInfo['s3Bucket']} and BucketURL :: ${assetInfo['asset_info']}"
			def _exists = checkIfWebsiteExists(assetInfo['asset_info'])
			if (_exists) {
				cleanupS3BucketPolicy(stage, assetInfo)
				sh "aws s3 rm s3://${assetInfo['asset_info']} --recursive"
			}
			def isEmpty

			if (configLoader.JAZZ.BUCKET_PER_SERVICE == "true") {
				isEmpty = isBucketEmpty(assetInfo['folder_name'])
			} else {
				isEmpty = isBucketFolderEmpty(assetInfo['folder_name'])
			}

			if (_exists && isEmpty) {
				echo "Cleaning up service folder since service folder is empty:: ${assetInfo['asset_info']}"
				if (configLoader.JAZZ.BUCKET_PER_SERVICE == "true") {
					sh "aws s3 rb s3://${assetInfo['folder_name']} --force"
				} else {
					sh "aws s3 rm s3://${assetInfo['folder_name']} --recursive"
				}
			}
		} catch (ex) {
			handleFailureEvent(ex.getMessage())
		}
	}
}

/**
 * Check if the website folder existing in the S3 buckets for each environments
 * @param stage
 * @return  true/false
 */
def checkIfWebsiteExists(bucketUrl) {
	def status = true;
	try {
		sh "aws s3 ls s3://${bucketUrl}"
	} catch (ex) {
		echo "Bucket does not exist"
		status = false
	}
	return status
}

def isBucketEmpty(bucketName){
	def status = false;
	try {
		sh("aws s3api list-objects --bucket ${bucketName}  --output json --query '[length(Contents[])]'")
	} catch (ex) {
		echo "Bucket $bucketName is empty"
		status = true
	}
	return status
}

def isBucketFolderEmpty(bucketFolderName){
	def status = false;
	try {
		sh("aws s3 ls s3://${bucketFolderName}  --output json --query '[length(Contents[])]'")
	} catch (ex) {
		echo "Bucket $bucketFolderName is empty"
		status = true
	}
	return status
}

/**
 * Delete the the bucket policies related to the service folder
 * @param service
 * @param domain
 * @param stage
 * @return
 */
def cleanupS3BucketPolicy(stage, assetInfo) {
	echo "cleanupS3BucketPolicy called"

	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"
			sh "aws configure set preview.cloudfront true"
			def bucketPolicy = sh(
				script: "aws s3api get-bucket-policy --bucket ${assetInfo['s3Bucket']} --output json",
				returnStdout: true
			).trim()
			def policyObject = parseJsonMap(parseJsonMap(bucketPolicy).Policy)
			def policyObjectUpdated = [:]
			policyObjectUpdated.Version = policyObject.Version
			policyObjectUpdated.Id = policyObject.Id
			def statements = []
			for (items in policyObject.Statement) {
				if ((items.Sid != "list-${assetInfo['asset_name']}")) {
					def copy = [:]
					copy.putAll(items)
					statements.add(copy)
				}
			}

			echo "updated Policy : $statements"
			if (statements.size() > 0) {
				policyObjectUpdated.Statement = statements
				def policy_json = JsonOutput.toJson(policyObjectUpdated)
				updateBucketPolicy(policy_json, assetInfo['s3Bucket'])
			}
			resetCredentials()
		} catch (ex) {
			resetCredentials()
			if (ex.getMessage().indexOf("groovy.json.internal.LazyMap") < 0) {
				handleFailureEvent(ex.getMessage())
			}
		}
	}
}

/**
	Reset credentials
*/
def resetCredentials() {
	echo "resetting AWS credentials"
	sh "aws configure set profile.cloud-api.aws_access_key_id XXXXXXXXXXXXXXXXXXXXXXXXXX"
	sh "aws configure set profile.cloud-api.aws_access_key_id XXXXXXXXXXXXXXXXXXXXXX"
}

@NonCPS
def updateBucketPolicy(policy_json, bucketName){
	try {
		sh "aws s3api put-bucket-policy \
				--output json \
				--bucket "+ bucketName + " \
				--policy \'${policy_json}\'"
	} catch (e) {
		handleFailureEvent(ex.getMessage())
	}
}
/**
 * Delete the the cloud Front policies related to the service folder
 * @param service
 * @param domain
 * @param stage
 * @return
 */
def cleanupCloudFrontDistribution(stage) {
	withCredentials([
		[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
	]) {
		try {
			sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
			sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
			sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"
			sh "aws configure set preview.cloudfront true"

			def distributionID
			def _Etag

			distributionID = getDistributionId(stage)

			if (distributionID && distributionID != "") {
				def distributionConfig = getDistributionConfig(distributionID)
				_Etag = generateDistributionConfigForDisable(distributionConfig)
				_Etag = disableCloudFrontDistribution(distributionID, _Etag, "disable-cf-distribution-config.json", stage)
			}
		} catch (ex) {
			if ((ex.getMessage()).indexOf("getDistributionId Failed") > -1) {
				echo "Could not find a CloudFront distribution Id for service: ${service_config['service']} and environment $stage"
			} else {
				handleFailureEvent(ex.getMessage())
			}
		}
	}
}

/**
 * Get dist Id if exists
 *
 */
def getDistributionId(environment) {
	def distributionID
	try {
		def outputStr = listDistribution(environment)
		if (outputStr) {
			echo "### OutputStr for getting Distribution Id: $outputStr"
			def outputObj = new JsonSlurperClassic().parseText(outputStr)
			if (outputObj && outputObj[0].Id) {
				distributionID = outputObj[0].Id
			}
		}
		return distributionID
	} catch (ex) {
		return distributionID
	}
}

def listDistribution(environment){
	def outputStr = null
	def service = "${service_config['domain']}-${service_config['service']}"
	try {
		outputStr = sh(
			script: "aws cloudfront list-distributions \
				--output json \
				--query \"DistributionList.Items[?Origins.Items[?Id=='${configLoader.INSTANCE_PREFIX}-${environment}-static-website-origin-$service']].{Distribution:DomainName, Id:Id}\"",
			returnStdout: true
		)
		return outputStr
	} catch (ex) {
		return outputStr
	}
}

/**
 * Get and save the CloudFront distribution Config corresponding to the service
 * @param distributionID
 * @return
 */
def getDistributionConfig(distributionID) {
	def distributionConfig
	try {
		distributionConfig = sh(
			script: "aws cloudfront get-distribution-config \
						--output json --id "+ distributionID,
			returnStdout: true
		)
		return distributionConfig
	} catch (ex) {
		handleFailureEvent(ex.getMessage())
	}

}

/**
 * Generate Disable Distribution configuration
 * @param service
 * @param stage
 * @return
 */
def generateDistributionConfigForDisable(distributionConfig) {
	def distributionConfigObj
	def eTag
	try {
		if (distributionConfig) {
			distributionConfigObj = parseJsonMap(distributionConfig)
		}
		eTag = distributionConfigObj.ETag
		distributionConfigObj.DistributionConfig.Enabled = false
		def updatedCfg = JsonOutput.toJson(distributionConfigObj.DistributionConfig)
		echo "Updated configuration for disabling CloudFront... $updatedCfg"
		try {
			sh "echo \'$updatedCfg\' > disable-cf-distribution-config.json"
		} catch (ex) { }
		return eTag
	} catch (ex) {
		handleFailureEvent(ex.getMessage())
	}
}

/**
 * Disable Distribution configuration
 * @param distributionID
 * @param _Etag
 * @param configFile
 * @return
 */
def disableCloudFrontDistribution(distributionID, _Etag, configFile, stage) {
	def disableOutput
	def eTag
	try {
		disableOutput = sh(
			script: "aws cloudfront update-distribution \
						--output json \
						--id $distributionID \
						--distribution-config file://"+ configFile + " \
						--if-match $_Etag",
			returnStdout: true
		)
		echo "disableOutput... $disableOutput"
		if (disableOutput) {
			def disableConfigObj = new JsonSlurperClassic().parseText(disableOutput)
			eTag = disableConfigObj.ETag
		}
		echo "disable eTag: $eTag"
		return eTag
	} catch (ex) {
		handleFailureEvent(ex.getMessage())
	}
}

def LoadConfiguration() {
	def result = readFile('deployment-env.yml').trim()
	echo "result of yaml parsing....$result"
	def prop = [:]
	def resultList = result.tokenize("\n")

	// delete commented lines
	def cleanedList = []
	for (i in resultList) {
		if (i.toLowerCase().startsWith("#")) { } else {
			cleanedList.add(i)
		}
	}
	// echo "result of yaml parsing after clean up....$cleanedList"
	for (item in cleanedList) {
		// Clean up to avoid issues with more ":" in the values
		item = item.replaceAll(" ", "").replaceFirst(":", "#");
		def eachItemList = item.tokenize("#")
		//handle empty values
		def value = null;
		if (eachItemList[1]) {
			value = eachItemList[1].trim();
		}

		if (eachItemList[0]) {
			prop.put(eachItemList[0].trim(), value)
		}

	}
	echo "Loaded configurations....$prop"
	return prop
}

def loadServerlessConfig() {

	def configPackURL = scmModule.getCoreRepoCloneUrl("serverless-config-pack")
	echo "loadServerlessConfig:: ${service_config['providerRuntime']}::" + configPackURL

	dir('_config') {
		checkout([$class: 'GitSCM', branches: [
			[name: '*/master']
		], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
				[credentialsId: configLoader.REPOSITORY.CREDENTIAL_ID, url: configPackURL]
			]])
	}
	if (service_config['providerRuntime'].indexOf("nodejs") > -1) {
		sh "cp _config/serverless-nodejs.yml ./serverless.yml"
	} else if (service_config['providerRuntime'].indexOf("java") > -1) {
		sh "cp _config/serverless-java.yml ./serverless.yml"
	} else if (service_config['providerRuntime'].indexOf("python") > -1) {
		sh "cp _config/serverless-python.yml ./serverless.yml"
	}

  if (service_config['event_source_dynamodb'] || service_config['event_source_sqs']) {
    sh "cp _config/aws-events-policies/custom-policy.yml ./policyFile.yml"
  }

  if (!service_config['event_source_dynamodb'] || !service_config['event_source_sqs'] || !service_config['event_source_s3'] || !service_config['event_source_kinesis']) {
    removeEventResources ()
  }

}

def removeEventResources(){
   sh "sed -i -- '/#Start:resources/,/#End:resources/d' ./serverless.yml"
}


/**
 * For getting token to access catalog APIs.
 * Must be a service account which has access to all services
 */
def setCredentials() {
	def url = g_base_url + '/jazz/login'
	withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: g_svc_admin_cred_ID, passwordVariable: 'PWD', usernameVariable: 'UNAME']]) {
		echo "user name is $UNAME"
		def login_json = []
		login_json = [
			'username': UNAME,
			'password': PWD
		]
		def tokenJson_token = null
		def payload = JsonOutput.toJson(login_json)
		try {
			def token = sh(script: "curl --silent -X POST -k -v \
				-H \"Content-Type: application/json\" \
				 $url \
				-d \'${payload}\'", returnStdout: true).trim()

			def tokenJson = jsonParse(token)
			tokenJson_token = tokenJson.data.token

			return tokenJson_token
		}
		catch (e) {
			echo "Failed to fetch authentication details."
		}
	}
}

/**
* Send email to the recipient with the build status and any additional text content
* Supported build status values = STARTED, FAILED & COMPLETED
* @return
*/
def send_status_email(build_status) {

	def email_content = ''
	def body_subject = ''
	def body_text = ''
	def cc_email = ''
	def body_html = ''
	if (build_status == 'STARTED') {
		echo "email status started"
		body_subject = "Jazz Build Notification: Deletion STARTED for service: ${service_config['service']}"
	} else if (build_status == 'FAILED') {
		echo "email status failed"
		def build_url = env.BUILD_URL + 'console'
		body_subject = "Jazz Build Notification: Deletion FAILED for service: ${service_config['service']}"
		body_text = body_text + '\n\nFor more details, please click this link: ' + build_url
	} else if (build_status == 'COMPLETED') {
		body_subject = "Jazz Build Notification: Deletion COMPLETED successfully for service: ${service_config['service']}"
	} else {
		echo "Unsupported build status, nothing to email.."
		return
	}
	if (email_content != '') {
		body_text = body_text + '\n\n' + email_content
	}
	def fromStr = 'Jazz Admin <' + configLoader.JAZZ.ADMIN + '>'
	body = JsonOutput.toJson([
		from : fromStr,
		to : service_config['created_by'],
		subject : body_subject,
		text : body_text,
		cc : cc_email,
		html : body_html
	])
	try {
		def sendMail = sh(script: "curl -X POST \
					 ${g_base_url}/jazz/email \
					 -k -v -H \"Authorization: $g_login_token\" \
					 -H \"Content-Type: application/json\" \
					 -d \'${body}\'", returnStdout: true).trim()
		def responseJSON = parseJsonMap(sendMail)
		if (responseJSON.data) {
			echo "successfully sent e-mail to $email_id"
		} else {
			echo "exception occured while sending e-mail: $responseJSON"
		}
	} catch (e) {
		echo "Failed while sending build status notification"
	}
}

/*
* Load build modules
*/
def loadBuildModules(buildModuleUrl){
	dir('build_modules') {
		checkout([$class: 'GitSCM', branches: [
			[name: '*/master']
		], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
				[credentialsId: repo_credential_id, url: buildModuleUrl]
			]])

		def resultJsonString = readFile("jazz-installer-vars.json")
		configModule = load "config-loader.groovy"
		configLoader = configModule.initialize(resultJsonString)
		echo "config loader loaded successfully."

		scmModule = load "scm-module.groovy"
		scmModule.initialize(configLoader)
		echo "SCM module loaded successfully."

		events = load "events-module.groovy"
		echo "Event module loaded successfully."

		serviceMetadataLoader = load "service-metadata-loader.groovy"
		serviceMetadataLoader.initialize(configLoader)
		echo "Service metadata loader module loaded successfully."

		utilModule = load "utility-loader.groovy"
		utilModule.initialize(configLoader)
		echo "Util module loaded successfully."

		environmentMetadataLoader = load "environment-deployment-metadata-loader.groovy"
		echo "Environment metadata loader module loaded successfully."

		sonarModule = load "sonar-module.groovy"
		echo "Sonar module loaded successfully."

    lambdaEvents = load "aws-lambda-events-module.groovy"
    lambdaEvents.initialize(configLoader, utilModule)
    echo "Lambda event module loaded successfully."

	}
}

def getBuildModuleUrl() {
	if (scm_type && scm_type != "bitbucket") {
		// right now only bitbucket has this additional tag scm in its git clone path
		return "http://${repo_base}/${repo_core}/jazz-build-module.git"
	} else {
		return "http://${repo_base}/scm/${repo_core}/jazz-build-module.git"
	}
}

def getStage(environment){
	if (environment.equals('dev') || environment.endsWith('-dev')) {
		environment = 'dev'
	}
	return environment
}

def setDeploymentBucketName(stage, environment){
	sh "sed -i -- 's/oaf-apis-deployment-bucket-" + stage + "/oaf-apis-deployment-bucket-" + environment + "/g' serverless.yml"
}

@NonCPS
def jsonParse(jsonString) {
	def nonLazyMap = new groovy.json.JsonSlurperClassic().parseText(jsonString)
	return nonLazyMap
}

@NonCPS
def parseJsonMap(jsonString) {
	def jsonSlurper = new groovy.json.JsonSlurperClassic()
	def jsonObj = jsonSlurper.parseText(jsonString)
	def lazyMap = [:]
	lazyMap.putAll(jsonObj)
	return lazyMap
}
