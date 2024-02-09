# Simple AWS Pipeline

## Infrastructure Diagram

- **AWS Infras**
![image](https://drive.google.com/uc?export=view&id=1M1pW-_6vOcRuvNHDWqQoqhoN1TqH85XM)

---
## Demo Structures

1. Implementing CodeCommit
2. Implementing Container Registry and CodeBuild
3. Implementing CodeDeploy with ECS
4. Implementing CodePipeline
5. Clean up

---
## Implementing CodeCommit

In this part, you will be creating and configuring a code commit repo as well as configuring access from your local machine. You will need to use SSH or HTTP authentication for using CodeCommit, SSH will be used within this demo. 

#### Generating SSH

> - If you're using Linux or MacOS, following this official guide to generate the SSH key https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html
> - If you're using Windows, following this official guide to generate the SSH key https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-windows.html 

Open your local terminal, 
- Generate ssh key using this command `ssh-keygen -t rsa -b 4096` and call the key `codecommit`, don't set any passphrase for the key.  
- Keys will be stored at `~/.ssh`
- Run `cat codecommit.pub` or open the file with any text editor and copy the output onto your clipboard.  

Now move to the IAM users, [click here](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/users) -> Choose a user with `AdministratorAccess` Permission (For simplicity in this demo) & open that user -> Move to the **Security Credentials** tab
- Scroll down and you'll see **SSH public keys for AWS CodeCommit** section, upload your SSH public key you've just created previously.  
- Copy down the `SSH Key ID` into your clipboard

Now open your local terminal, create a file named `config` at `~/.ssh` and at the top of the file add the following:
```
Host git-codecommit.*.amazonaws.com
  User KEY_ID_YOU_COPIED_ABOVE
  IdentityFile ~/.ssh/codecommit
```

Change the `KEY_ID_YOU_COPIED_ABOVE` placeholder to your SSH key ID. Save the file and run a `chmod 600 ~/.ssh/config` to configure permissions correctly.

#### Testing Connection

Run this command `ssh git-codecommit.us-east-1.amazonaws.com` on your local terminal and if successful it should show something like this.

```
You have successfully authenticated over SSH. You can use Git to interact with AWS CodeCommit. Interactive shells are not supported.Connection to git-codecommit.us-east-1.amazonaws.com closed by remote host.
```
#### Creating The Code Commit Repo

Move to the AWS CodeCommit console, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codecommit/repositories?region=us-east-1) -> Click on `Create repository`.  
- `Repository name` & `Description` = `demo-codecommit-repos-XXX` (where XXX is some unique numbers you pick for yourself)
- Scroll down and create the repository 
- After its created. For **Connection steps**, choose `SSH`  
- Pick the instructions for your specific operating system.  
- Locate the command for `Clone the repository` and copy it into your clipboard.  
- Open your local terminal, move to the folder where you want the repo stored. 
- Then run the clone command in the instruction from aws.

##### Adding The Demo Code

