properties([
		buildDiscarder(logRotator(numToKeepStr: '10')),
		pipelineTriggers([
				cron('@daily')
		]),
])

def SUCCESS = hudson.model.Result.SUCCESS.toString()
currentBuild.result = SUCCESS

try {
	stage('Check') {
		node('linux') {
			checkout scm
			try {
				withEnv(["JAVA_HOME=${tool 'jdk8'}"]) {
					sh './gradlew clean check --no-daemon --refresh-dependencies --stacktrace'
				}
			}
			catch (e) {
				currentBuild.result = 'FAILED: check'
				throw e
			}
		}
	}

	if (currentBuild.result == 'SUCCESS') {
		stage('Deploy Artifacts') {
			node('linux') {
				checkout scm
				try {
					withCredentials([file(credentialsId: 'spring-signing-secring.gpg', variable: 'SIGNING_KEYRING_FILE')]) {
						withCredentials([string(credentialsId: 'spring-gpg-passphrase', variable: 'SIGNING_PASSWORD')]) {
							withCredentials([usernamePassword(credentialsId: 'oss-s01-token', passwordVariable: 'OSSRH_S01_TOKEN_PASSWORD', usernameVariable: 'OSSRH_S01_TOKEN_USERNAME')]) {
								withCredentials([usernamePassword(credentialsId: '02bd1690-b54f-4c9f-819d-a77cb7a9822c', usernameVariable: 'ARTIFACTORY_USERNAME', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
									withEnv(["JAVA_HOME=${tool 'jdk8'}"]) {
										sh './gradlew deployArtifacts --no-daemon --refresh-dependencies --stacktrace --no-parallel -Psigning.secretKeyRingFile=$SIGNING_KEYRING_FILE -Psigning.keyId=$SPRING_SIGNING_KEYID -Psigning.password=$SIGNING_PASSWORD -PossrhTokenUsername=$OSSRH_S01_TOKEN_USERNAME -PossrhTokenPassword=$OSSRH_S01_TOKEN_PASSWORD -PartifactoryUsername=$ARTIFACTORY_USERNAME -PartifactoryPassword=$ARTIFACTORY_PASSWORD'
										sh './gradlew finalizeDeployArtifacts --no-daemon --refresh-dependencies --stacktrace --no-parallel -Psigning.secretKeyRingFile=$SIGNING_KEYRING_FILE -Psigning.keyId=$SPRING_SIGNING_KEYID -Psigning.password=$SIGNING_PASSWORD -PossrhTokenUsername=$OSSRH_S01_TOKEN_USERNAME -PossrhTokenPassword=$OSSRH_S01_TOKEN_PASSWORD -PartifactoryUsername=$ARTIFACTORY_USERNAME -PartifactoryPassword=$ARTIFACTORY_PASSWORD'
									}
								}
							}
						}
					}
				}
				catch (e) {
					currentBuild.result = 'FAILED: artifacts'
					throw e
				}
			}
		}
	}
}
finally {
	def buildStatus = currentBuild.result
	def buildNotSuccess = !SUCCESS.equals(buildStatus)
	def lastBuildNotSuccess = !SUCCESS.equals(currentBuild.previousBuild?.result)

	if (buildNotSuccess || lastBuildNotSuccess) {
		stage('Notify') {
			node {
				final def RECIPIENTS = [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]

				def subject = "${buildStatus}: Build ${env.JOB_NAME} ${env.BUILD_NUMBER} status is now ${buildStatus}"
				def details = "The build status changed to ${buildStatus}. For details see ${env.BUILD_URL}"

				emailext(
						subject: subject,
						body: details,
						recipientProviders: RECIPIENTS,
						to: "$SPRING_SESSION_TEAM_EMAILS"
				)
			}
		}
	}
}
