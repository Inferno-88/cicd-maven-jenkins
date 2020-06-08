#!usr/bin/groovy

//Pipeline variables
def changedFiles = []
def affectedModulesArray = []
def affectedModulesString
def buildAllModules = false
def isNotPullRequest = true

//Maven Goals
def mavenBuildGoal = "install"
def mavenTestGoal = "jacoco:prepare-agent@preTest surefire:test jacoco:report@postTest"
def mavenIntegrationTestGoal = " jacoco:prepare-agent-integration@preIT failsafe:integration-test " +
        "jacoco:report-integration@postIT failsafe:verify"
def mavenDeployGoal = "jar:jar deploy:deploy"
def mavenDockerDeployGoal = "jib:build"

pipeline {

    environment {
        JAVA_TOOL_OPTIONS = "-Duser.home=/var/maven"
        DOCKER_MAVEN_ARGS = "-v /var/jenkins_home:/var/maven"
        DOCKER_MAVEN_IMAGE = "maven:3.6.3-openjdk-11-slim"
        MAVEN_THREAD_COUNT = 3
    }

    //disable concurrent build to avoid race conditions
    options {
        disableConcurrentBuilds()
    }

    //using specific agents for different stages
    agent none

    stages {

        //this stage finds current builds user and group
        stage("Find Jenkins User:Group") {
            agent any
            options {
                skipDefaultCheckout()
            }
            steps {
                script {
                    def user
                    def group
                    if (isUnix()) {
                        user = sh(returnStdout: true, script: 'id -u').trim()
                        group = sh(returnStdout: true, script: 'id -g').trim()
                    } else {
                        user = bat(returnStdout: true, script: 'id -u').trim()
                        group = bat(returnStdout: true, script: 'id -g').trim()
                    }
                    DOCKER_MAVEN_ARGS = DOCKER_MAVEN_ARGS + " -u ${user}:${group}"
                }
            }
        }

        //this stage prepares Docker container with Maven
        stage("Prepare Maven Agent") {
            agent {
                docker {
                    image "${DOCKER_REGISTRY_GROUP_URL}/${DOCKER_MAVEN_IMAGE}"
                    registryUrl "https://${DOCKER_REGISTRY_GROUP_URL}"
                    registryCredentialsId DOCKER_REGISTRY_CRED_ID
                    args DOCKER_MAVEN_ARGS
                }
            }
            stages {

                //this stage will find all the modules that were modified
                stage("Find Affected Modules") {
                    steps {
                        //build list of changed files
                        script {
                            if (env.CHANGE_ID) { //check if triggered by Pull Request
                                echo "Pull Request Trigger"
                                //get changed files from git diff
                                if (isUnix()) {
                                    changedFiles = sh(returnStdout: true, script: "git --no-pager diff origin/${CHANGE_TARGET} --name-only")
                                            .trim().split()
                                } else {
                                    changedFiles = bat(returnStdout: true, script: "git --no-pager diff origin/${CHANGE_TARGET} --name-only")
                                            .trim().split()
                                }
                                //only compile if triggered by Pull Request
                                mavenBuildGoal = 'compile'
                                isNotPullRequest = false
                            } else if (currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause').size() > 0) {
                                //check if triggered by User 'Build Now'
                                echo "User Trigger. Build all modules."
                                buildAllModules = true
                            } else { //defaults to Push Trigger
                                echo "Push Trigger"
                                //get changed files
                                def changeLogSets = currentBuild.changeSets
                                for (int i = 0; i < changeLogSets.size(); i++) {
                                    def entries = changeLogSets[i].items
                                    for (int j = 0; j < entries.length; j++) {
                                        def entry = entries[j]
                                        def files = new ArrayList(entry.affectedFiles)
                                        for (int k = 0; k < files.size(); k++) {
                                            def file = files[k]
                                            changedFiles.add(file.path)
                                        }
                                    }
                                }
                            }
                        }
                        //build list of affected modules
                        script {
                            for (int i = 0; i < changedFiles.size(); i++) {
                                def file = changedFiles[i]
                                if (file.contains("common") || file == "pom.xml") {
                                    //if changedFiles include parent pom.xml or common module then build all modules
                                    echo "*common* or root pom.xml has changed. Build all modules."
                                    buildAllModules = true
                                    break
                                } else if (file.indexOf("/") > 1) {
                                    //filter all affected modules. indexOf("/") means that file is inside a subfolder (module)
                                    affectedModulesArray.add(file.substring(0, file.indexOf("/")))
                                }
                            }
                            if (buildAllModules) {
                                echo "Affected modules: All"
                            } else if (!affectedModulesArray.isEmpty()) {
                                affectedModulesString = affectedModulesArray.unique().join(",")
                                echo "Affected modules: ${affectedModulesString}"
                            }
                        }
                    }
                }

                //this stage will build affected modules only
                stage("Build Affected Modules") {
                    when {
                        expression {
                            return !affectedModulesArray.isEmpty() && !buildAllModules
                        }
                    }
                    steps {
                        script {
                            if (isUnix()) {
                                sh "mvn clean ${mavenBuildGoal} -B -pl ${affectedModulesString} -am -DskipTests -T ${MAVEN_THREAD_COUNT}"
                            } else {
                                bat "mvn clean ${mavenBuildGoal} -B -pl ${affectedModulesString} -am -DskipTests -T ${MAVEN_THREAD_COUNT}"
                            }
                        }
                    }
                }

                //this stage will build all modules
                stage("Build All Modules") {
                    when {
                        expression {
                            return buildAllModules
                        }
                    }
                    steps {
                        script {
                            if (isUnix()) {
                                sh "mvn clean ${mavenBuildGoal} -B -DskipTests -T ${MAVEN_THREAD_COUNT}"
                            } else {
                                bat "mvn clean ${mavenBuildGoal} -B -DskipTests -T ${MAVEN_THREAD_COUNT}"
                            }
                        }
                    }
                }

                //this stage will run all unit tests
                stage("Unit Test") {
                    when {
                        expression {
                            return !affectedModulesArray.isEmpty() || buildAllModules
                        }
                    }
                    steps {
                        script {
                            if (buildAllModules) {
                                if (isUnix()) {
                                    sh "mvn ${mavenTestGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                } else {
                                    bat "mvn ${mavenTestGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                }
                            } else {
                                if (isUnix()) {
                                    sh "mvn ${mavenTestGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                } else {
                                    bat "mvn ${mavenTestGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                }
                            }
                        }
                    }
                }

                //this stage will run all integration tests
                stage("Integration Test") {
                    when {
                        expression {
                            return isNotPullRequest && (!affectedModulesArray.isEmpty() || buildAllModules)
                        }
                    }
                    steps {
                        script {
                            if (buildAllModules) {
                                if (isUnix()) {
                                    sh "mvn ${mavenIntegrationTestGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                } else {
                                    bat "mvn ${mavenIntegrationTestGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                }
                            } else {
                                if (isUnix()) {
                                    sh "mvn ${mavenIntegrationTestGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                } else {
                                    bat "mvn ${mavenIntegrationTestGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                }
                            }
                        }
                    }
                }

                //this stage will run maven repo & docker registry deploy in parallel
                stage("Publish") {
                    when {
                        expression {
                            return isNotPullRequest && (!affectedModulesArray.isEmpty() || buildAllModules)
                        }
                    }
                    failFast true
                    parallel {

                        //this stage will build and deploy images to Docker Registry
                        stage("Publish to Docker") {
                            steps {
                                script {
                                    if (buildAllModules) {
                                        if (isUnix()) {
                                            sh "mvn ${mavenDockerDeployGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                        } else {
                                            bat "mvn ${mavenDockerDeployGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                        }
                                    } else {
                                        if (isUnix()) {
                                            sh "mvn ${mavenDockerDeployGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                        } else {
                                            bat "mvn ${mavenDockerDeployGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                        }
                                    }
                                }
                            }
                        }

                        //this stage will deploy artifacts to Maven Repository
                        stage("Publish to Maven") {
                            steps {
                                script {
                                    if (buildAllModules) {
                                        if (isUnix()) {
                                            sh "mvn ${mavenDeployGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                        } else {
                                            bat "mvn ${mavenDeployGoal} -B -T ${MAVEN_THREAD_COUNT}"
                                        }
                                    } else {
                                        if (isUnix()) {
                                            sh "mvn ${mavenDeployGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                        } else {
                                            bat "mvn ${mavenDeployGoal} -B -pl ${affectedModulesString} -T ${MAVEN_THREAD_COUNT}"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        //this stage will deploy image on Kubernetes Cluster
        stage("Deploy") {
            agent any
            options {
                skipDefaultCheckout()
            }
            when {
                expression {
                    return isNotPullRequest && (!affectedModulesArray.isEmpty() || buildAllModules)
                }
            }
            steps {
                echo "Deploy somewhere..."
            }
        }
    }
}
