try{
pipeline {
  agent {
        label {
            label ""
        }
      }
  parameters {
      string(defaultValue: "dev", description: 'Which Git Branch  MODMAN?', name: 'GIT_BRANCH_MODMAN')
      string(defaultValue: "dev", description: 'Which Git Branch  REUSABLE?', name: 'GIT_BRANCH_REUSABLE')
      booleanParam(defaultValue: false, description: 'Run SQL SCRIPT MODMAN', name: 'RUN_SQL_MODMAN')
      booleanParam(defaultValue: false, description: 'Run SQL SCRIPT REUSABLE', name: 'RUN_SQL_REUSABLE')
      booleanParam(defaultValue: false, description: 'App Server Deploymnet', name: 'APP_SERVER_DEPLOYMENT')
      booleanParam(defaultValue: true, description: 'Deploy MODMAN', name: 'DEPLOY_MODMAN')
      booleanParam(defaultValue: true, description: 'Deploy REUSABLE', name: 'DEPLOY_REUSABLE')
      booleanParam(defaultValue: false, description: 'Flush Redis ?', name: 'FLUSH_REDIS')
      string(defaultValue: "modman-admin ", description: 'AWS CodeDeply Deploymnet Group Name MODMAN ?', name: 'CODEDEPLOY_DEPLOYMENTGROUP_MODMAN')
      string(defaultValue: "reusables-admin", description: 'AWS CodeDeply Deploymnet Group Name REUSABLE ?', name: 'CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE')
      string(defaultValue: "modman-App", description: 'AWS CodeDeply Deploymnet Group Name MODMAN for App', name: 'CODEDEPLOY_DEPLOYMENTGROUP_MODMAN_APP')
      string(defaultValue: "reusables-App", description: 'AWS CodeDeply Deploymnet Group Name REUSABLE for App ?', name: 'CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE_APP')
      string(defaultValue: "myApp", description: 'AWS CodeDeploy Application Name ?', name: 'CODEDEPLOY_APPLICATION')
      string(defaultValue: "bucket-codedeploy", description: 'S3 bucket name for CodeDeploy Revisions ?', name: 'BUCKET')
      string(defaultValue: "ap-southeast-1", description: 'AWS REGION ?', name: 'REGION')
      string(defaultValue: "/dir/indexing/", description: 'The Directory for Indexing', name: 'WORKING_DIRECTORY_INDEXING')
      string(defaultValue: "/dir/modman", description: 'The Directory for MODMAN', name: 'WORKING_DIRECTORY_REDIS_BUILD')
      string(defaultValue: "/dir/reusable", description: 'The Directory for REUSABLE', name: 'WORKING_DIRECTORY_REUSABLE')
      string(defaultValue: "/docroot/app/etc", description: 'The Directory for Redis Flush', name: 'WORKING_DIRECTORY_REDIS_FLUSH')
      string(defaultValue: "catalog_category_1,catalog_product_1", description: 'catalog Names for redis Indexing', name: 'CATALOGS_NAMES')
      string(defaultValue: "0", description: 'DB number to Flush', name: 'DB_NUMBER')
      string(defaultValue: "/dir/app/modman", description: 'The Directory for MODMAN - App Server', name: 'WORKING_DIRECTORY_MODMAN_APP')
      string(defaultValue: "/dir/app/reusable", description: 'The Directory for REUSABLE - App Server', name: 'WORKING_DIRECTORY_REUSABLE_APP')
      string(defaultValue: "/dir/static/", description: 'The Directory for STATIC ASSETS Admin Server', name: 'WORKING_DIRECTORY_STATIC_ASSETS')
      string(defaultValue: "bucket-static-assets", description: 'S3 bucket name for Static Assets', name: 'BUCKET_STATIC_ASSETS')


    }//closing parameters
  stages {
    stage('GitSCM Checkout Modman Admin Server') {
      steps {
        ws("/var/lib/jenkins/workspace/modman/") {
          cleanWs()
              checkout([$class: 'GitSCM', branches: [[name: '*/$GIT_BRANCH_MODMAN']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://gitxxx.com/myproject/modman-module.git']]])

        }
      }
    } //closing GitSCM Checkout Modman
    
    stage('SQL script Automations MODMAN') {
      steps {
        ws("/var/lib/jenkins/workspace/modman/") {
        withCredentials([string(credentialsId: 'ADMIN_INSTANCE_ID', variable: 'ADMIN_INSTANCE_ID')]) {
      sh '''
      if [ $(find ${WORKSPACE} -maxdepth 1 -name "*.sql" | wc -l) = "1"  ];then
        if [ ${RUN_SQL_MODMAN} == "true" ]; then
            echo "cleaning sql queries from the bucket"
            aws s3 rm s3://${BUCKET}/sql-automation/ --recursive  --include "/*.sql"  --region ${REGION}
            ##copying sql files to the s3
            for i in $(find ${WORKSPACE} -maxdepth 1 -name "*.sql")
            do
             aws s3 cp $i s3://${BUCKET}/sql-automation/ --region ${REGION} 
            done
          
            #RUN SSM RUN Command
            sh_command_id=$(aws ssm send-command --document-name "Run_SQL" --targets "Key=instanceids,Values=${INSTANCE_ID}" --parameters '{"workingDirectory":["'"${WORKING_DIRECTORY_REDIS_FLUSH}"'"],"executionTimeout":["500"],"s3Bucket":["'"${BUCKET}"'"],"region":["'"${REGION}"'"]}' --timeout-seconds 600 --max-concurrency "50" --max-errors "0" --cloud-watch-output-config '{"CloudWatchOutputEnabled":true}' --region ${REGION} --output text --query "Command.CommandId");
            sleep 150
            
            #check output of the command
            OUTPUT=$(aws ssm list-command-invocations --command-id "$sh_command_id" --details --region ${REGION} --query "CommandInvocations[].CommandPlugins[].{Status:Status,Output:Output}")
            
            #get status of the output
            FINAL_OUTPUT=$(echo $OUTPUT | jq '.[] | .Status')
            SUCCESS='"Success"'
           
            #check if status is SUCCESS
            if [ ${FINAL_OUTPUT} = ${SUCCESS} ]
             then
              echo "SQL script executed Successfully for MODMAN"
            else
             exit 1;
             echo "!!!SQL script execution Failure for MODMAN!!!"
            fi
    
    
            else
             echo " Skipping SQL Scripts for MODMAN"
            exit 0
            fi
        
         else
         echo "no SQL files found for MODMAN "
        fi

        '''
 
      }
    }
    }
    } //closing SQL script Automations MODMAN
    
    stage('Deploy MODMAN on Admin Server ') {
     steps {
        ws("/var/lib/jenkins/workspace/modman/") {
            if ("${params.DEPLOY_MODMAN}" == "true") {
            sh '''
              ##pull the code-deploy necessary files from s3
              mkdir .codedeploy-scripts
              aws s3 cp s3://${BUCKET}/code-deploy-files/appspec.yml ${WORKSPACE} --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/change_permission.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/linker.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/remove_files.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/check_directory.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION} 

              ##Changing directory according to the client
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REDIS_BUILD}|" appspec.yml
              echo "changed appspec.yml"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REDIS_BUILD}|" .codedeploy-scripts/change_permission.sh
              echo "changed change_permission.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REDIS_BUILD}|" .codedeploy-scripts/linker.sh
              echo "changed linker.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REDIS_BUILD}|" .codedeploy-scripts/remove_files.sh
              echo "changed remove_files.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REDIS_BUILD}|" .codedeploy-scripts/check_directory.sh
              echo "changed check_directory.sh"
              
              ##make executables
              chmod +x --recursive ${WORKSPACE}/.codedeploy-scripts/*
             
             '''
             
             //Defining Variables
              DATE = sh (
                script: 'date +%Y%m%d%H%M%S',
                returnStdout: true
              ).trim()


              ZIPFILE = "${BUILD_NUMBER}-${DATE}-${CODEDEPLOY_DEPLOYMENTGROUP_MODMAN}.zip"
              
              //Creating and Uploading the Code Zip file to S3 Bucket
              sh (
                script: "zip -r $ZIPFILE .",
                returnStdout: true
              ).trim()

              sh (
                script: "aws s3 cp $ZIPFILE s3://$BUCKET/$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_MODMAN/ --region $REGION",
                returnStdout: true
              ).trim()
              
             // Deploying through CodeDeploy
              DEPLOYMENT=sh (
                script: "aws deploy create-deployment --application-name $CODEDEPLOY_APPLICATION --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name $CODEDEPLOY_DEPLOYMENTGROUP_MODMAN --description \"Deployment through Jenkins $JOB_NAME $BUILD_NUMBER\"  --s3-location bucket=$BUCKET,bundleType=zip,key=$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_MODMAN/$ZIPFILE --region $REGION --output text",
                returnStdout: true
              ).trim()
              

              //Wait for the deployment to be successful
             sh (
                script: "aws deploy wait deployment-successful --deployment-id $DEPLOYMENT --region ${REGION}",
                returnStdout: true
              ).trim()
              
              sh '''
              EXIT_STATUS=$?

              if [ ${EXIT_STATUS} -eq 0 ]; then
                echo "Deployment Successful on MODMAN!!"
              else
                #If the deployment fails, rollback to the last succeeded deployment.
                echo "Deployment Failed on MODMAN!!"
                exit 1;

              fi
            '''
      }
            
        }
    }
    } //closing Deploy MODMAN on Admin Server

    stage('GitSCM Checkout reusables Admin Server') {
      steps {
        ws("/var/lib/jenkins/workspace/reusables/") {
          cleanWs()
              checkout([$class: 'GitSCM', branches: [[name: '*/$GIT_BRANCH_REUSABLE']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://gitxxx.com/myproject/reusables-module.git']]])
            }
      }
    } //closing GitSCM Checkout reusables

    stage('SQL script Automations REUSABLE') {
      steps {
        ws("/var/lib/jenkins/workspace/reusables/") {
        withCredentials([string(credentialsId: 'ADMIN_INSTANCE_ID', variable: 'ADMIN_INSTANCE_ID')]) {
      sh '''
      if [ $(find ${WORKSPACE} -maxdepth 1 -name "*.sql" | wc -l) = "1"  ];then
        if [ ${RUN_SQL_MODMAN} == "true" ]; then
            echo "cleaning sql queries from the bucket"
            aws s3 rm s3://${BUCKET}/sql-automation-reusables/ --recursive  --include "/*.sql"  --region ${REGION}
            ##copying sql files to the s3
            for i in $(find ${WORKSPACE} -maxdepth 1 -name "*.sql")
            do
             aws s3 cp $i s3://${BUCKET}/sql-automation-reusables/ --region ${REGION} 
            done
          
            #RUN SSM RUN Command
            sh_command_id=$(aws ssm send-command --document-name "Run_SQL_REUSABLES" --targets "Key=instanceids,Values=${INSTANCE_ID}" --parameters '{"workingDirectory":["'"${WORKING_DIRECTORY_REDIS_FLUSH}"'"],"executionTimeout":["500"],"s3Bucket":["'"${BUCKET}"'"],"region":["'"${REGION}"'"]}' --timeout-seconds 600 --max-concurrency "50" --max-errors "0" --cloud-watch-output-config '{"CloudWatchOutputEnabled":true}' --region ${REGION} --output text --query "Command.CommandId");
            sleep 150
            
            #check output of the command
            OUTPUT=$(aws ssm list-command-invocations --command-id "$sh_command_id" --details --region ${REGION} --query "CommandInvocations[].CommandPlugins[].{Status:Status,Output:Output}")
            
            #get status of the output
            FINAL_OUTPUT=$(echo $OUTPUT | jq '.[] | .Status')
            SUCCESS='"Success"'
          
            if [ ${FINAL_OUTPUT} = ${SUCCESS} ]
             then
              echo "SQL script executed Successfully for Resuables"
            else
             exit 1;
             echo "!!!SQL script execution Failure for Resuables!!!"
            fi
    
    
            else
             echo " Skipping SQL Scripts for Resuables"
            exit 0
            fi
        
         else
         echo "no SQL files found on Resuables "
        fi

        '''
 
      }
    }
    }
    } //closing SQL script Automations REUSABLE
    
    stage('Deploy REUSABLE on Admin Server ') {
        steps {
         ws("/var/lib/jenkins/workspace/reusables/"){
             if ("${params.DEPLOY_REUSABLE}" == "true") {
                
         sh '''
              ##pull the code-deploy necessary files from s3 
              mkdir .codedeploy-scripts
              aws s3 cp s3://${BUCKET}/code-deploy-files/appspec.yml ${WORKSPACE} --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/change_permission.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/linker.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files/remove_files.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION} 
              aws s3 cp s3://${BUCKET}/code-deploy-files/check_directory.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              ##Changing directory according to the client
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE}|" appspec.yml
              echo "changed appspec.yml"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE}|" .codedeploy-scripts/change_permission.sh
              echo "changed change_permission.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE}|" .codedeploy-scripts/linker.sh
              echo "changed linker.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE}|" .codedeploy-scripts/remove_files.sh
              echo "changed remove_files.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE}|" .codedeploy-scripts/check_directory.sh
              echo "changed check_directory.sh"


              ##make executables
              chmod +x --recursive ${WORKSPACE}/.codedeploy-scripts/*
             
             '''
             
             //Defining Variables
              DATE = sh (
                script: 'date +%Y%m%d%H%M%S',
                returnStdout: true
              ).trim()


              ZIPFILE = "${BUILD_NUMBER}-${DATE}-${CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE}.zip"
              
              //Creating and Uploading the Code Zip file to S3 Bucket
              sh (
                script: "zip -r $ZIPFILE .",
                returnStdout: true
              ).trim()

              sh (
                script: "aws s3 cp $ZIPFILE s3://$BUCKET/$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE/ --region $REGION",
                returnStdout: true
              ).trim()
              
             // Deploying through CodeDeploy
              DEPLOYMENT=sh (
                script: "aws deploy create-deployment --application-name $CODEDEPLOY_APPLICATION --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name $CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE --description \"Deployment through Jenkins $JOB_NAME $BUILD_NUMBER\" --s3-location bucket=$BUCKET,bundleType=zip,key=$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE/$ZIPFILE --region $REGION --output text",
                returnStdout: true
              ).trim()
              

              //Wait for the deployment to be successful
             def DEP_STATUS=sh (
                script: "aws deploy wait deployment-successful --deployment-id $DEPLOYMENT --region ${REGION}",
                returnStatus: true
                )
                
             if (DEP_STATUS == 0) {
                 echo "Deployment Successful for REUSABLES on Admin Server!!"
             }
             if (DEP_STATUS != 0) {
              sh '''
                #If the deployment fails, rollback to the last succeeded deployment for Reusables is Automated
                echo "Deployment Failed for REUSABLES..!!"
                echo "Redeploymnet of Modman with last successfull build  is Carring out "
                #getting last successful deploymnet id of the Modman
                LAST_SUCCESS_ID_MODMAN=$(aws deploy list-deployments --application-name ${CODEDEPLOY_APPLICATION} --deployment-group-name ${CODEDEPLOY_DEPLOYMENTGROUP_MODMAN} --include-only-statuses Succeeded --region ${REGION} --query deployments[1] --output text);
                #get the key of prevously successful deployment
                KEY=$(aws deploy get-deployment --deployment-id ${LAST_SUCCESS_ID_MODMAN} --region ${REGION} --query deploymentInfo.revision.s3Location.key);
      
                #carry out the deploymnet with the new key again for Modman
                ROLLBACK_DEPLOYMENT=$(aws deploy create-deployment --application-name ${CODEDEPLOY_APPLICATION} \
                --deployment-config-name CodeDeployDefault.OneAtATime \
                --deployment-group-name ${CODEDEPLOY_DEPLOYMENTGROUP_MODMAN} \
                --description "Deployment through Jenkins ${JOB_NAME} ${BUILD_NUMBER}" \
                --s3-location bucket=${BUCKET},bundleType=zip,key=${KEY} \
                --region ${REGION} --output text);
                
                #check for the deploymnet status
                aws deploy wait deployment-successful --deployment-id ${ROLLBACK_DEPLOYMENT} --region ${REGION}
                EXIT_STATUS=$?
                echo ${EXIT_STATUS}
                if [ ${EXIT_STATUS} -eq 0 ]; then
                  echo "ReDeployment Successful for Modman!!"
                else
                  echo "ReDeployment Failed!!! for Modman Please check AWS Code deploy Logs for Modman!!"
                  exit 1;
                fi
            '''
             }
          }
        }
      }
    } // closing Deploy REUSABLE on Admin Server

    stage('Redis Flush') {
      steps {
        if ("${params.FLUSH_REDIS}" == "true") {
      withCredentials([string(credentialsId: 'ADMIN_INSTANCE_ID', variable: 'ADMIN_INSTANCE_ID')]) {
      sh '''
        #SSM RUN Command
        sh_command_id=$(aws ssm send-command --document-name "Redis-Flush" --targets "Key=instanceids,Values=${ADMIN_INSTANCE_ID}" --parameters '{"workingDirectory":["'"${WORKING_DIRECTORY_REDIS_FLUSH}"'"],"executionTimeout":["360"],"dbNumber":["'"$DB_NUMBER"'"]}' --timeout-seconds 600 --max-concurrency "50" --max-errors "0" --cloud-watch-output-config '{"CloudWatchOutputEnabled":true}' --region ${REGION} --output text --query "Command.CommandId");
        sleep 50

        #get output of the Command
        OUTPUT=$(aws ssm list-command-invocations --command-id "$sh_command_id" --details --region ${REGION} --query "CommandInvocations[].CommandPlugins[].{Status:Status}")

        #Check status of the SSM run command
        FINAL_OUTPUT=$(echo $OUTPUT | jq '.[] | .Status')

        SUCCESS='"Success"'
          
        if [ ${FINAL_OUTPUT} = ${SUCCESS} ]
          then
          echo "Redis Flush Successful "
        else
          exit 1;
          echo "!!!Redis Flush Failure!!!"
        fi
      '''
      }
    }
      }
    } // closing Redis Flush

    stage('Redis Indexing') {
      steps {
      withCredentials([string(credentialsId: 'ADMIN_INSTANCE_ID', variable: 'ADMIN_INSTANCE_ID')]) {
      sh '''
        sh_command_id=$(aws ssm send-command --document-name "Redis-Indexing" --targets "Key=instanceids,Values=${ADMIN_INSTANCE_ID}" --parameters '{"workingDirectory":["'"${WORKING_DIRECTORY_INDEXING}"'"],"executionTimeout":["3600"],"catalogNames":["'"$CATALOGS_NAMES"'"]}' --timeout-seconds 600 --max-concurrency "50" --max-errors "0" --cloud-watch-output-config '{"CloudWatchOutputEnabled":true}' --region ${REGION} --output text --query "Command.CommandId");
        sleep 300
        
        OUTPUT=$(aws ssm list-command-invocations --command-id ${sh_command_id} --details --region ${REGION} --query "CommandInvocations[].CommandPlugins[].{Status:Status,Output:Output}")
          
        FINAL_OUTPUT=$(echo $OUTPUT | jq '.[] | .Status')
        
        SUCCESS='"Success"'
        
        if [ ${FINAL_OUTPUT} = ${SUCCESS} ]
          then
          echo "Indexing Successful "
        else
          exit 1;
          echo "!!!Indexing Failure!!!"
        fi
      '''
      }
      }
    } //closing Redis Indexing

    stage('Redis Build') {
      steps {
      withCredentials([string(credentialsId: 'ADMIN_INSTANCE_ID', variable: 'ADMIN_INSTANCE_ID')]) {
      sh '''
        sh_command_id=$(aws ssm send-command --document-name "Redis-Build" --targets "Key=instanceids,Values=${ADMIN_INSTANCE_ID}" --parameters '{"workingDirectory":["'"${WORKING_DIRECTORY_REDIS_BUILD}"'"],"executionTimeout":["500"]}' --timeout-seconds 600 --max-concurrency "50" --max-errors "0" --cloud-watch-output-config '{"CloudWatchOutputEnabled":true}' --region ${REGION} --output text --query "Command.CommandId");
        sleep 120
        OUTPUT=$(aws ssm list-command-invocations --command-id "$sh_command_id" --details --region ${REGION} --query "CommandInvocations[].CommandPlugins[].{Status:Status,Output:Output}")

        FINAL_OUTPUT=$(echo $OUTPUT | jq '.[] | .Status')

        SUCCESS='"Success"'
            
        if [ ${FINAL_OUTPUT} = ${SUCCESS} ]
          then
          echo "Redis Build Successful "
        else
          exit 1;
          echo "!!!Redis Build Failure!!!"
        fi
      '''
      }
      }
    } //closing Redis build
    
    stage('GitSCM Checkout Modman App Server') {
      steps {
        ws("/var/lib/jenkins/workspace/modman-App/") {
          if ("${params.APP_SERVER_DEPLOYMENT}" == "true") {
          cleanWs()
              checkout([$class: 'GitSCM', branches: [[name: '*/$GIT_BRANCH_MODMAN']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/myproject/modman-module.git']]])

        }
      }
      }
    } 
    
    stage('Deploy MODMAN on App Server ') {
     steps {
        ws("/var/lib/jenkins/workspace/modman-App/") {
           if ("${params.APP_SERVER_DEPLOYMENT}" == "true") {
            if ("${params.DEPLOY_MODMAN}" == "true") {
            sh '''
              ##pull the code-deploy necessary files from s3 
              mkdir .codedeploy-scripts
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/appspec.yml ${WORKSPACE} --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/change_permission.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/varnish_restart.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/remove_files.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION} 
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/check_directory.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              ##Changing directory according to the client
              sed -i -e "s|changedir|${WORKING_DIRECTORY_MODMAN_APP}|" appspec.yml
              echo "changed appspec.yml"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_MODMAN_APP}|" .codedeploy-scripts/change_permission.sh
              echo "changed change_permission.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_MODMAN_APP}|" .codedeploy-scripts/varnish_restart.sh
              echo "changed linker.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_MODMAN_APP}|" .codedeploy-scripts/remove_files.sh
              echo "changed remove_files.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_MODMAN_APP}|" .codedeploy-scripts/check_directory.sh
              echo "changed check_directory.sh"
              ##make executables
              chmod +x --recursive ${WORKSPACE}/.codedeploy-scripts/*
             
             '''
             
             //Defining Variables
              DATE = sh (
                script: 'date +%Y%m%d%H%M%S',
                returnStdout: true
              ).trim()


              ZIPFILE = "${BUILD_NUMBER}-${DATE}-${CODEDEPLOY_DEPLOYMENTGROUP_MODMAN_APP}.zip"
              
              //Creating and Uploading the Code Zip file to S3 Bucket
              sh (
                script: "zip -r $ZIPFILE .",
                returnStdout: true
              ).trim()

              sh (
                script: "aws s3 cp $ZIPFILE s3://$BUCKET/$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_MODMAN_APP/ --region $REGION",
                returnStdout: true
              ).trim()
              
             // Deploying through CodeDeploy
              DEPLOYMENT=sh (
                script: "aws deploy create-deployment --application-name $CODEDEPLOY_APPLICATION --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name $CODEDEPLOY_DEPLOYMENTGROUP_MODMAN_APP --description \"Deployment through Jenkins $JOB_NAME $BUILD_NUMBER\" --s3-location bucket=$BUCKET,bundleType=zip,key=$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_MODMAN_APP/$ZIPFILE --region $REGION --output text",
                returnStdout: true
              ).trim()
              

              //Wait for the deployment to be successful
             sh (
                script: "aws deploy wait deployment-successful --deployment-id $DEPLOYMENT --region ${REGION}",
                returnStdout: true
              ).trim()
              
              sh '''
              EXIT_STATUS=$?

              if [ ${EXIT_STATUS} -eq 0 ]; then
                echo "Deployment Successful on MODMAN App Servers!!"
              else
                #If the deployment fails, rollback to the last succeeded deployment.
                echo "Deployment Failed on MODMAN App Servers!!"
                exit 1;

              fi
            '''
      }
    }
            
        }
    }
    } //closing Deploy MODMAN on App Server
    
    stage('GitSCM Checkout reusables App Server') {
      steps {
        ws("/var/lib/jenkins/workspace/reusables-app/") {
          if ("${params.APP_SERVER_DEPLOYMENT}" == "true") {
          cleanWs()
              checkout([$class: 'GitSCM', branches: [[name: '*/$GIT_BRANCH_REUSABLE']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/specommerce/magento-reusable-modules.git']]])
            }
          }
      }
    } //closing GitSCM Checkout reusables App

    
    stage('Deploy REUSABLE on App Server ') {
        steps {
         ws("/var/lib/jenkins/workspace/reusables-app/"){
          if ("${params.APP_SERVER_DEPLOYMENT}" == "true") {
             if ("${params.DEPLOY_REUSABLE}" == "true") {
                
         sh '''
              ##pull the code-deploy necessary files from s3 
              mkdir .codedeploy-scripts
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/appspec.yml ${WORKSPACE} --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/change_permission.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/varnish_restart.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/remove_files.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION} 
              aws s3 cp s3://${BUCKET}/code-deploy-files-app-server/check_directory.sh ${WORKSPACE}/.codedeploy-scripts/ --region ${REGION}


              ##Changing directory according to the client
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE_APP}|" appspec.yml
              echo "changed appspec.yml"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE_APP}|" .codedeploy-scripts/change_permission.sh
              echo "changed change_permission.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE_APP}|" .codedeploy-scripts/varnish_restart.sh
              echo "changed linker.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE_APP}|" .codedeploy-scripts/remove_files.sh
              echo "changed remove_files.sh"
              sed -i -e "s|changedir|${WORKING_DIRECTORY_REUSABLE_APP}|" .codedeploy-scripts/check_directory.sh
              echo "changed check_directory.sh"
              ##make executables
              chmod +x --recursive ${WORKSPACE}/.codedeploy-scripts/*
             
             '''
             
             //Defining Variables
              DATE = sh (
                script: 'date +%Y%m%d%H%M%S',
                returnStdout: true
              ).trim()


              ZIPFILE = "${BUILD_NUMBER}-${DATE}-${CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE_APP}.zip"
              
              //Creating and Uploading the Code Zip file to S3 Bucket
              sh (
                script: "zip -r $ZIPFILE .",
                returnStdout: true
              ).trim()

              sh (
                script: "aws s3 cp $ZIPFILE s3://$BUCKET/$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE_APP/ --region $REGION",
                returnStdout: true
              ).trim()
              
             // Deploying through CodeDeploy
              DEPLOYMENT=sh (
                script: "aws deploy create-deployment --application-name $CODEDEPLOY_APPLICATION --deployment-config-name CodeDeployDefault.OneAtATime --deployment-group-name $CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE_APP --description \"Deployment through Jenkins $JOB_NAME $BUILD_NUMBER\" --s3-location bucket=$BUCKET,bundleType=zip,key=$CODEDEPLOY_APPLICATION/$CODEDEPLOY_DEPLOYMENTGROUP_REUSABLE_APP/$ZIPFILE --region $REGION --output text",
                returnStdout: true
              ).trim()
              

              //Wait for the deployment to be successful
             sh (
                script: "aws deploy wait deployment-successful --deployment-id $DEPLOYMENT --region ${REGION}",
                returnStdout: true
              ).trim()
              
              sh '''
              EXIT_STATUS=$?

              if [ ${EXIT_STATUS} -eq 0 ]; then
                echo "Deployment Successful for REUSABLES on App Servers!!"
              else
                #If the deployment fails, rollback to the last succeeded deployment.
                echo "Deployment Failed for REUSABLES on App Servers!!"
                exit 1;

              fi
            '''
          }
        }
        }
      }
    } // closing Deploy REUSABLE on App Server

    stage('S3 Static Assets Sync') {
      steps {
      withCredentials([string(credentialsId: 'ADMIN_INSTANCE_ID', variable: 'ADMIN_INSTANCE_ID')]) {
      sh '''
        sh_command_id=$(aws ssm send-command --document-name "Static_Assets_S3Sync" --targets "Key=instanceids,Values=${ADMIN_INSTANCE_ID}" --parameters '{"workingDirectory":["'"${WORKING_DIRECTORY_STATIC_ASSETS}"'"],"executionTimeout":["500"],"s3Bucket":["'"${BUCKET_STATIC_ASSETS}"'"],"region":["'"${REGION}"'"]}' --timeout-seconds 600 --max-concurrency "50" --max-errors "0" --cloud-watch-output-config '{"CloudWatchOutputEnabled":true}' --region ${REGION} --output text --query "Command.CommandId");
        sleep 200
        OUTPUT=$(aws ssm list-command-invocations --command-id "$sh_command_id" --details --region ${REGION} --query "CommandInvocations[].CommandPlugins[].{Status:Status,Output:Output}")

        FINAL_OUTPUT=$(echo $OUTPUT | jq '.[] | .Status')

        SUCCESS='"Success"'
            
        if [ ${FINAL_OUTPUT} = ${SUCCESS} ]
           then
           echo "S3 Static Assets Sync completed Successfully "
        else
           exit 1;
           echo "!!! S3 Static Assets Sync Failure!!!"
        fi
      '''
      }
      }
    } //closing S3 Static Assets Sync

    stage('CloudFront invalidations') {
      steps {
      sh '''
        #send http post request to api with event BUCKET_NAME with lambda intregration
        curl --request POST --data '{"BUCKET_NAME":"'"${BUCKET_STATIC_ASSETS}"'"}' https://<api-name>.<api-gateway-name>.<aws-region>.amazonaws.com/<stage>
        EXIT_STATUS=$?
        echo ${EXIT_STATUS}
        if [ ${EXIT_STATUS} -eq 0 ]; then
          echo "S3 bucket invalidations completed"
        else
          echo "Invalidations Failed!!!"
          exit 1;
        fi
      '''
      }
    }
} //closing stages
    
} //closing pipeline

}//closing try
catch (err){
  currentBuild.result = "FAILURE"
  throw err
}//closing catch
