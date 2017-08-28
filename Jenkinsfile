def projectProperties = [
	[$class: 'BuildDiscarderProperty',
		strategy: [$class: 'LogRotator', numToKeepStr: '5']],
	pipelineTriggers([cron('@daily')])
]
properties(projectProperties)

def SUCCESS = hudson.model.Result.SUCCESS.toString()
currentBuild.result = SUCCESS

try {
	stage('Check') {
		node {
			checkout scm
			try {
				sh "./gradlew clean check --no-daemon --stacktrace"
			}
			catch(Exception e) {
				currentBuild.result = 'FAILED: check'
				throw e
			}
		}
	}

	if (currentBuild.result == 'SUCCESS') {
		stage('Artifactory Deploy') {
			node {
				checkout scm
				withCredentials([usernamePassword(credentialsId: '02bd1690-b54f-4c9f-819d-a77cb7a9822c', usernameVariable: 'ARTIFACTORY_USERNAME', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
					sh "./gradlew artifactoryPublish -PartifactoryUsername=$ARTIFACTORY_USERNAME -PartifactoryPassword=$ARTIFACTORY_PASSWORD --no-daemon --stacktrace"
				}
			}
		}
	}
}
finally {
	def buildStatus = currentBuild.result
	def buildNotSuccess =  !SUCCESS.equals(buildStatus)
	def lastBuildNotSuccess = !SUCCESS.equals(currentBuild.previousBuild?.result)

	if (buildNotSuccess || lastBuildNotSuccess) {
		stage('Notifiy') {
			node {
				final def RECIPIENTS = [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]

				def subject = "${buildStatus}: Build ${env.JOB_NAME} ${env.BUILD_NUMBER} status is now ${buildStatus}"
				def details = """The build status changed to ${buildStatus}. For details see ${env.BUILD_URL}"""

				emailext (
					subject: subject,
					body: details,
					recipientProviders: RECIPIENTS,
					to: "$SPRING_SESSION_TEAM_EMAILS"
				)
			}
		}
	}
}
