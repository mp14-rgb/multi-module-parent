
import org.jenkinsci.plugins.workflow.flow.FlowExecution;
import org.jenkinsci.plugins.workflow.job.WorkflowRun;
import io.jenkins.blueocean.rest.impl.pipeline.PipelineNodeGraphVisitor;

import org.jenkinsci.plugins.workflow.graph.FlowGraphWalker;
import org.jenkinsci.plugins.workflow.graph.FlowNode;

def buildAll = false
def impactedModules = []
def affectedModules = []
def affectedList
def goal = "package"

// could use eg. params.parallel build parameter to choose parallel/serial 
def runParallel = true

def unitTestStages = []
def integrationTestStages = []
def deployITStages = []

pipeline {
	
	agent any
	//disable concurrent build to avoid race conditions and to save resources
	options {
		preserveStashes(buildCount: 5)
        	disableConcurrentBuilds()
    	}
	
	parameters {
		booleanParam(name: "RELEASE", description: 'Generate Release', defaultValue: false)
		choice(name: "DEPLOY_TO", description: 'Deply To Environment', choices: ["", "DEV", "QA2", "STAGE", "QA", "INT", "PRE", "PROD"])
	}

	//load tools - these should be configured in jenkins global tool configuration
	 tools { 
        	maven 'Maven 3.6.3' 
        	jdk 'JDK8' 
    	}
	
	stages {
		stage("print env"){
			steps {
				script {
					sh 'printenv'
					println(getStageFlowLogUrl())
				}
			}
		}
		//this stage will get all the files that were modified
		stage("get diff") {
			steps {
				script {
					pullRequest.createStatus(status: 'pending',
                         context: 'Calculate Diff',
                         description: 'Pending.... get diff',
                         targetUrl: "${env.RUN_DISPLAY_URL}?page=pipeline/"+getStageFlowLogUrl())
					sh 'printenv'
					def changes = []
					
					if(env.CHANGE_ID) { //check if triggered via Pull Request
						echo "Pull Request Trigger"
						//get changes via git diff so we can know which module should be built
						if (isUnix()) {
							changes = sh(returnStdout: true, script: "git --no-pager diff origin/${CHANGE_TARGET} --name-only").trim().split()
						}
						else {
							changes = bat(returnStdout: true, script: "git --no-pager diff origin/${CHANGE_TARGET} --name-only").trim().split()
						}						
						//use compile goal instead of package if the trigger came from Pull Request. we dont want to package our module for every pull request
						goal = "compile"
					} else if(currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause').size() > 0) { //check if triggered via User 'Build Now'
						echo "User Trigger"
						buildAll = true						
						//use package goal if the triggered by User.
						goal = "package"
					} else { //defaults to Push Trigger
						echo "Push Trigger"
						//get changes via changelogs so we can know which module should be built
						def changeLogSets = currentBuild.changeSets
						for (int i = 0; i < changeLogSets.size(); i++) {
							def entries = changeLogSets[i].items
							for (int j = 0; j < entries.length; j++) {
								def entry = entries[j]
								def files = new ArrayList(entry.affectedFiles)
								for (int k = 0; k < files.size(); k++) {
									def file = files[k]
									changes.add(file.path)
								}
							}	
						}						
					}
					//iterate through changes
					changes.each {c -> 
						if(c.contains("common") || c == "pom.xml") { //if changes includes parent pom.xml and common module, we should build all modules
							affectedModules = []
							buildAll = true
							return
						}else {
							if(c.indexOf("/") > 1) { //filter all affected module. indexOf("/") means the file is inside a subfolder (module)
								affectedModules.add(c.substring(0,c.indexOf("/")))
							}
							
						}
					}
					println("Changes : " + changes)
					println("affectedModules : " + affectedModules)
					def testMap = [:]
					testMap.put("affectedModules", affectedModules)
					writeJSON(file: 'affectedModules.json', json: testMap, pretty: 4)
					def filedata = readJSON file:'affectedModules.json'
    					println(filedata)
					stash includes: 'affectedModules.json', name: 'affectedModules'
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Done.... get diff',
                         targetUrl: "${env.JOB_URL}")
				}
			}
		}
		stage('clean modules') {
			when {
				expression {
					sh 'printenv'
					if(affectedModules.size() == 0) {
						unstash 'affectedModules'
						def filedata = readJSON file:'affectedModules.json'
						println(filedata)
						affectedModules = filedata.get('affectedModules').toArray()
						println("affectedModules : " + affectedModules)
						
					}
					return buildAll || affectedModules.size() > 0
				}
			}
			steps {
				script {
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Pending.... clean modules',
                         targetUrl: "${env.JOB_URL}")
					
					if(buildAll){
						// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
						impactedModules = sh(returnStdout: true, script: "mvn clean -B -T 5 | grep com.demo | awk -F \":| \" '{print \$4}'").trim().split()
					} else {
						//remove duplicate items and separate them using "," delimeter
						affectedList = affectedModules.unique().join(",")
						// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
						impactedModules = sh(returnStdout: true, script: "mvn clean -B  -pl ${affectedList} -amd -T 5 | grep com.demo | awk -F \":| \" '{print \$4}'").trim().split()
					}
					println("impactedModules : " + impactedModules)
				    	println("impactedModules.size() : " + impactedModules.size())
					println("Cleaned impacted modules")
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Done.... cleaned modules',
                         targetUrl: "${env.JOB_URL}")
				}
			}
		 }
		//this stage will build all if the flag buildAll = true
		stage("compile all") {
			when {
				expression {
					return buildAll 
				}
			}
			steps {
				script {
					sh 'printenv'
					println(getStageFlowLogUrl())
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Pending.... compile all modules',
                         targetUrl: "${env.JOB_URL}")
					//goal = install | compile		
					// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
					if (isUnix()) {
						sh "mvn compile test-compile -B -T 5 -nsu"
					} else {
						bat "mvn compile test-compile -B -T 5 -nsu"
					}
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Done.... complied all modules',
                         targetUrl: "${env.JOB_URL}")
				}

			}
		}
		//this stage will build the affected modules only if affectedModules.size() > 0
		stage("compile modules") {
			when {
				expression {
					return affectedModules.size() > 0
				}
			}

			steps {
				script {
					sh 'printenv'
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Pending.... compile modules',
                         targetUrl: "${env.JOB_URL}")
					//remove duplicate items and separate them using "," delimeter
					affectedList = affectedModules.unique().join(",")
					//goal = install | compile		
					// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
					if (isUnix()) {
						sh "mvn compile test-compile -B -T 5 -nsu -pl ${affectedList} -amd"
					} else {
						bat "mvn compile test-compile -B -T 5 -nsu -pl ${affectedList} -amd"	
					}
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Done.... compile modules',
                         targetUrl: "${env.JOB_URL}")
				}

			}
		}
		stage('initialise test') {
			when {
				expression {
					return impactedModules.size() > 0
				}
			}
			steps {
				script {
					sh 'printenv'
					println(getStageFlowLogUrl())
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Pending.... initialise test',
                         targetUrl: "${env.JOB_URL}")
					// Set up List<Map<String,Closure>> describing the builds
					def unitTestCmd = "mvn test -B -T 5 -nsu -PunitTest"
					unitTestStages = prepareDynamicStages("Unit Test", unitTestCmd, impactedModules)
					println("unitTestStages : " + unitTestStages)
					// Set up List<Map<String,Closure>> describing the builds
					def integrationTestCmd = "mvn test -B -T 5 -nsu -PintegrationTest"
					integrationTestStages = prepareDynamicStages("Integration Test", integrationTestCmd, impactedModules)
					println("integrationTestStages : " + integrationTestStages)
					// Set up List<Map<String,Closure>> describing the builds
					def deployITCmd = "mvn test -B -T 5 -nsu -PdeployIT"
					deployITStages = prepareDynamicStages("Deploy IT", deployITCmd, impactedModules)
					println("deployITStages : " + deployITStages)
					
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Done.... initialise test',
                         targetUrl: "${env.JOB_URL}")
				}
			}
		  }
		//this stage will run unit test for the affected modules only if unitTestStages.size() > 0
		stage("verify unit test") {
			when {
				expression {
					return unitTestStages.size() > 0
				}
			}
			steps {
				script {
					println(getStageFlowLogUrl())
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Pending.... verify unit test',
                         targetUrl: "${env.JOB_URL}")
					for (unitTests in unitTestStages) {
					    if (runParallel) {
						    unitTests.failFast = true
						    parallel(unitTests)
					    } else {
					    	// run serially (nb. Map is unordered! )
					    	for (unitTest in unitTests.values()) {
							unitTest.call()
					    	}
					    }
					}
					
					pullRequest.createStatus(status: 'pending',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Done.... verify unit test',
                         targetUrl: "${env.JOB_URL}")
				}

			}
		}
		//this stage will run unit test for the affected modules only if integrationTestStages.size() > 0
		stage("verify integration test") {
			when {
				expression {
					return integrationTestStages.size() > 0
				}
			}
			steps {
				script {
					for (integrationTests in integrationTestStages) {
					    if (runParallel) {
					    	parallel(integrationTests)
					    } else {
					    	// run serially (nb. Map is unordered! )
					    	for (integrationTest in integrationTests.values()) {
							integrationTest.call()
					    	}
					    }
					}
				}

			}
		}
		//this stage will run unit test for the affected modules only if deployITStages.size() > 0
		stage("verify deploy IT") {
			when {
				expression {
					return deployITStages.size() > 0
				}
			}
			steps {
				script {
					for (deployITs in deployITStages) {
					    if (runParallel) {
					    	parallel(deployITs)
					    } else {
					    	// run serially (nb. Map is unordered! )
					    	for (deployIT in deployITs.values()) {
							deployIT.call()
					    	}
					    }
					}
				}

			}
		}
		//this stage will build all if the flag buildAll = true
		stage("package all") {
			when {
				expression {
					return buildAll
				}
			}
			steps {
				script {
					//goal = install | compile		
					// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
					if (isUnix()) {
						sh "mvn package -B -DskipTests -Pbuild -T 5 -nsu" 
					} else {
						bat "mvn package -B -DskipTests -Pbuild -T 5 -nsu"
					}
				}

			}
		}
		//this stage will build the affected modules only if affectedModules.size() > 0
		stage("package modules") {
			when {
				expression {
					return affectedModules.size() > 0 && goal == "package"
				}
			}

			steps {
				script {
					//remove duplicate items and separate them using "," delimeter
					affectedList = affectedModules.unique().join(",")
					//goal = install | compile		
					// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
					if (isUnix()) {
						sh "mvn package -B -pl ${affectedList} -amd -DskipTests -Pbuild -T 5 -nsu"
					} else {
						bat "mvn package -B -pl ${affectedList} -amd -DskipTests -Pbuild -T 5 -nsu"	
					}
				}

			}
		}
	}
	
	post {
		always {
			echo "Job : ${currentBuild.currentResult}"
		}
		success {
			script {
			// CHANGE_ID is set only for pull requests, so it is safe to access the pullRequest global variable
				if (env.CHANGE_ID) {
				    pullRequest.removeLabel('Build Failed')
					pullRequest.createStatus(status: 'success',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'all check passed',
                         targetUrl: "${env.JOB_URL}")
				}
			}
		}

		failure {
			script {
				// CHANGE_ID is set only for pull requests, so it is safe to access the pullRequest global variable
				if (env.CHANGE_ID) {
				    pullRequest.addLabel('Build Failed')
					pullRequest.createStatus(status: 'failure',
                         context: 'continuous-integration/jenkins/pr-merge',
                         description: 'Checks failed',
                         targetUrl: "${env.JOB_URL}")
					updateGithubCommitStatus(currentBuild)
				}
			 }
		}
	}
}
// Create List of dynamic stages for execution
def prepareDynamicStages(stageName, command, impactedModules) {
	def stageList = []
	def i=1
	def dynamicExecutionMap = [:]
	for (impactedModuleName in impactedModules ) {
		def childStageName = "${stageName} : ${impactedModuleName} ${i}"
		def moduleScript = "${command} -pl ${impactedModuleName}"
		dynamicExecutionMap.put(childStageName, prepareStage(childStageName, moduleScript))
		println("i : " + i)
		if(i % 5 == 0 || impactedModules.size() == i){
			def dynamicStageMap = [:]
			dynamicStageMap.putAll(dynamicExecutionMap)
			stageList.add(dynamicStageMap)
			dynamicExecutionMap = [:]
			i=0
		}
		i++
	}
	
	return stageList
}

