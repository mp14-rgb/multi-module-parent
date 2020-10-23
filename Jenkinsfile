def buildAll = false
def dependencyModules = []
def affectedModules = []
def affectedList
def goal = "package"
// Take the string and echo it.
def transformIntoStage(stageName) {
    // We need to wrap what we return in a Groovy closure, or else it's invoked
    // when this method is called, not when we pass it to parallel.
    // To do this, you need to wrap the code below in { }, and either return
    // that explicitly, or use { -> } syntax.
    return {
        stage (stageName) {
            wrap([$class: 'TimestamperBuildWrapper']) {
                echo "Element: $stageName"
                script {
                        echo "Element: $stageName"
                }
            }
        } // ts / node
        
    } // closure
}
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
					
				}
			}
		}
		stage("get module sequence"){
			steps {
				script {
					def depedencyTree = []
					def depedentModuleSequence = [:]
					def GROUP_ID = "com.demo"
					//get module depedency sequence via git diff so we can know which module should be built
					if (isUnix()) {
						depedencyTree = sh(returnStdout: true, script: "mvn dependency:tree | grep ${GROUP_ID}").trim().split()
					}
					else {
						depedencyTree = bat(returnStdout: true, script: "mvn dependency:tree | grep ${GROUP_ID}").trim().split()
					}
					//iterate through changes
					def moduleName = ""
					def moduleRef = "< " + GROUP_ID + ":"
					def dependentModuleRef = "] +- " + GROUP_ID + ":"
					depedencyTree.each {d -> 
						if(d.indexOf(moduleRef) > 0) { 
							moduleName = d.substring(d.indexOf(moduleRef)+moduleRef.length(),d.indexOf(" >"))
							depedentModuleSequence.put(moduleName, [])
						} else if(d.indexOf(dependentModuleRef) > 1) {
							def temp = d.substring(d.indexOf(dependentModuleRef)+dependentModuleRef.length(), d.length());
							def dependentModuleName = temp.substring(0, temp.indexOf(":"))
							println("Depedent Module : " + moduleName +" - " + dependentModuleName)
							depedentModuleSequence.get(moduleName).add(dependentModuleName)
						}
					}
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
						sh(returnStdout: true, script: "mvn clean ${goal} -B -DskipTests -Pbuild -T 5")
					} else {
						bat(returnStdout: true, script: "mvn clean ${goal} -B -DskipTests -Pbuild -T 5")
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
		stage('Dynamic Unit Test Stages') {
		    agent {node 'nodename'}
		    steps {
			script {
			    def stepsForParallel = [:]
			    for(int i=0; i < affectedModules.size(); i++) {
				def s = affectedModules[i]
				def stepName = "Untit Test : ${s}"
				stepsForParallel[stepName] = transformIntoStage(stepName)
			    }
			    stepsForParallel['failFast'] = false
			    parallel stepsForParallel
			}
		    }
		}
	}	
}
