 Setup CI/CD using AWS CodePipeline


- Create a Pipeline (Console)
To create a pipeline in the console, you'll need to provide the source file location and information about the providers you will use for your actions.

When you use the pipeline wizard, AWS CodePipeline creates the names of stages (Source, Build, Staging).
These names cannot be changed. 
However, you can delete Build and Staging if you prefer to alter the names. 
You can give more specific names (for example, BuildToGamma or DeployToProd) to stages you add later.

Also, existing pipeline configuration can be exported and used to create pipeline in another region.

Sign in to the AWS Management Console and open the AWS CodePipeline console at http://console.aws.amazon.com/codepipeline.

On the CodePipeline Home page, choose Create pipeline.

On the Step 1: Choose pipeline settings page, in the Pipeline name box, type the name for your pipeline like WebAppPipeline.

For Service role, Select Existing service role and choose the Role name from drop down starting with DevopsWorkshop

For Artifact store, Select Custom location and choose the Bucket from drop down starting with cicd-workshop, and then choose Next step.

Question: Does in a single AWS account, each pipeline you create in a region must have a unique name? 
          Can you reused names for pipelines in different regions?



On the Step 2: 
Source page, in the Source provider drop-down list, choose the type of repository where your source code is stored and specify its required options:

AWS CodeCommit: 
In Repository name, choose the name of the AWS CodeCommit repository you created in Lab 1 to use as the source location for your pipeline. 
In Branch name, from the drop-down list, choose the master branch.
In Change Detection Mode leave the default selection of Amazon CloudWatch Events selection. Choose Next step.

On the Step 3: 
Build page, do the following
Choose AWS CodeBuild, and then Select an existing build project we created in Lab 1.
Then choose Next step.

On the Step 4: 
Deploy page, do the following, and then choose Next step.
Choose the following default providers from the Deployment provider drop-down list.

AWS CodeDeploy: 
Type or choose the name of an existing AWS CodeDeploy application in Application name and the name of a deployment group for that application in Deployment group created in Lab2 and then choose Next step.

On the Step 5: 
Review page, review your pipeline configuration, and then choose Create pipeline to create the pipeline.
Please provide for tutor Images that shows successfully created pipeline, pipeline-complete

Now that you've created your pipeline, you can view it in the console. 
Pipeline will start automatically in few minutes. 
Otherwise, test it by manually clicking the Release button.

Please provide for tutor Images that shows successfully executed pipeline, pipeline-released

Stage 2: Create CodeDeploy Deployment group for Production

Run the following to create a deployment group and associates it with the specified application and the user's AWS account. You need to replace the service role ARN we created using roles stack.
```sh
user:~/environment/WebAppRepo (master) $ echo YOUR-CODEDEPLOY-ROLE-ARN: $(aws cloudformation describe-stacks --stack-name DevopsWorkshop-roles | jq -r '.Stacks[0].Outputs[]|select(.OutputKey=="CodeDeployRoleArn")|.OutputValue')
```

```sh
user:~/environment $ aws deploy create-deployment-group --application-name DevOps-WebApp  \
--deployment-config-name CodeDeployDefault.OneAtATime \
--deployment-group-name DevOps-WebApp-ProdGroup \
--ec2-tag-filters Key=Name,Value=ProdWebApp01,Type=KEY_AND_VALUE \
--service-role-arn <<REPLACE-WITH-YOUR-CODEDEPLOY-ROLE-ARN>>
```

 Question: Are We using the different group name and Production tag to attach instance to the deployment group?

Stage 3: Edit a Pipeline (Console)

You can use the AWS CodePipeline console to add, edit, or remove stages in a pipeline, as well as to add, edit, or remove actions in a stage.

We will edit the pipeline to add the stage for production deployment and introduce manual gating for production deployment.

On the pipeline details page, choose Edit.
This opens the editing page for the pipeline.
To add a stage, choose + Add stage after the existing Deploy Stage.
Provide a name for the stage as Production, and then add an one action to it. 
Items marked with an asterisk are required.
Then choose + Add action group. 
In Edit Action section: provide name as ProductionDeployment and action provider as AWS CodeDeploy

In AWS CodeDeploy: 
Type or choose the name of an existing AWS CodeDeploy application in Application name and the name the production deployment group created previous stage

In Input artifacts: 
select the BuildArtifact
Choose Save.
Finally, save changes by clicking Save button on top.

Stage 4: 
Add Manual approval action
In AWS CodePipeline, you can add an approval action to a stage in a pipeline at the point where you want the pipeline execution to stop so that someone with the required AWS Identity and Access Management permissions can approve or reject the action.

If the action is approved, the pipeline execution resumes. 
If the action is rejected—or if no one approves or rejects the action within seven days of the pipeline reaching the action and stopping—the result is the same as an action failing, and the pipeline execution does not continue.

Create SNS topic for Approval notification. And note the topic ARN from the result.

```sh
user:~/environment $ aws sns create-topic --name WebApp-Approval-Topic --region <YOUR-REGION>
```

Subscribe to the topic using your email id. Replace the ARN and email id placeholders accordingly.

```sh
user:~/environment $ aws sns subscribe --topic-arn <<REPLACE-YOUR-TOPIC-ARN>> \
--protocol email \
--notification-endpoint <<REPLACE-YOUR-EMAIL-ID>>
```
An Email would be sent for confirmation on the subscription. 
Acknowledge the subscription to receive mails from topic.

Attach email as a result of this step

On the pipeline details page, choose Edit. 
This opens the editing page for the pipeline. 
Choose + Add stage at the point in the pipeline between Deploy and Production stage, and type a name Approval for the stage.
Choose the + Add action group.
On the Edit action page, do the following:
In Action name, type a name to identify the action like EmailApproval.
In Action provider, choose Manual approval.
In SNS topic ARN, choose the name of the topic created to send notifications for the approval action.
Choose Save.
Save changes to pipeline by clicking Save button on top.
To test your action, choose Release change to process that commit through the pipeline, commit a change to the source specified in the source stage of the pipeline.

Stage 5: Approve or Reject an Approval Action in AWS CodePipeline

If you receive a notification that includes a direct link to an approval action, choose the Approve or reject link, sign in to the console if necessary, and then continue with step 7 below. Otherwise, use all the following steps.

Open the AWS CodePipeline console at https://console.aws.amazon.com/codepipeline/.
On the All Pipelines page, choose the name of the pipeline.
Locate the stage with the approval action.
Hover over the information icon to view the comments and URL, if any. 
The information pop-up message will also display the URL of content for you to review, if one was included.
If a URL was provided, choose the Manual approval link in the action to open the target Web page, and then review the content.
Return to the pipeline details view, and then choose the Review button.
In the Approve or reject the revision window, type comments related to your review, such as why you are approving or rejecting the action, and then choose the Approve or Reject button.

Please provide a screenshot of Approval

Once you approve, the pipeline continues and completes successfully. 