def prepareStage(childStageName, command) {
	return {
		stage("Build stage:${childStageName}") {
			println("Building ${childStageName}")
			sh(script:"${command}", returnStatus:true)
		}
	}
}
def getRepoURL() {
  sh "git config --get remote.origin.url > .git/remote-url"
  return readFile(".git/remote-url").trim()
}

def getCommitSha() {
  sh "git rev-parse HEAD > .git/current-commit"
  return readFile(".git/current-commit").trim()
}

void updateGithubCommitStatus(build) {
  // workaround https://issues.jenkins-ci.org/browse/JENKINS-38674
  repoUrl = getRepoURL()
  commitSha = getCommitSha()

  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
    commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
    errorHandlers: [[$class: 'ShallowAnyErrorHandler']],
    statusResultSource: [
      $class: 'ConditionalStatusResultSource',
      results: [
        [$class: 'BetterThanOrEqualBuildResult', result: 'SUCCESS', state: 'SUCCESS', message: build.description],
        [$class: 'BetterThanOrEqualBuildResult', result: 'FAILURE', state: 'FAILURE', message: "failed pr check"],
        [$class: 'AnyBuildResult', state: 'FAILURE', message: 'Loophole']
      ]
    ]
  ])
}

def getStageFlowLogUrl(){
    def buildDescriptionResponse = httpRequest httpMode: 'GET', url: "${env.BUILD_URL}wfapi/describe", authentication: 'REST-API'
    def buildDescriptionJson = readJSON text: buildDescriptionResponse.content
    def stageDescriptionId = false
	println(buildDescriptionJson)
    buildDescriptionJson.stages.each{ it ->
        if (it.name == env.STAGE_NAME){
            stageDescriptionId = it.id
        }
    }
return stageDescriptionId
}
def getNodeWsUrl(flowNode = null) {
    if(!flowNode) {
        flowNode = getContext(org.jenkinsci.plugins.workflow.graph.FlowNode)
    }
    if(flowNode instanceof org.jenkinsci.plugins.workflow.cps.nodes.StepStartNode && flowNode.typeFunctionName == 'node') {
        // Could also check flowNode.typeDisplayFunction == 'Allocate node : Start'
        return "/${flowNode.url}ws/"
    }

    return flowNode.parents.findResult { getNodeWsUrl(it) }
}
