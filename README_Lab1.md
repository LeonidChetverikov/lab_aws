Build project on the cloud

AWS Cloud9 IDE - Set up
AWS Cloud9 is a cloud-based integrated development environment (IDE) that lets you write, run, and debug your code with just a browser. 
It includes a code editor, debugger, and terminal. 
Cloud9 comes pre-packaged with essential tools for popular programming languages and the AWS Command Line Interface (CLI) pre-installed so you don't need to install files or configure your laptop for this workshop. 
Your Cloud9 environment will have access to the same AWS resources as the user with which you logged into the AWS Management Console.


Step-by-step Instructions

Go to the AWS Management Console, click Services then select Cloud9 under Developer Tools.

Click Create environment.

Enter MyDevEnvironment into Name and optionally provide a Description.

Click Next step.

You may leave Environment settings at their defaults of launching a new t2.micro EC2 instance which will be paused after 30 minutes of inactivity.

Click Next step.

Review the environment settings and click Create environment. 
It will take several minutes for your environment to be provisioned and prepared.
Once ready, your IDE will open to a welcome screen. 
Below that, you should see a terminal prompt similar to: setup 
You can run AWS CLI commands in here just like you would on your local computer. Verify that your user is logged in by running the following command.
user:~/environment $ aws sts get-caller-identity
You'll see output indicating your account and user information:


{

    "Account": "123456789012",

    "UserId": "AKIAI44QH8DHBEXAMPLE",

    "Arn": "arn:aws:iam::123456789012:user/user"

}

Stage 1: Create an AWS CodeCommit Repository

To create the AWS CodeCommit repository (console)

Open the AWS CodeCommit console at https://console.aws.amazon.com/codecommit.

On the Welcome page, choose Get Started Now. (If a Dashboard page appears instead, choose Create repository.)

On the Create repository page, in the Repository name box, type WebAppRepo.

In the Description box, type My demonstration repository.

Choose Create repository to create an empty AWS CodeCommit repository named WebAppRepo.

The remaining steps in this tutorial assume you have named your AWS CodeCommit repository WebAppRepo. 
If you use a name other than WebAppRepo, be sure to use it throughout this tutorial.
 For more information about creating repositories, including how to create a repository from the terminal or command line, see Create a Repository.

Stage 2: Clone the Repo

In this step, you will connect to the source repository created in the previous step. Here, you use Git to clone and initialize a copy of your empty AWS CodeCommit repository. Then you specify the user name and email address used to annotate your commits.

From CodeCommit Console, you can get the https clone url link for your repo.
Go to Cloud9 IDE terminal prompt
Run git clone to pull down a copy of the repository into the local repo:
```sh
user:~/environment $ git clone https://git-codecommit.  <<YOUR-REGION>>  .amazonaws.com/v1/repos/WebAppRepo
```

Provide your Git HTTPs credential when prompted. You would be seeing the following message if cloning is successful. warning: You appear to have cloned an empty repository.

Stage 3: Commit changes to Remote Repo

Download the Sample Web App Archive by running the following command from IDE terminal.

user:~/environment $ wget https://s3.amazonaws.com/devops-workshop-0526-2051/v1/Web-App-Archive.zip

Unarchive and copy all the contents of the unarchived folder to your local repo folder.

user:~/environment $ unzip Web-App-Archive.zip

user:~/environment $ mv -v Web-App-Archive/* WebAppRepo/

After moving the files, your local repo should like the one below. cloud9 3. Change the directory to your local repo folder. Run git add to stage the change:

user:~/environment $ cd WebAppRepo

user:~/environment/WebAppRepo/ $ git add *

Run git commit to commit the change:

user:~/environment/WebAppRepo/ $ git commit -m "Initial Commit"

Run git config credential to store the credential.

user:~/environment/WebAppRepo/ $ git config credential.helper store

Run git push to push your commit through the default remote name Git uses for your AWS CodeCommit repository (origin), from the default branch in your local repo (master):

user:~/environment/WebAppRepo/ $ git push -u origin master

Provide your Git HTTPs credential when prompted. Credential helper will store it, hence you won't be asked again for subsequent push.


Stage 4: Prepare Build Service
First, let us create the necessary roles required to finish labs. Run the CloudFormation stack to create service roles. Ensure you are launching it in the same region as your AWS CodeCommit repo.
user:~/environment/WebAppRepo (master) $ aws cloudformation create-stack --stack-name DevopsWorkshop-roles \
--template-body https://s3.amazonaws.com/devops-workshop-0526-2051/v1/01-aws-devops-workshop-roles.template \
--capabilities CAPABILITY_IAM

TASK:
Please add descrip[tion of service roles created. Add the output of the stack.


How to ddo task:

user:~/environment/WebAppRepo (master) $ sudo yum -y install jq

user:~/environment/WebAppRepo (master) $ echo YOUR-BuildRole-ARN: $(aws cloudformation describe-stacks --stack-name DevopsWorkshop-roles | jq -r '.Stacks[0].Outputs[]|select(.OutputKey=="CodeBuildRoleArn")|.OutputValue')

user:~/environment/WebAppRepo (master) $ echo YOUR-S3-OUTPUT-BUCKET-NAME: $(aws cloudformation describe-stacks --stack-name DevopsWorkshop-roles | jq -r '.Stacks[0].Outputs[]|select(.OutputKey=="S3BucketName")|.OutputValue')

Let us create CodeBuild project from CLI. 
To create the build project using AWS CLI, we need JSON-formatted input. Create a json file named 'create-project.json' under 'MyDevEnvironment'.  Copy the content below to create-project.json. 
(Replace the placeholders marked with <<>> with values for BuildRole ARN, S3 Output Bucket and region from the previous step.)
```js
{

  "name": "devops-webapp-project",

  "source": {

    "type": "CODECOMMIT",

    "location": "https://git-codecommit.<<REPLACE-YOUR-REGION-ID>>.amazonaws.com/v1/repos/WebAppRepo"

  },

  "artifacts": {

    "type": "S3",

    "location": "<<REPLACE-YOUR-S3-OUTPUT-BUCKET-NAME>>",

    "packaging": "ZIP",

    "name": "WebAppOutputArtifact.zip"

  },

  "environment": {

    "type": "LINUX_CONTAINER",

    "image": "aws/codebuild/java:openjdk-8",

    "computeType": "BUILD_GENERAL1_SMALL"

  },

  "serviceRole": "<<REPLACE-YOUR-BuildRole-ARN>>"

}
```

Switch to the directory that contains the file you just saved, and run the create-project command:

user:~/environment $ aws codebuild create-project --cli-input-json file://create-project.json


Stage 5: Let's build the code on cloud

A build spec is a collection of build commands and related settings in YAML format, that AWS CodeBuild uses to run a build. 
Create a file namely, buildspec.yml under WebAppRepo folder. Copy the content below to the file and save it.

version: 0.1

```yaml
phases:
  install:
    commands:
      - echo Nothing to do in the install phase...
  pre_build:
    commands:
      - echo Nothing to do in the pre_build phase...
  build:
    commands:
      - echo Build started on `date`
      - mvn install
  post_build:
    commands:
      - echo Build completed on `date`
artifacts:
  files:
    - target/javawebdemo.war
  discard-paths: no
  ```

Commit & push the build specification file to repository

Run the start-build command:

user:~/environment/WebAppRepo (master) $ aws codebuild start-build --project-name devops-webapp-project

PLease add test command into JSON and provide JSON to proctor review.


user:~/environment/WebAppRepo (master) $ aws codebuild batch-get-builds --ids <<ID>>