Download this zip file, [click here](https://aws-labs-resources-kng318.s3.amazonaws.com/materials/demofile.zip) -> Unzip the files and copy all contents of it into clone repos in your local machine -> Then from the terminal move into that folder. Run these commands.

```
git add . 
git commit -m "adding demo code"
git push 
```

So now you have a codecommit repo, with some code and are ready to move to the next step. 

---
## Implementing Container Registry and CodeBuild

In this part is where you will configure the Elastic Container Registry and use the codebuild service to build a docker container and push it into this registry.

#### Create a Private Repository

Move to the ECR console, [click here](https://us-east-1.console.aws.amazon.com/ecr/repositories?region=us-east-1) -> Click on the **Repositories** under **Private registry** on the left sidebar -> Then click `Create repository`.  
- `Visibility settings` = `Private`
- `Repository name`= `demo-ecr-repo`
- Scroll down and create the repository
- Note down the URL and name (it should match the above name).

This is the repository that codebuild will store the docker image in, created from the codecommit repo.

#### Setup a CodeBuild Project

Next, we will configure a codebuild project to take what's in the codecommit repo, build a docker image & store it within ECR in the above repository.

Move to the AWS CodeBuild console, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codebuild/projects?region=us-east-1) -> Click on `Create build project`
- In the `Project configuration` section
	- `Project name` = `demo-codebuild`.  
	- Leave all other options in this section as default.
- In the `Source` section
	- `Source Provider` = `AWS CodeCommit`  
	- `Repository` = name the codecommit repo you create
	- `Reference type` =  `Branch` 
	- `Branch` = `main`
	- Leave all other options in this section as default.
- In the `Environment` section
	- `Provisioning model` = `On-demand`
	- `Environment image` = `Managed Image` 
	- `Compute` = `EC2`
	- `Operating system` = `Amazon Linux`  
	- `Runtime(s)` = `Standard` 
	- `Image` = `aws/codebuild/amazonlinux2-x86_64-standard:X.0` where X is the highest number.  
	- `Image version` = `Always use the latest image for this runtime version`  
	- `Service role` = `New Service Role` and leave the default suggested name 
	- Expand `Additional Configuration`
	- Check the `Privileged` box (Because we're creating a docker image)  
	- Scroll down to `Environment variables`, we're going to be adding some environment variables. Add the following:

| Name | value | Type |
| ---- | ---- | ---- |
| AWS_DEFAULT_REGION | us-east-1 | Plain Text |
| AWS_ACCOUNT_ID | REPLACE_YOUR_AWS_ACCOUNT_ID | Plain Text |
| IMAGE_TAG | latest | Plain Text |
| IMAGE_REPO_NAME | REPLACE_YOUR_ECR_REPO_NAME | Plain Text |

- In the `Buidspec` section
	- The buildspec.yml file is what tells codebuild how to build your code.. the steps involved, what things the build needs, any testing and what to do with the output (artifacts).
	- A build project can have build commands included... or, you can point it at a buildspec.yml file, i.e one which is hosted on the same repository as the code.. and that's what you're going to do.
	- Check `Use a buildspec file`  
	- You don't need to enter a name as it will use by default buildspec.yml in the root of the repo. If you want to use a different name, or have the file located elsewhere (i.e in a folder) you need to specify this here.
- In the `Artifacts` section
	- No changes to this section, as we're building a docker image and have no testing yet, this part isn't needed.
- In the `Logs` section
	- This is where the logging is configured, to Cloudwatch logs or S3 (optional).
	- For `Groupname` enter `demo-codebuild` and for `Stream Name` enter `simplepipeline`
- Create the build project

#### Build Security and Permissions

Our build project will be accessing ECR to store the docker image, and we need to ensure it has the permissions to do that. The build process will use an IAM role created by codebuild, so we need to update that roles permissions with ALLOWS for ECR.

Move to the IAM users, [click here](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/users) -> Then click on **Roles** on the left sidebar
- Locate and click the codebuild role have been generated previously
- Move to the **Permissions** tab and add an inline policy for this role
- Select to edit the raw `JSON` and delete the skeleton JSON, replacing it with
```
{
  "Statement": [
	{
	  "Action": [
		"ecr:BatchCheckLayerAvailability",
		"ecr:CompleteLayerUpload",
		"ecr:GetAuthorizationToken",
		"ecr:InitiateLayerUpload",
		"ecr:PutImage",
		"ecr:UploadLayerPart"
	  ],
	  "Resource": "*",
	  "Effect": "Allow"
	}
  ],
  "Version": "2012-10-17"
}
```
- Name it `ECRAccessPermission-Codebuild` and create the policy. This means codebuild can now access ECR.
- Create the policy
#### Test The CodeBuild Project

In the AWS CodeBuild console, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codebuild/projects?region=us-east-1)
- Open `demo-codebuild`  
- Start Build => Start now
- Check progress under phase details tab and build logs tab

---
## Implement Code Deploy

In this stage, you will configure automated deployment of the pipeline application to ECS Fargate
#### Configure a Fargate cluster

Move to the ECS console, [click here](https://us-east-1.console.aws.amazon.com/ecs/v2/home?region=us-east-1) -> Create a Cluster.  
- `Cluster name` = `demopipeline-cluster`. 
- `Infrastructure` = `AWS Fargate (serverless)`
- Scroll down and create the cluster

> Note: If this is the first time you create the ecs cluster, it might dont have the sufficient permission to create and error. Just go to the CloudFormation and delete the stack and re-create the cluster again 

#### Create Task and Container Definitions

In the ECS console -> Move to **Task Definitions** in the left sidebar -> Create a new task definition.  
- `Task definition family` =  `demo-task`.  
- `Launch type` = `AWS Fargate`
- `Operating Architecture` = `Linux/X86_64`  
- `CPU` = `0.5vCPU` 
- `Memory` = `1GB`  
- `Task execution role` = `Create new role`
- `Name` = put the same as for your ECR repo which is `demo-ecr-repo`
- `Image URI` move back to the ECR console and click `Copy URI` next to the latest image.  
- Scroll down to `HealthCheck` section, expand and enter this for `Command` = `CMD-SHELL, curl -f http://localhost/ || exit 1` 
- Scroll to the bottom and click `Create`

#### Deploy to ECS - Create a Service

In the Task you've just created. Click **Deploy** then `Create Service`. 
- In `Environment` section
	- For existing cluster drop-down, choose cluster you've created
	- For Compute options, pick `Launch type` then pick `FARGATE` 
- In `Deployment configuration` section 
	- `Service Name` = `demo-service`  
	- `Desired Tasks` =1  
	- Expand `Deployment Options`, for `Deployment type` pick `rolling update` 
- In `Networking` section  
	- `VPC` = `Default VPC` 
	- `Subnets` make sure all subnets are selected. 
	- `Security Group`, choose `User Existing Security Group` and ensure that `Default`  are selected.  
	- Ensure `Public IP` is `Turned On`  
- Expand `Service auto scaling` section and make sure it's **not** selected.  
- Scroll down and create the service  
- Wait for the service to finished deploying.

The service is now running with the :latest version of the container on ECR, this was done using a manual deployment

#### TEST

After the service is deployed -> Move to the Tasks, click on the task has been created by the service 
- In there, click on the **Networking** tab, copy the `Public IP` then open the `Security groups` (Add an inbound rule allow access http from anywhere IPv4 and save the sg)
- Now paste the IP of the task in browser and see if the page works which mean you have configure the CodeDeploy with ECS successful

---
## Implementing CodePipeline

In this stage of the demo you will create a Code Pipeline which will utilise CodeCommit, CodeBuild and ECS to create a continuous process. The aim is that every time a new commit is made to the codecommit repo, a new docker image is created and pushed to ECR then deploy to ECS. 

#### Create Pipeline

Move to the CodePipeline console, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codepipeline/home?region=us-east-1) -> Click on `create pipeline`  
- `Pipeline name` = `simplepipeline`
- Pipeline type, let default `V2`
- `Service role` =  `New service role` and keep the default name
- Scroll down and expand `Advanced Settings` and make sure that `Default location` is set for the `Artifact store` and `Default AWS managed key` is set for the `Encryption key`  
- Then next

**Source Stage**
- Pick `AWS CodeCommit` as the `Source Provider`  
- Pick `simplepipeline-codecommit` for the repo name 
- Select the branch from the `Branch name` dropdown 
- From `Detection options` pick `Amazon CloudWatch Events` and for `Output artifact format` pick `CodePipeline default`  
- Then next

**Build Stage**
- Pick `AWS CodeBuild` for the `Build provider`  
- Pick `US East (N.Virginia)` for the `Region`  
- Choose the project you created earlier in the `Project name` search box  
- For `Build type` choose `Single Build` 
- Then next

**Deploy Stage**
- Pick `AWS ECS` as the `Deploy provider`
- Pick `US East (N.Virginia)` for the `Region`  
- Choose your cluster in the `Cluster name` dropdown
- Choose your service in the `Service name` dropdown
- Enter `imagedefinitions.json` in the `Image definitions file`
- Then next & scroll down and create the pipeline

#### Test Pipeline

To test the pipeline, you can change the content of the index.html file and push it to the repo

```
git add -A .
git commit -m "updated index.html"
git push
```

Make sure each pipeline step completes
Now move back to the ECS cluster, click on the Tasks, then click on the task has been created by the service 
- In there, click on the **Networking** tab, copy the `Public IP` then open the `Security groups` (Add an inbound rule allow access http from anywhere IPv4 and save the sg)
- Now paste the IP of the task in browser and you'll see the content is changed.

After complete this stage, you have successful implement and configure a pipeline in aws using CodeCommit, CodeBuild and CodePipeline

---
### Clean Up

In this stage you're going to clean up and remove all resources during the demo series.

#### ECS
Move to the ECS console, [click here](https://us-east-1.console.aws.amazon.com/ecs/v2/home?region=us-east-1) -> Click on the cluster, scroll down in the **Services** tab -> 
Choose the service and click **Delete service** -> check the `Force delete` and delete the service

Click on the **Task definitions** in the left sidebar, select the task -> Select all the task revision -> Click **Action** and Deregister

Now move back to the **Clusters**, click on your ECS cluster and delete the cluster

#### CodePipeline
Move to CodePipeline console, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codepipeline/start?region=us-east-1) -> Click on the **Pipelines** on the left sidebar -> Choose your pipeline and delete it

#### CodeBuild
Move to CodeBuild console, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codebuild/start?region=us-east-1) -> Click the **Build projects** on the left sidebar -> Choose your codebuild project and delete it

#### CodeCommit
Move to CodeCommit repositories, [click here](https://us-east-1.console.aws.amazon.com/codesuite/codecommit/repositories?region=us-east-1) -> Choose your repo and delete it

#### S3
Move to S3 Bucket, [click here](https://s3.console.aws.amazon.com/s3/buckets?region=us-east-1 -> Remove the Artifact Bucket

#### ECR
Move to ECR repositories, [click here](https://us-east-1.console.aws.amazon.com/ecr/private-registry/repositories?region=us-east-1) -> Choose the repo and delete it 

#### IAM
Move to IAM, [click here](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/roles) -> Remove Service Role created during the advanced demo

For the SSH key for CodeCommit attached to your user, you can leave it there for future use or delete it
