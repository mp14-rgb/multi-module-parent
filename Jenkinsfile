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
        	disableConcurrentBuilds()
    	}

	//load tools - these should be configured in jenkins global tool configuration
	 tools { 
        	maven 'Maven 3.6.3' 
        	jdk 'JDK8' 
    	}
	stages {
		//this stage will get all the files that were modified
		stage("get diff") {
			steps {
				script {
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
				}
			}
		}
		stage('clean') {
			when {
				expression {
					return buildAll || affectedModules.size() > 0
				}
			}
			steps {
				script {
					//if(buildAll){
						// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
						if (isUnix()) {
							impactedModules = sh(returnStdout: true, script: "mvn clean -B -DskipTests -Pbuild -T 5 | grep 'com.demo' | awk -F \':| \' '{print \$4}'").trim().split()
						} else {
							impactedModules = bat(returnStdout: true, script: "mvn clean -B -DskipTests -Pbuild -T 5 | grep 'com.demo' | awk -F \':| \' '{print \$4}'").trim().split()
						}
					//} else {
						//remove duplicate items and separate them using "," delimeter
						//affectedList = affectedModules.unique().join(",")
						// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
						//if (isUnix()) {
						//	impactedModules = sh(returnStdout: true, script: "mvn clean -B -pl ${affectedList} -am -DskipTests -Pbuild -T 5 | grep com.companyname.blogger").trim().split()
						//} else {
						//	impactedModules = bat(returnStdout: true, script: "mvn clean -B -pl ${affectedList} -am -DskipTests -Pbuild -T 5 | grep com.companyname.blogger").trim().split()
						//}
					//}
					println("impactedModules : " + impactedModules)
				    	println("Cleaned impacted modules")
				}
			}
		 }
		stage('Initialise') {
			when {
				expression {
					return impactedModules.size() > 0
				}
			}
			steps {
				script {
				    	// Set up List<Map<String,Closure>> describing the builds
					unitTestStages = prepareParallelStages("UnitTest", impactedModules)
					println("unitTestStages : " + unitTestStages)
					integrationTestStages = prepareParallelStages("IntegrationTest", impactedModules)
					println("integrationTestStages : " + integrationTestStages)
					deployITStages = prepareParallelStages("DeployIT", impactedModules)
					println("deployITStages : " + deployITStages)
				    	println("Initialised pipeline.")
				}
			}
		  }
		//this stage will build all if the flag buildAll = true
		stage("build all") {
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
						sh "mvn clean ${goal} -B -DskipTests -Pbuild -T 5" 
					} else {
						bat "mvn clean ${goal} -B -DskipTests -Pbuild -T 5"
					}
				}

			}
		}
		//this stage will build the affected modules only if affectedModules.size() > 0
		stage("build modules") {
			when {
				expression {
					return affectedModules.size() > 0
				}
			}

			steps {
				script {
					//remove duplicate items and separate them using "," delimeter
					affectedList = affectedModules.unique().join(",")
					//goal = install | compile		
					// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
					if (isUnix()) {
						sh "mvn clean ${goal} -B -pl ${affectedList} -am -DskipTests -Pbuild -T 5"
					} else {
						bat "mvn clean ${goal} -B -pl ${affectedList} -am -DskipTests -Pbuild -T 5"	
					}
				}

			}
		}
		//this stage will run unit test for the affected modules only if unitTestStages.size() > 0
		stage("UnitTest stages") {
			when {
				expression {
					return unitTestStages.size() > 0
				}
			}
			steps {
				script {
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
				}

			}
		}
		//this stage will run unit test for the affected modules only if integrationTestStages.size() > 0
		stage("IntegrationTest stages") {
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
		stage("deployITStages stages") {
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
	}	
}
// Create List of parallel stagesfor execution
def prepareParallelStages(stageName, affectedModuleList) {
	def stageList = []
	def i=0
	def parallelExecutionMap = [:]
	for (name in affectedModuleList ) {
		def n = "${stageName} : ${name} ${i}"
		parallelExecutionMap.put(n, prepareStage(n))
		if(i % 5 == 0){
			def parallelStageMap = [:]
			parallelStageMap.putAll(parallelExecutionMap)
			stageList.add(parallelStageMap)
			parallelExecutionMap = [:]
		}
		i++
	}
	
	return stageList
}

def prepareStage(String name) {
	return {
		stage("Build stage:${name}") {
			println("Building ${name}")
			sh(script:'sleep 5', returnStatus:true)
		}
	}
}
