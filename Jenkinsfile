#!groovy
/*
 * Author:  Anand Dasari
 * Client: MicroFocus
 * Date:   Dec 2021
 * Description:  Prod CI/CD pipeline on Production Jenkins
 *
 Changes made as part of Jenkins 2.332.1 upgrade
 Line 62- New SFjenkins-Deployer Key
 Line 68- Added safe Directory git parameter for workspace
 Need to explore on extending Git Time Out which is currently by default set at 10 mins. Code as below
 17-May-2023 - Enabled Shallow checkout and updated refspec values
*/ 
pipeline 
{
  agent 
  {
    kubernetes {
            inheritFrom 'sfdxagent'    // pod template name
            defaultContainer 'sfdxagent'  /// container name
    }
  }
  options {
    skipDefaultCheckout()  //don't do default checkout
  }
    stages{
        stage('Run Shell') {
            steps {
                sh 'hostname'
                echo 'BRANCH NAME: ' + env.BRANCH_NAME
                echo 'CHANGE_ID: ' + env.CHANGE_ID 
            }
        }
        stage('Checkout SCM')
        {
            steps
            {
                //checkout scm
                checkout([
                    $class: 'GitSCM', 
                    branches: [[name: '${env.BRANCH_NAME}'], [name: 'mfitest1']], 
                    extensions: scm.extensions + [[$class: 'CloneOption', noTags: true, reference: '', shallow: true, depth:10]], 
                    userRemoteConfigs: [[
                    url: 'https://hmf.gitlab.otxlab.net/cdo/entapps/sfdc/salesforce.git',
                    credentialsId: 'GitLab-SFDC-Token',
                    name: 'origin', 
                    refspec: '+refs/heads/feature/mfitest1*:refs/remotes/origin/feature/mfitest1* +refs/heads/mfitest1:refs/remotes/origin/mfitest1'
                    ]],
                    doGenerateSubmoduleConfigurations: false
                    ])
                    sh "git config --global --add safe.directory  ${env.WORKSPACE}"
            }
        }
        
        stage('Login to Org'){
            environment{
                    SF_KEY = credentials('SF_SBX_SVR_KEY')
              }
            steps {               
                    script {
                        try {
                            //sh "sfdx auth:sfdxurl:store -f ./config/test1_auth.txt -s"
                            sh 'sfdx force:auth:jwt:grant --instance-url ${SF_SBX_URL} --client-id ${MFITEST1_CONSUMER_KEY} --username ${MFITST1_USR_NAME} --jwt-key-file ${SF_KEY} -s'
                           // sh "sfdx force:org:list"
                        } catch (Exception e) {
                            echo 'Exception occurred: ' + e.toString()
                        }
                    }
            }

        }
        stage('validate-code'){
            when {     
                expression {
                    env.BRANCH_NAME =~ 'feature/mfitest1/*' || env.BRANCH_NAME =~ 'PR-*';
                }                
            } 
            steps{
                echo'======== Validating Changes ============'
                validate()
            }

        }
        stage('Deploy to Org'){
            when {     
                expression {
                    return env.BRANCH_NAME == 'mfitest1'
                }                
            }
            steps{
                echo'======== Deploying Changes ============'
                deploy()
            }

        }
    }
    post {
        success {
            script {
                def mailRecipients = 'mfi.sfdeploymentteam@microfocus.com'
                def jobName = currentBuild.fullDisplayName
                emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                    mimeType: 'text/html',
                    subject: "[Jenkins] ${jobName}",
                    to: "${mailRecipients}",
                    replyTo: "${mailRecipients}",
                    recipientProviders: [[$class: 'CulpritsRecipientProvider']]
             
                    }
            }

        failure {
        script {
                def mailRecipients = 'mfi.sfdeploymentteam@microfocus.com'
                def jobName = currentBuild.fullDisplayName
                emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                    mimeType: 'text/html',
                    subject: "[Jenkins] ${jobName}",
                    to: "${mailRecipients}",
                    replyTo: "${mailRecipients}",
                    recipientProviders: [[$class: 'CulpritsRecipientProvider']]
                }
        }
    }
}
/**
 * function to validate the changes to the organisation
**/
def validate(){
    // validation happens only from feature and PR branch, hnce compare these with main branch
    sh "rm -fr changes"
    sh "mkdir changes"
    if ( (env.BRANCH_NAME).startsWith('feature') ) {
        echo "**** ALL changed files in Feature Branch ****"
        sh "git diff --name-only origin/${env.BRANCH_NAME}..origin/mfitest1"
        echo "_____  creating package.xml and destructive changes---------"                            
        sh "sfdx sgd:source:delta --to 'origin/${env.BRANCH_NAME}' --from 'origin/mfitest1'  --output changes/"
    }else if( (env.BRANCH_NAME).startsWith('PR') ) {
        echo "****** Changes from PR *****"
        sh "git diff --name-only origin/pr/${env.CHANGE_ID}..origin/mfitest1"
        echo "_____  creating package.xml and destructive changes---------"                            
        sh "sfdx sgd:source:delta --to 'origin/pr/${env.CHANGE_ID}' --from 'origin/mfitest1'  --output changes/"
    }
        
    echo "***  package xml with added modified metadata ****"
    sh "cat changes/package/package.xml"

    echo ""
    echo "--- destructiveChanges.xml generated with deleted metadata ---"
    sh "cat changes/destructiveChanges/destructiveChanges.xml"
    echo  "" 

    def addSize = getXMLoutput('package/package.xml')
    echo "Additons ===== ${addSize}"
    if (addSize != 0) {
            echo "***** There are changes in package.xml and validating now ******"
            sh "sfdx project deploy start --manifest changes/package/package.xml --dry-run -l NoTestRun --verbose -w 50"
    }
    else {
        echo "No change to Apply"
    }

    def removeSize = getXMLoutput('destructiveChanges/destructiveChanges.xml')
    echo "${removeSize}"
    if (removeSize != 0) {
            echo "***** There are changes in destructiveChanges.xml and validating now ******"
            sh "sfdx project deploy start --manifest changes/destructiveChanges/package.xml --post-destructive-changes changes/destructiveChanges/destructiveChanges.xml --dry-run --verbose -w 5"
    }
    else {
        echo "No change to Delete"
    }
}

