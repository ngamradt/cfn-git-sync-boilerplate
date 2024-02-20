# CloudFormation Git Sync Boilerplate

## Overview

This project uses CloudFormation templates to set up the new [CloudFormation Git Sync](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/git-sync-concepts-terms.html?icmpid=docs_console_unmapped#git-sync-terms-depoyment-file) feature for an S3 bucket (with web hosting enabled) and a simple CodePipeline deployment.

One of the goals of this project is to leverage [Infrastructure as Code (IaC)](https://en.wikipedia.org/wiki/Infrastructure_as_code) as much as possible, so any process that can be done in CloudFormation is done in CloudFormation.

For more details, please see my [related blog post](https://nealgamradt.com/posts/2024/02/aws-cloudformation-git-sync/index.html) and [YouTube video](https://youtu.be/u9xsfUCoFS8?si=eZT-oxPOnuSKf49b&t=0).

## Synopsis

1. Run some local CloudFormation templates via the AWS Console.
2. An S3 CloudFormation template is then run via CloudFormation Git Sync (CGS).
3. A CodePipeline CloudFormation template is run via CGS.
4. Future updates to the S3 or CloudFormation template are run via CGS.
5. The CodePipeline is a simple two-step CodePipeline:
    - Grab the source of this repository.
    - Deploy the source of this repository to S3.
6. Once the source is deployed, we have an simple HTTP website which was mostly created using IaC.

> [!NOTE]
> Though I understand the reasoning for leveraging CodeStar connections, I dislike them because there is no way to complete the connection without going to your web browser and using [ClickOps](https://docs.cloudposse.com/glossary/clickops/). When working with the cloud, any process which _requires_ interaction with web console is a failed process.  In this case, it isn't entirely the fault of AWS, but it is still a failure.

## Outcome

Here is the [live demo website](http://ngamradt-demo-website-us-east-2.s3-website.us-east-2.amazonaws.com) that is using S3 website hosting.

## Implementation

Follow these steps in the exact order, and you should be good-to-go.

> [!NOTE]
> To keep things simple, we will be running some of these templates using the AWS Console and some basic ClickOps to run the templates in the CloudFormation console.  You could also use the [AWS CLI](https://aws.amazon.com/cli/) to run these templates, but we won't be covering that for now.

1. Make your own local copy of this repository using the ["Use this template" button](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template).
2. Log into your [GitHub account](https://github.com).
3. Log into the AWS Console for your [AWS Account](https://aws.amazon.com). 
4. Select the region where you want to set up your infrastructure.
    - for this demo, the Ohio region was used.
5. Go to the CloudFormation console page.
6. Deploy the [IAM Role](website/iac/cfn/iam/role/git-sync.yaml) template that will be used by Git Sync.
    - If you haven't deployed a stack using a template before, [follow the instructions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) in the AWS documentation.
    - This first parameter in the template has a suggested stack name, please copy and paste that name for the stack.
    - Just go with all the defaults for deploying the template.
    - You will need to check a checkbox on the last step to confirm you understand the risks of creating an IAM role.
7. Deploy the [CodeStar Connection](website/iac/cfn/codestar/connection.yaml) template.
    - Follow the same process as in step 4.
8. You will then need to go to the [AWS Console CodeStar Connections page](https://us-east-2.console.aws.amazon.com/codesuite/settings/connections?region=us-east-2&connections-meta=eyJmIjp7InRleHQiOiIifSwicyI6e30sIm4iOjIwLCJpIjowfQ) and use ClickOps to complete the configuration of the pending connection.
    - The example URL is in the Ohio region, so you might need to change regions if you set up your connection in a different region.
9. Once the CodeStar Connection is fully configured, we are set to connect the final pieces.
10. The Git Sync configurations need to point to placeholder stacks.  By default, these stacks do almost nothing.  They establishe an [AWS::CloudFormation::WaitConditionHandle](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-waitconditionhandle.html) resource.
    - I assume this is the most lightweight resource that CloudFormation has, which is why it is used as the placeholder resource (this is speculation).
11. Using the AWS Console, deploy the [S3 Website Sync Parent](website/iac/cfn/codestar/parent/s3-website-git-sync.yaml) template.
    - This is the placeholder stack that the first Git Sync Configuration will be attach to.
    - Make sure to use the "Suggested Stack Name" parameter value as your actual stack name.
12. Using the AWS Console, deploy the [S3 Website Sync](website/iac/cfn/codestar/sync-configuration/s3-website.yaml) template.
    - This template will establish the Git Sync connection to the parent stack from the previous step.
    - Make sure to use the "Suggested Stack Name" parameter value as your actual stack name.
    - The [S3 Website Git Sync configuration file](website/deployment/s3/website.yaml) is referenced to link everything together.
13. If everything is set up correctly, the CGS process should run for the S3 bucket.
    - The Git Sync feature will pull down the template from your repository.
    - The template will be run in the parent stack.
    - If everything is successful, the placeholder resource will be replaced with the resources from the CGS template.
14. Using the AWS Console, deploy the [CodePipeline Deploy Parent](website/iac/cfn/codestar/parent/codepipeline-deploy-git-sync.yaml) template.
    - This is the placeholder stack that the second Git Sync Configuration will be attach to.
    - Make sure to use the "Suggested Stack Name" parameter value as your actual stack name.
15. Using the AWS Console, deploy the [CodePipeline Deploy Sync](website/iac/cfn/codestar/sync-configuration/codepipeline-deploy.yaml) template.
    - This template will establish the Git Sync connection to the parent stack from the previous step.
    - Make sure to use the "Suggested Stack Name" parameter value as your actual stack name.
    - The [CodePipeline Deploy Git Sync configuration file](website/deployment/codepipeline/deploy.yaml) is referenced to link everything together.
    - The Git Sync feature will pull down the template from your repository.
    - The template will be run in the parent stack.
    - If everything is successful, the placeholder resource will be replaced with the resources from the CGS template.
17. The CodePipeline will now take all of the files from your reposiotry and deploy them to the root of your S3 bucket.
18. You can find the URL for your bucket in either the AWS S3 Console or by looking at the CloudFormation outputs for the S3 bucket stack.
19. Load the URL for your bucket website into your browser, you should see the contents of the [index.html](index.html) file from your repository load.
20. You can now do the following things:
    - Update your [S3 bucket file](website/iac/cfn/s3/website.yaml) or [S3 Website stack deployment file](website/deployment/s3/website.yaml) and commit it to the `main` branch.  Once committed, the stack should be automatically updated with those changes.
    - Update your [CodePipeline deploy file](website/deployment/codepipeline/deploy.yaml) or [CodePipeline dstack deployment file](website/deployment/codepipeline/deploy.yaml) and commit it to the `main` branch.  Once committed, the stack should be automatically updated with those changes.
    - Make changes to the `index.html` or `error.html` files and commit them to the `main` branch.  Once committed, the CodePipeline will deploy these (and any other files in the repository) to the S3 bucket.

## Final Thoughts

The fact that you can run updates to a CloudFormation stack directly from GitHub file changes is a useful feature.  Previously, you would have to leverage something like CodePipeline to deploy templates if you wanted to use the concept of [GitOps](https://en.wikipedia.org/wiki/DevOps#GitOps), so this feature can make things easier.  However, the current process to hook this all togetther is combersome (even with moving as much of the process as possible into CloudFormation templates).  You need to leverage too much ClickOps (or alternatively, the AWS CLI) to do the initial setup.

My recommendation would be that the CodeStar connection get enhanced to just read the stack deployment files and establish the needed Git Sync connections in CloudFormation.  I am sure this would take a lot of work and there would likely be security concerns, but that would make this whole feature much more powerful, flexible, and easy to use.

For now, I could still see a lot of use cases for this feature even in this early state.  For instance, this could be useful in situations where you don't have many CloudFormation stacks in your repository and you don't want to bother with a CodePipeline to deploy those few templates.


