#!/usr/bin/env groovy
library('common-utils')
def build_user = getUserNameLaunchedJob()

properties([
    parameters([
        string(name: 'EXECUTOR', defaultValue: 'dockerhost', description: 'Executor (node) being used to run job'),
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to take tests from'),
        string(name: 'SUITE', defaultValue: 'allTests_selenoid', description: 'Suite name, e.g. allTests_grid, allTests_selenoid etc. (w/o file extension and path)'),
        string(name: 'TIER', defaultValue: 'test', description: 'Environment to test on. Can be grid, browserstack etc.'),
        string(name: 'TESTS', defaultValue: '', description: 'Comma separated test ids'),
        string(name: 'SUPPORT_RUN_ID', defaultValue: '', description: 'run_id value'),
        choice(name: 'PROJECT_ID', choices: 'New web platform [2]\nExperimental project [13]', description: 'Test Rail project'),
        choice(name: 'ASSIGNEE', choices: 'Mr. QA Robot [50]\nNobody [0]', description: 'Choose assignee for the test run'),
        choice(name: 'SUITE_ID', choices: 'RegressionTree [4472]\nAuto suite from experimental space [4347]', description: 'Choose assignee for the test run'),
        choice(name: 'RUN_ID', choices: 'Create new test run [0]\nExperimental test run [6981]', description: 'Select any run'),
        string(name: 'UPSTREAM_BUILD_ID', defaultValue: '', description: 'Number of parent build'),
        string(name: 'UPSTREAM_JOB_NAME', defaultValue: '', description: 'Name of parent job'),
        string(name: 'BUILD_USER', defaultValue: build_user, description: 'User who started build'),
        string(name: 'TIMEOUT', defaultValue: '30', description: 'Build timeout'),
        booleanParam(name: 'UPDATE_CASES', defaultValue: true, description: 'Update test cases (add new test cases to test run if ones do not exist)'),
        booleanParam(name: 'TESTRAIL', defaultValue: false, description: 'Publish results in Test Rail.'),
    ])
])

def pathToSuite(suiteName = "custom") {
    def suite = [:]
    suite['file'] = "${suiteName}.xml"
    suite['folder'] = "e2eTests/suites"
    suite['fullPath'] = "e2eTests/suites/${suite['file']}"
    return suite
}

timeout(time: params.TIMEOUT, unit: 'MINUTES') {
    timestamps {
        node(params.EXECUTOR) {
            deleteDir();

            git(url: 'https://gitlab.lmru.adeo.com/Elbrus/AEM/QA/AutoTestsJava/jat.git',
                    credentialsId: 'jenkins-gitlab',
                    branch: "${BRANCH}")

            sh("docker pull nexus.lmru.adeo.com:5000/bricks/maven:latest")
            sh("docker pull nexus.lmru.adeo.com:5000/selenoid/firefox:63.0")
            sh("docker pull nexus.lmru.adeo.com:5000/selenoid/chrome:70.0")
            sh("docker pull nexus.lmru.adeo.com:5000/selenoid/firefox:64.0")
            sh("docker pull nexus.lmru.adeo.com:5000/selenoid/chrome:74.0")
            sh("docker pull nexus.lmru.adeo.com:5000/aerokube/selenoid:latest-release")

            def containerName = "${env.JOB_NAME.split('/')[-1]}_selenoid_${env.BUILD_NUMBER}"
            def selenoidOptions = "--name ${containerName} -p 0:4444" +
                    " -v /var/run/docker.sock:/var/run/docker.sock" +
                    " -v `pwd`/e2eTests/config/:/etc/selenoid/:ro" +
                    " -v `pwd`/logs/:/opt/selenoid/logs/"

            def logsCliParams = '-log-output-dir /opt/selenoid/logs -save-all-logs'
            def sessionCliParams = '-service-startup-timeout 5m0s -session-attempt-timeout 5m0s -session-delete-timeout 1s -retry-count 3'
            def performanceCliParams = '-mem 1g'

            def suite

            try {
                docker.image("nexus.lmru.adeo.com:5000/aerokube/selenoid:latest-release ${logsCliParams} ${sessionCliParams} ${performanceCliParams}").withRun(selenoidOptions) { c ->
                    def output = sh(returnStdout: true, script: "docker port ${containerName}").trim()
                    env.PORT = (output =~ /\S+:(\d+)/)[0][1]
                    echo "Selenoid entrypoint will be 172.17.0.1:${env.PORT}"

                    docker.image('nexus.lmru.adeo.com:5000/bricks/maven:latest').inside("--link ${c.id}:${containerName}") {
                        if (params.TESTS == '') {
                            suite = pathToSuite(params.SUITE)
                        } else { // if separate tests specified
                            suite = pathToSuite()

                            // generate custom suite
                            sh(returnStdout: true, script: "mvn -f e2eTests test-compile")
                            sh(returnStdout: true, script: "mvn -e -X -f e2eTests exec:java -Dexec.mainClass=\"all.utils.SuitesGenerator\" -Dexec.classpathScope=test -Dtests=${params.TESTS}")
                        }

                        if (fileExists(suite['fullPath'])) {
                            sh(returnStdout: true, script: "mvn -f e2eTests clean test -e " +
                                    "-DsuiteFile=suites/${suite['file']} " +
                                    "-DignoreTestFailures=false " +
                                    "-Dtier=${params.TIER} " +
                                    "-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=error -B " +
                                    "-Dorg.slf4j.simpleLogger.defaultLogLevel=error").trim()
                        } else {
                            throw new Exception("Suite file ${suite['file']} not found. Aborted.")
                        }
                    }
                }
            } catch (err) {
                currentBuild.result = 'FAILURE'
                echo err.getMessage()
            } finally {
                sleep(3)
                sh("docker stop ${containerName} || true && docker rm ${containerName} || true")

                def allureResultsPath = 'e2eTests/allure-results'
                def buildResultsPath = 'e2eTests/build/reports/tests'

                // archive build artifacts
                archiveArtifacts allowEmptyArchive: true, artifacts: "${buildResultsPath}/*.png", onlyIfSuccessful: false

                if (params.TESTS != '') {
                    archiveArtifacts allowEmptyArchive: true, artifacts: "${suite['fullPath']}", onlyIfSuccessful: false
                }

                allure([report: "allure-report", results: [[path: allureResultsPath]]])
                zip archive: true, dir: allureResultsPath, glob: '', zipFile: 'allure-results.zip'
                zip archive: true, dir: 'logs', glob: '', zipFile: 'selenoid-logs.zip'

                if (params.TESTRAIL) {
                    sh("echo Execute test rail publishing script:")
                    sh("e2eTests/publishResultsToTestRail " +
                            "e2eTests " +
                            "allure-report " +
                            "'${BUILD_URL}' " +
                            "'${params.PROJECT_ID}' " +
                            "'${params.SUITE_ID}' " +
                            "'${params.ASSIGNEE}' " +
                            "'${params.RUN_ID}' " +
                            "'${params.UPDATE_CASES}'")
                }

                deleteDir();
            }
        }
    }
}