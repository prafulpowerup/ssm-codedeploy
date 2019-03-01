# ssm-codedeploy
automate your deploymnet along with administrative task

Here in this blog I am discussing how we have used AWS SSM Run Command for administration tasks and AWS CodeDeploy To automate the deployments on admin server and App server.
Since AWS Systems Manager public documents limits the actions you want to perform on your managed instances, so we have created your own documents, which performs actions like running SQL commands, flushing Cache, and restarting services. When creating a new document, we recommend that you use latest schema version (https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-doc-syntax.html).

