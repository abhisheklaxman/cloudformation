import groovy.transform.Field
import groovy.json.JsonSlurperClassic
import net.sf.json.JSONArray
import net.sf.json.JSONObject

// other parameters which is not required
@Field def buildRootDir = "AakashCode"
@Field def slackMessageChannel = "#cloud-deployments"

// waf params
@Field def wafEndpointType = "ALB"

def sendSlackMessage(titleText, messageText, messageColor, channelName){
    echo "Message Sent"	
}

def downloadFileFromGit(gitUrl, branchName, filePath) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding',
        credentialsId: 'githubcredentials',
        usernameVariable: 'GIT_USERNAME',
        passwordVariable: 'GIT_PASSWORD']]) {

        // Get the waf.yaml from devops repo
        sh "git archive --remote=${gitUrl} --format=tar ${branchName} ${filePath} | tar xf -"
        sh "ls"
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

                        def gitUrl = "git@github.com:Akayrathee/cloudformation.git"
                        def branchName = "master"
                        def filePath = "waf.yaml"
                        downloadFileFromGit(gitUrl, branchName, filePath)
                        def waf = readYaml file: 'waf.yaml'
                        echo "We are here"
                        
                        withAWS(credentials: 'aakashawscredntials', region: stackRegion){
                            def outputs = cfnUpdate(
                                stack:"${stackName}",
                                file:waf,
                                params:cfnParams,
                                timeoutInMinutes:180,
                                pollInterval:10000
                            )
                            print(outputs)
                    }}
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
                        
			    def gitUrl = "git@github.com:Akayrathee/cloudformation.git"
                            def branchName = "master"
                            def filePath = "config.yaml"                
                            def datas = readYaml file: 'config.yaml'

                            cfnUpdateTasks = [:]
                            allCfnUpdateSuccessful = true

                            echo "Data : ${datas}"
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
