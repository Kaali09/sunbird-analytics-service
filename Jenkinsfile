node('build-slave') {
    try {
        String ANSI_GREEN = "\u001B[32m"
        String ANSI_NORMAL = "\u001B[0m"
        String ANSI_BOLD = "\u001B[1m"
        String ANSI_RED = "\u001B[31m"
        String ANSI_YELLOW = "\u001B[33m"
        ansiColor('xterm') {
            stage('Checkout') {
                cleanWs()
                if(params.github_release_tag == ""){
                    checkout scm
                    commit_hash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    branch_name = sh(script: 'git name-rev --name-only HEAD | rev | cut -d "/" -f1| rev', returnStdout: true).trim()
                    artifact_version = branch_name + "_" + commit_hash
            println(ANSI_BOLD + ANSI_YELLOW + "github_release_tag not specified, using the latest commit hash: " + commit_hash + ANSI_NORMAL)
                }
                else {
                    def scmVars = checkout scm
                    checkout scm: [$class: 'GitSCM', branches: [[name: "refs/tags/$params.github_release_tag"]],  userRemoteConfigs: [[url: scmVars.GIT_URL]]]
                    artifact_version = params.github_release_tag
            println(ANSI_BOLD + ANSI_YELLOW + "github_release_tag specified, building from github_release_tag: " + params.github_release_tag + ANSI_NORMAL)
                }
                echo "artifact_version: "+ artifact_version
                if (!env.hub_org) {
                    println(ANSI_BOLD + ANSI_RED + "Uh Oh! Please set a Jenkins environment variable named hub_org with value as registery/sunbidrded" + ANSI_NORMAL)
                    error 'Please resolve the errors and rerun..'
                } else
                    println(ANSI_BOLD + ANSI_GREEN + "Found environment variable named hub_org with value as: " + hub_org + ANSI_NORMAL)
            }
        }
        stage('Pre-Build') {
            sh '''
                #sed -i "s/'replication_factor': '2'/'replication_factor': '1'/g" database/data.cql
                '''
        }
        stage('Build') {
            sh '''
                mvn clean install -DskipTests
                mvn play2:dist -pl analytics-api
                '''
        }
        stage('Package') {
            dir('sunbird-analytics-service-distribution') {
                sh "cp ../analytics-api/target/analytics-api-2.0-dist.zip ."
                sh "/opt/apache-maven-3.6.3/bin/mvn3.6 package -Pbuild-docker-image -Drelease-version=${build_tag}"
            }
        }
        stage('Retagging'){
             sh """
                docker tag sunbird-analytics-service:${build_tag} ${hub_org}/sunbird-analytics-service:${build_tag}
                echo {\\"image_name\\" : \\"sunbird-analytics-service\\", \\"image_tag\\" : \\"${build_tag}\\", \\"node_name\\" : \\"${env.NODE_NAME}\\"} > metadata.json
                """
        }
        stage('Archive artifacts'){
            sh """
                        mkdir lpa_service_artifacts
                        cp analytics-api/target/analytics-api-2.0-dist.zip lpa_service_artifacts
                        zip -j lpa_service_artifacts.zip:${artifact_version} lpa_service_artifacts/*
                    """
            archiveArtifacts artifacts: "lpa_service_artifacts.zip:${artifact_version}", fingerprint: true, onlyIfSuccessful: true
            sh """echo {\\"artifact_name\\" : \\"lpa_service_artifacts.zip\\", \\"artifact_version\\" : \\"${artifact_version}\\", \\"node_name\\" : \\"${env.NODE_NAME}\\"} > metadata.json"""
            archiveArtifacts artifacts: 'metadata.json', onlyIfSuccessful: true
            currentBuild.description = artifact_version
        }
    }
    catch (err) {
        currentBuild.result = "FAILURE"
        throw err
    }
}