/**
 * function to deploy changes to the organisation
**/
def deploy(){
     ///Assume deploy only happens from proper branch , hence compare head   
    sh "rm -fr changes"
    sh "mkdir changes"
    echo "**** ALL changed files in ****"
    sh "git diff --name-only HEAD^ HEAD"

    echo "_____  creating package.xml and destructive changes---------"                            
    sh "sfdx sgd:source:delta --to 'HEAD' --from 'HEAD^'   --output changes/"
        
    echo "***  package xml with added modified metadata ****"
    sh "cat changes/package/package.xml"


    echo ""
    echo "--- destructiveChanges.xml generated with deleted metadata ---"
    sh "cat changes/destructiveChanges/destructiveChanges.xml"
    echo  "" 

    def addSize = getXMLoutput('package/package.xml')
    echo "Additons ===== ${addSize}"
    if (addSize != 0) {
            echo "***** There are changes in package.xml and Deploying now ******"
            sh "sfdx project deploy start --manifest changes/package/package.xml -l NoTestRun --verbose -w 50"
    }
    else {
        echo "No change to Apply"
    }

    def removeSize = getXMLoutput('destructiveChanges/destructiveChanges.xml')
    echo "${removeSize}"
    if (removeSize != 0) {
            echo "***** There are changes in destructiveChanges.xml and Deploying now ******"
            sh "sfdx project deploy start --manifest changes/destructiveChanges/package.xml --post-destructive-changes changes/destructiveChanges/destructiveChanges.xml --verbose -w 5"
    }
    else {
        echo "No change to Delete"
    }
}
/**
 * Function to parse xml and gives output
 * getXMLoutput --
**/
def getXMLoutput(pParam) {
    def File = readFile "${env.WORKSPACE}/changes/${pParam}"
    echo "${File}"
    def types = 0
    def xml = new XmlSlurper().parseText(File)
    types = xml.types.size()
    return types;
}  