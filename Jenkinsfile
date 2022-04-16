def downloadFileFromGit(gitUrl, branchName, filePath) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding',
        credentialsId: 'githubcredentials',
        usernameVariable: 'GIT_USERNAME',
        passwordVariable: 'GIT_PASSWORD']]) {

        // Get the waf.yaml from devops repo
        sh "git archive --remote=${gitUrl} --format=tar ${branchName} ${filePath} | tar xf -"
    }
}

def updateCloudFormationStacksParallel(stackName, stackRegion, cfnParams) {
    cfnUpdateTasks["${stackName}"] = {
        node {
  
            stage("${stackName}") {
                script {
                    try {
                        sendSlackMessage(
                            "WAF Config Sync in Progress...",
                            " ► Stack: ${stackName} \n ► Region: ${stackRegion}\n",
                            'good',
                            slackMessageChannel
                        )

                        def gitUrl = "https://github.com/Akayrathee/cloudformation"
                        def branchName = "master"
                        def filePath = "waf.yaml"
                        downloadFileFromGit(gitUrl, branchName, filePath)
                    }
                    catch(error) {
                        allCfnUpdateSuccessful = false
                        // Alert to slack about failure
                        echo("Updation of the stack ${stackName} failed. Error = " + error.toString())
                        sendSlackMessage(
                            "WAF Sync Failure",
                            " ► Stack: ${stackName} \n ► Region: ${stackRegion}\n ► buildUrl: ${env.BUILD_URL}\n",
                            'danger',
                            slackMessageChannel
                        )
                    }
                }
            }
        }
    }
}

pipeline {
    agent any 

    stages {
        stage("EB configuration check") {
            steps {
                timestamps {
                    script {
                        dir(buildRootDir) {                
                            def datas = readYaml file: 'config.yaml'

                            cfnUpdateTasks = [:]
                            allCfnUpdateSuccessful = true

                            for(data in datas.wafV2Config) {
                                echo "Updating WAF-${data.stackName} ..."

                                def cfnParams = new JsonSlurperClassic().parseText("{}")
                                cfnParams["ActivateHttpFloodProtectionParam"] = data.activateHttpFloodProtectionParam
                                cfnParams["AssociatedResourceArn"] = data.albArn
                                cfnParams["EndpointType"] = wafEndpointType
                                cfnParams["RateLimitRuleAction"] = data.rateLimitRuleAction
                                cfnParams["RequestThreshold"] = data.requestThreshold

                                echo "Cloudformation params: ${cfnParams}"

                                updateCloudFormationStacksParallel("WAF-${data.stackName}", data.region, cfnParams)

                            }

                            echo "Updating stacks in parallel..."
                            echo "${cfnUpdateTasks}"
                            parallel cfnUpdateTasks
                            echo "Parallel tasks complete."

                            if (!allCfnUpdateSuccessful) {
                                error("One or more stackUpdation failed; Please check the log for more information.")
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            sendSlackMessage(
                "WafDeploymentPipeline Failure",
                " Sync Failed\n ► buildUrl: ${env.BUILD_URL}\n",
                'danger',
                slackMessageChannel
            )
            echo "Failure"
        }
        cleanup {
            echo "Cleaning Workspace.."
            cleanWs()
            echo "Exiting Script"
        }
    }
}
