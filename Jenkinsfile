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
	
	parameters {
		booleanParam(name: "parallel", description: 'Enable Prallel Execution', defaultValue: true)
		booleanParam(name: "RELEASE", description: 'Generate Release', defaultValue: false)
		choice(name: "DEPLOY_TO", description: 'Deply To Environment', choices: ["", "DEV", "QA2", "STAGE", "QA", "INT", "PRE", "PROD"])
		[$class: 'ChoiceParameter', 
		    choiceType: 'PT_SINGLE_SELECT', 
		    description: 'Select the Environemnt from the Dropdown List', 
		    filterLength: 1, 
		    filterable: false, 
		    name: 'Env', 
		    script: [
			$class: 'GroovyScript', 
			fallbackScript: [
			    classpath: [], 
			    sandbox: false, 
			    script: 
				"return['Could not get The environemnts']"
			], 
			script: [
			    classpath: [], 
			    sandbox: false, 
			    script: 
				"return['dev','stage','prod']"
			]
		    ]
		]
		[$class: 'CascadeChoiceParameter', 
		    choiceType: 'PT_SINGLE_SELECT', 
		    description: 'Select the AMI from the Dropdown List',
		    name: 'AMI List', 
		    referencedParameters: 'Env', 
		    script: 
			[$class: 'GroovyScript', 
			fallbackScript: [
				classpath: [], 
				sandbox: false, 
				script: "return['Could not get Environment from Env Param']"
				], 
			script: [
				classpath: [], 
				sandbox: false, 
				script: '''
				if (Env.equals("dev")){
				    return["ami-sd2345sd", "ami-asdf245sdf", "ami-asdf3245sd"]
				}
				else if(Env.equals("stage")){
				    return["ami-sd34sdf", "ami-sdf345sdc", "ami-sdf34sdf"]
				}
				else if(Env.equals("prod")){
				    return["ami-sdf34sdf", "ami-sdf34ds", "ami-sdf3sf3"]
				}
				'''
			    ] 
		    ]
		]
		[$class: 'DynamicReferenceParameter', 
		    choiceType: 'ET_ORDERED_LIST', 
		    description: 'Select the  AMI based on the following infomration', 
		    name: 'Image Information', 
		    referencedParameters: 'Env', 
		    script: 
			[$class: 'GroovyScript', 
			script: 'return["Could not get AMi Information"]', 
			script: [
			    script: '''
				    if (Env.equals("dev")){
					return["ami-sd2345sd:  AMI with Java", "ami-asdf245sdf: AMI with Python", "ami-asdf3245sd: AMI with Groovy"]
				    }
				    else if(Env.equals("stage")){
					return["ami-sd34sdf:  AMI with Java", "ami-sdf345sdc: AMI with Python", "ami-sdf34sdf: AMI with Groovy"]
				    }
				    else if(Env.equals("prod")){
					return["ami-sdf34sdf:  AMI with Java", "ami-sdf34ds: AMI with Python", "ami-sdf3sf3: AMI with Groovy"]
				    }
				    '''
				]
			]
		]
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
		stage('clean modules') {
			when {
				expression {
					return buildAll || affectedModules.size() > 0
				}
			}
			steps {
				script {
					if(buildAll){
						// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
						impactedModules = sh(returnStdout: true, script: "mvn clean -B -DskipTests -Pbuild -T 5 | grep com.demo | awk -F \":| \" '{print \$4}'").trim().split()
					} else {
						//remove duplicate items and separate them using "," delimeter
						affectedList = affectedModules.unique().join(",")
						// -T 5 means we can build modules in parallel using 5 Threads, we can scale this
						impactedModules = sh(returnStdout: true, script: "mvn clean -B  -pl ${affectedList} -amd -DskipTests -Pbuild -T 5 | grep com.demo | awk -F \":| \" '{print \$4}'").trim().split()
					}
					println("impactedModules : " + impactedModules)
				    	println("impactedModules.size() : " + impactedModules.size())
					println("Cleaned impacted modules")
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
						sh "mvn ${goal} -B -pl ${affectedList} -amd -DskipTests -Pbuild -T 5"
					} else {
						bat "mvn ${goal} -B -pl ${affectedList} -amd -DskipTests -Pbuild -T 5"	
					}
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
					// Set up List<Map<String,Closure>> describing the builds
					def unitTestCmd = "mvn test -B -T 5 -PunitTest"
					unitTestStages = prepareDynamicStages("Unit Test", unitTestCmd, impactedModules)
					println("unitTestStages : " + unitTestStages)
					// Set up List<Map<String,Closure>> describing the builds
					def integrationTestCmd = "mvn test -B -T 5 -PintegrationTest"
					integrationTestStages = prepareDynamicStages("Integration Test", integrationTestCmd, impactedModules)
					println("integrationTestStages : " + integrationTestStages)
					// Set up List<Map<String,Closure>> describing the builds
					def deployITCmd = "mvn test -B -T 5 -PdeployIT"
					deployITStages = prepareDynamicStages("Deploy IT", deployITCmd, impactedModules)
					println("deployITStages : " + deployITStages)
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
					for (unitTests in unitTestStages) {
					    if (params.parallel) {
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
		stage("verify integration test") {
			when {
				expression {
					return integrationTestStages.size() > 0
				}
			}
			steps {
				script {
					for (integrationTests in integrationTestStages) {
					    if (params.parallel) {
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
					    if (params.parallel) {
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
