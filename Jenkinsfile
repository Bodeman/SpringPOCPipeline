@Library('utilities') _

pipeline {
    agent any
    environment { 
		//try {
			//Load parameters
		//	properties([parameters([
		//		string(name: 'mvnHome', defaultValue: "tool 'maven Config'", description: 'Location of maven'),
		//		string(name: 'workingGitURL', defaultValue: 'https://github.com/Bodeman/Dev.git', description: 'Repository for git project. Typically https://github.com/[owner]/[projectname].git'),
		//		string(name: 'workingBranch', defaultValue: 'errorTest', description: 'Branch of repository to build')
		//	])])
		
		//} catch(err) {
			//could not load variables
		//	echo 'Could not load variables as parameters.'
		//}
		
        mvnHome = tool 'Maven_Config' 
		
		//Toggle between environment configurations using this value. 0 = CAMMIS, 1 = Bode
		Environment_Config = 1
		
		//If the Jenkins master is windows || Linux
		Jenkins_Master = "Windows"
		
		// GitHub setup
		workingGitURL = ''
		workingBranch= ''
		
		//POM file locations for Maven
		workingPOM = ''
		
		// Jenkins setup 
		workingJob= 'TestPipeline'
		workingProject= 'SpringPOC'
		workingJenkinsDir= ''
		
		//Sonar configurations
		SonarHost='http://localhost:9000/'
		SonarScanner=''
		
		// AWS code Deploy setup
		AWSCDapplicationName= 'SpringPOC'
		AWSCDDeploymentGroupName= 'SpringPOCDG'
		AWSCDSubDirectory= 'SpringPOC'
		
		// Jira project setup
		workingJiraProject ='PTP'
		
		//Added variables for shared libraries
		loglevel = 'WARNING'
		notify_channel = 'CONSOLE'
		
		//Early abort variable
		continueBuild = true
		
		//Disable compile set to TRUE
		Compile_Disable = true
    }
    stages {
    	stage('Preparation') {
			steps {
				script {
					try {
					workingGitURL = workingconfigs.seturl Environment_Config
					workingBranch = workingconfigs.setbranch Environment_Config
					workingPOM = workingconfigs.setPOM Environment_Config
					workingJenkinsDir = workingconfigs.setJenkinsDir Environment_Config
					logger "${loglevel}", "DEBUG", "workingGitURL = ${workingGitURL}"
					logger "${loglevel}", "DEBUG", "workingBranch = ${workingBranch}"
					logger "${loglevel}", "DEBUG", "workingPOM = ${workingPOM}"
					logger "${loglevel}", "DEBUG", "workingJenkinsDir = ${workingJenkinsDir}"
					pullproject workingGitURL, workingBranch, continueBuild
					}
					catch (e) {
						logger "${loglevel}", "ERROR", ": ${e}"}
					}
				}
		}
		stage('Starting Build') {
            steps {
				script{
					try{
					if(!continueBuild) {
						currentBuild.result = 'ABORTED'
						error('Stopping early due to critical failure.')
					}
					notifications "${notify_channel}", "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
					}
					catch (e) {
						logger "${loglevel}", "ERROR", "Error[${e}]"
					}
				}
			}
		} 
       stage('Build') {
            steps {
				script {
				//TODO: diable the calls to CAMMIS infrastructure
//					input message: 'Approve deployment?'
					build Jenkins_Master, mvnHome, workingPOM, Compile_Disable, loglevel
				}
            }
            post {
            failure {
                    logger "${loglevel}", "WARN", "Build Stage failed"

                }
			always {
                    logger "${loglevel}", "DEBUG", "Build Stage always"

                }
			} 
        } 
		
		stage('SonarQube analysis') { 
    		steps { 
				withSonarQubeEnv('SonarQubeServer') {
					sonar Jenkins_Master, SonarHost
				}
			}
			post {
                always {
                    logger "${loglevel}", "DEBUG", "SonarQube Analysis  Done"
                }
				failure {
                    logger "${loglevel}", "WARN", "SonarQube Analysis Failed"
					//script {
						//logger "${loglevel}", "WARN", "AWS Code Deploy failed"
						//testIssue = [fields: [ project: [key: "${workingJiraProject}"],
						//			summary: 'Jenkins Build Failure.',
						//			description: "Jenkins Build Failed - SonarQube Analysis Failed -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
						//			priority: [name: 'Highest'],
						//			issuetype: [name: 'Bug']]]

						//response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						//echo response.successful.toString()
						//echo response.data.toString()
						
						//slackSend (color: '#FFFF00', message: "Failed: Job - SonarQube Analysis Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
					//}
				}
				success {
					logger "${loglevel}", "INFO", "SonarQube Analysis ran"
				}	
			}
		} 
    	stage('SonarQube Quality Gate') { 
			steps {
				node('master'){ 
					script {
						timeout(time: 1, unit: 'HOURS') { 
							echo '************ Inside Quality Gate'
							qualityGate = waitForQualityGate() 
							echo qualityGate.status.toString() 
							if (qualityGate.status != 'OK') {
							
								testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Sonar Quality Gate Failure -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

								response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

								echo response.successful.toString()
								echo response.data.toString()
						
								slackSend (color: '#FFFF00', message: "Failed: Job - Sonar Quality Gate Failure '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
								error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
							}
						}
					}
				}
			}
			post {
                always {
                    echo 'SonarQube Quality Gate  Done'
                }
				failure {
					echo 'SonarQube Quality Gate  failure'
				}
				success {
					echo 'SonarQube Quality Gate Success'
				}	
			}
		}
		stage('Unit Test Report') {   
            steps {
				junit '**/target/surefire-reports/*.xml'
	    	}     
        	post {
				always {
                 	echo 'always'
                }
				changed {
		 			echo 'change'
				}
				aborted {
					echo 'aborted'
				}
				failure {
					echo 'failure'
				}
				success {
					echo 'success'
				}
				unstable {
					echo 'unstable'
				}
            }
        } 
		stage('Code Coverage Report') {   
            steps {
				cobertura autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/target/site/cobertura/coverage.xml', conditionalCoverageTargets: '70, 0, 0', failUnhealthy: false, failUnstable: false, lineCoverageTargets: '80, 0, 0', maxNumberOfBuilds: 0, methodCoverageTargets: '80, 0, 0', onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false
	    	}
			post {
                always {
                    echo 'Code Coverage Report  Done'
                }
				failure {
					echo 'Code Coverage Report  failure'
				}
				success {
					echo 'Code Coverage Report Success'
				}	
			}
		} 
		stage('Nexus Release Upload') {   
			when {
                // check if branch is master
                branch 'master'
            }
			steps {
				
				step([$class: 'NexusPublisherBuildStep', 
						nexusInstanceId: 'NexusDemoServer', nexusRepositoryId: 'releases', packages: [[$class: 'MavenPackage', mavenAssetList: [[classifier: '', extension: '', filePath: '/var/lib/jenkins/workspace/SpringPOC/SpringPOC/target/springpoc-1.0.0-BUILD-SNAPSHOT.war']], 
						mavenCoordinate: [artifactId: 'SpringPOC-war', groupId: 'CA-MMIS.jenkins.ci."${workingProject}"', packaging: 'war', version: '${BUILD_NUMBER}']]]])
			}
			post {
                always {
                   echo 'Nexus Nexus Release Upload  Done'
                }
				failure {
					echo 'Nexus Nexus Release Upload failure'
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Nexus Upload Failed-  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Nexus Upload Failed Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")						
					}
				}
				success {
					echo 'Nexus Nexus Release Upload Success'
				}	
			}
		} 
		stage('Maven Nexus Deploy') {
			steps {
				sh "'${mvnHome}/bin/mvn' -X -B --file '${workingPOM}' -Dintegration-tests.skip=true deploy"
            }
			post {
                always {
                   echo 'Maven Nexus Deploy  Done'
                }
				failure {
					echo 'Maven Nexus Deploy  failure'
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Nexus Depolyment Failed -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - AWS Code Deploy Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
					}
				}
				success {
					echo 'Maven Nexus Deploy Success'
				}	
			}
		} 
		stage('Jira Update Issues') {
			steps {
				echo 'Jira Update Issues'
				
				step([$class: 'hudson.plugins.jira.JiraIssueUpdater', 
					issueSelector: [$class: 'hudson.plugins.jira.selector.DefaultIssueSelector'], 
					scm: [$class: 'GitSCM', branches: [[name: '*/"${workingBranch}"']], 
					userRemoteConfigs: [[url: "${workingGitURL}"]]]])
			}
			post {
                always {
					echo 'Jira Update Issues'
                }	
				failure {
					echo 'Jira Update Issues  failure'
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Jira Update Issues Failed-  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Jira Update Issues '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
					}
				}
				success {
					echo 'Jira Update Issues Success'
				}
			}		
		} 
		stage('Security Dependency Check') {
			steps {
				echo 'Security Dependency Check'
				dependencyCheckAnalyzer datadir: '', hintsFile: '', includeCsvReports: false, includeHtmlReports: false, includeJsonReports: false, includeVulnReports: false, isAutoupdateDisabled: false, outdir: '', scanpath: '', skipOnScmChange: false, skipOnUpstreamChange: false, suppressionFile: '', zipExtensions: ''
			}
			post {
                always {
					echo 'Security Dependency Check'
                }	
				failure {
					script {
						echo 'Security Dependency Check  failure'
						testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Security Dependency Check -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Security Dependency Check '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
					}
				}
				success {
					echo 'Security Dependency Check Success'
				}
			}
		}
 		stage('Security Dependency Publisher') {
			steps {
				echo 'Security Dependency Check'
				dependencyCheckPublisher canComputeNew: false, defaultEncoding: '', healthy: '', pattern: '', unHealthy: ''
			}
			post {
                always {
					echo 'Security Dependency Publisher'
                }	
				failure {
					script {
						echo 'Security Dependency Publisher  failure'
						testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Security Dependency Check -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Security Dependency Publisher '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
					}
				}
				success {
					echo 'Security Dependency Publisher Success'
				}
			}
		} 
		stage('AWS Code Deploy') {
			steps {
				echo 'AWS Code Deploy'
				echo "env.AWS_ACCESS_KEY_ID :" + env.AWS_ACCESS_KEY_ID
				echo "env.AWS_SECRET_ACCESS_KEY :" + env.AWS_SECRET_ACCESS_KEY
				
				step([$class: 'AWSCodeDeployPublisher', 
						applicationName: "${AWSCDapplicationName}",
						awsAccessKey: env.AWS_ACCESS_KEY_ID,
						awsSecretKey: env.AWS_SECRET_ACCESS_KEY, 
						credentials: 'awsAccessKey', 
						deploymentConfig: 'CodeDeployDefault.OneAtATime', 
						deploymentGroupAppspec: false, 
						deploymentGroupName: "${AWSCDDeploymentGroupName}", 
						deploymentMethod: 'deploy', 
						excludes: '', 
						iamRoleArn: '', 
						includes: '**', 
						pollingFreqSec: 15, 
						pollingTimeoutSec: 300, 
						proxyHost: '', 
						proxyPort: 0, 
						region: 'us-gov-west-1', 
						s3bucket: 'codedeploybucket', 
						s3prefix: '', 
						subdirectory: "${AWSCDSubDirectory}",
						versionFileName: '', 
						waitForCompletion: true])
			
			}
			post {
                always {
					echo 'AWS Code Deploy'
                }	
				failure {
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: "${workingJiraProject}"],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - AWS Code Deploy Failed-  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - AWS Code Deploy Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
					}
				}
				success {
					echo 'AWS Code Deploy Success'
					slackSend (color: '#00FF00', message: "Code Deploy SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
				}
			}		
		} 
	}
}
