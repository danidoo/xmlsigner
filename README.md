![workshop logo](https://github.com/ldecaro/xmlsigner/blob/master/images/hsm-fargate.png)

# Signer Microservice - Digital Invoice Signing

A microservice for signing digital invoices created using the XML format. This microservice is also able to digitaly sign any XML document. In payment industries, more specifically in Brazil, law requires all invoices to be created in digital format (XML) and to be digitally signed before sent to customers. An example is the NF-e or Nota Fiscal Eletrônica.

Signer creates a microservice in AWS using a Serverless Container architecture enabling you to, via IaaC, create and deploy a microservice that leverages the use of HTTP protocol to sign any file in XML format.

The XML Signature process uses a Cloud Hardware Secure Module ([AWS CloudHSM](https://aws.amazon.com/pt/cloudhsm/)) provisioned and configured in the Cloud. The microservice is written in Java using Amazon Corretto and cryptography process is enhanced with the use of the [Amazon Corretto Crypto Provider (ACCP)](https://aws.amazon.com/blogs/opensource/introducing-amazon-corretto-crypto-provider-accp/).

Care about performance? If we consider p99 requests the Fargate Container deployed with this project can sign a 6KB document in around 15ms (Tested with a real invoice from Brazil - NF-e)

## Proposed Architecture

This project includes the AWS CloudFormation scripts to create, configure, build and deploy the following infrastructure:
  * Network
  * CloudHSM  
  * Java Container for XML signature process
  * [ECS Fargate Service](https://aws.amazon.com/pt/fargate/)


The proposed architecture is:

![proposed solution](images/architecture.jpg)

## Quick links

1. [Installation](#Installation)
2. [Using the Signer Container](#Using-the-container)
3. [Troubleshoot](#Troubleshoot)

## Installation

You can have your microservice deployed and running in three automatic steps:
Note: Total time for this setup is around 25 minutes. Cost of this infrastructure for testing purposes is around US$ 3.50 per hour (us-east-1 / North Virginia). **You need a region where service CodeBuild is running**

### Deploy Network and CloudHSM
  
|Deploy | Region |
|:---:|:---:|
|[![launch stack](/images/launch_stack_button.png)][us-east-1-hsm-signer] | US East (N. Virginia)|
|[![launch stack](/images/launch_stack_button.png)][us-east-2-hsm-signer] | US East (Ohio)|
|[![launch stack](/images/launch_stack_button.png)][us-west-2-hsm-signer] | US West (Oregon)|

 
[Troubleshoot](#Troubleshooting-HSM-Installation)
 
**Details:** Just watch the show: You only need to change parameters in case the networks in the script overlap your networks. After the script finishes executing, a Step Function starts running automatically and to create and configures your CloudHSM. After the Step Function finishes you can run step #2: Build Container.

This script creates a Step Function with name staring with **LaunchCloudHSMCluster**. In the AWS Console, go to Step Functions and find that function. What for the execution to finish and move to the next step.

 
### Build Container
  
|Deploy | Region |
|:---:|:---:|
|[![launch stack](/images/launch_stack_button.png)][us-east-1-container-signer] | US East (N. Virginia)|
|[![launch stack](/images/launch_stack_button.png)][us-east-2-container-signer] | US East (Ohio)|
|[![launch stack](/images/launch_stack_button.png)][us-west-2-container-signer] | US West (Oregon)|


[Troubleshoot](#Troubleshooting-Container-Build)

**Details:** There are 4 parameters you need to define: HSM User, Password, Partiton and IP. You can choose any user and password. (HSM Master user password will be the same of your user password). To find out the HSM IP and partition is simple. In the AWS console type CloudHSM and. You will find your cluster. Click on the HSM cluster record to visualize the details. In the details screen **HSM ID** is the **Partition** and the IP is next to it.

 ![cloudhsm ip](/images/hsm-parameters.png)

After you run the CloudFormation script you should use the AWS Console and look for the service AWS CodeBuild. Find the build project named **xmlsigner** and start the build (image below). It will take around 10 minutes to finish. Now you can deploy the container in the ECS Fargate service.

 ![codebuild start](/images/start-build.png)


  ### Deploy Container on Fargate

|Deploy | Region |
|:---:|:---:|
|[![launch stack](/images/launch_stack_button.png)][us-east-1-ecs-signer] | US East (N. Virginia)|
|[![launch stack](/images/launch_stack_button.png)][us-east-2-ecs-signer] | US East (Ohio)|
|[![launch stack](/images/launch_stack_button.png)][us-west-2-ecs-signer] | US West (Oregon)|

Just follow the steps and wait for the script to finish executing. It is ready for use!

[Troubleshoot](#Troubleshooting-ECS-Deploy)

## Using the container

The container executes the following operations:

 * List Keys
 * Create Key
 * Sign
 * Sign embedding a certificate
 * Validate
 
 **IMPORTANT:** Although the container does not have ssh installed please do not use it in production. Security is beyond of the scope of this project. Please note there is no SSL enabled.
 
 After the script finishes executing there is a tab named output. That tab contains all the URLs of your microservice. You can create keys, list keys, delete keys, sign and validate a document. 
 
 ![proposed solution](images/cf-outputs.png)

Signing process may be executed using a self signed certificate or not. Everything is automatic. When you create a key a self signed certificate is already created for your convenience.

You choose whether to sign embedding a X.509 certificate into your invoice or not.

 
 Inside the folder **run** of this project there is a file name **certdata.json** and you can use it to define properties of your self signed certificate when creating keys. Please find below the commands you should use to operate the **xmlsigner** container:
 
### Store the service URL in an environment variable:
(remove the // at the end of the url)
```
URL=<alb-url>
```

### List existing keys:
 
```
 $ curl $URL/xml/listKeys
```

### Create keys:

```
 $ curl --data "@run/certdata.json" $URL/xml/create/<my-key-label> -X POST -H "Content-Type: text/plain"
```

### Sign XML Document

```
 $ curl --data "@run/sample.xml" $URL/xml/sign/<my-key-label> -X POST -H "Content-Type: application/xml"
```

### Sign and Embed Certificate

```
 $ curl --data "@run/sample.xml" $URL/xml/sign-cert/<my-key-label> -X POST -H "Content-Type: application/xml"
```

### Validate Signed Document

```
 $ curl --data "@run/<your-signed-xml>" <alb-url>/xml/validate -X POST -H "Content-Type: application/xml"
```

### Delete a key in the HSM

According to [this](https://docs.aws.amazon.com/cloudhsm/latest/userguide/alternative-keystore.html) documentation deleting keys is not supported by CloudHSM Keystore. Use the CloudHSM's **key_mgmt_util** tool.


[us-east-1-hsm-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=SignerHSM&templateURL=https://s3.amazonaws.com/signer-hsm/SignerHSM.yaml
[us-east-2-hsm-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=SignerHSM&templateURL=https://s3.amazonaws.com/signer-hsm/SignerHSM.yaml
[us-west-2-hsm-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=SignerHSM&templateURL=https://s3.amazonaws.com/signer-hsm/SignerHSM.yaml
[sa-east-1-hsm-signer]: https://console.aws.amazon.com/cloudformation/home?region=sa-east-1#/stacks/new?stackName=SignerHSM&templateURL=https://s3.amazonaws.com/signer-hsm/SignerHSM.yaml
[us-east-1-container-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=SignerContainer&templateURL=https://s3.amazonaws.com/signer-hsm/SignerContainer.yaml
[us-east-2-container-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=SignerContainer&templateURL=https://s3.amazonaws.com/signer-hsm/SignerContainer.yaml
[us-west-2-container-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=SignerContainer&templateURL=https://s3.amazonaws.com/signer-hsm/SignerContainer.yaml
[sa-east-1-container-signer]: https://console.aws.amazon.com/cloudformation/home?region=sa-east-1#/stacks/new?stackName=SignerContainer&templateURL=https://s3.amazonaws.com/signer-hsm/SignerContainer.yaml
[us-east-1-ecs-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=SignerECS&templateURL=https://s3.amazonaws.com/signer-hsm/SignerECS.yaml
[us-east-2-ecs-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=SignerECS&templateURL=https://s3.amazonaws.com/signer-hsm/SignerECS.yaml
[us-west-2-ecs-signer]: https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=SignerECS&templateURL=https://s3.amazonaws.com/signer-hsm/SignerECS.yaml
[sa-east-1-ecs-signer]: https://console.aws.amazon.com/cloudformation/home?region=sa-east-1#/stacks/new?stackName=SignerECS&templateURL=https://s3.amazonaws.com/signer-hsm/SignerECS.yaml

## Troubleshoot

### Troubleshooting HSM Installation

 Due to the chosen level of automation you might find problems in case the AZ automatically chosen to install the HSM doesn't have it available. Remember: you are just testing...pick another region and give it a try. In last case, you might want to check the Cloud Formation script from folder cf named SignerHSM.yaml and change the default AZ number.
 
 ### Troubleshooting Container Build
 
 In this step the CodeBuild tries to connect to github to fetch the source code of the container.  In case you don't have any credentials configured you will get the error below from CloudFormation. That means you need to configure a Git account in the Code Build. Relax..that is easy! Open the tab events in the failed cloudformation run and make sure your error looks like the one below.
 
  ![git login failed](/images/create-build-failed.png)
 
 Setup CodeBuild to connect to Github: In the CodeBuild service pretend you are creating a new build. Choose Create Build Project. In the configuration screen there is section named Source. Choose Github, and connect using OAuth. Once you connect press CANCEL and run the script again (validate you get the message **you are connected to Github using OAuth**). CodeBuild now knows how to fetch code from Github.
 
 **Delete the stack Signer container on CloudFormation and click again in the button above to run the stack**
 
 ![setup git credentials](/images/configure-git.png)
 
 ### Troubleshooting ECS Deploy

 In case you never used the ECS Service you might be missing a service linked role. In case the error you got from the SignerECS Stack is the one from the image below, please refer to [this](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using-service-linked-roles.html) documentation for details.
 
  ![linked role missing](images/service-linked-role.png)
 
 To fix the issue run the command below using the aws cli, delete the SignerECS stack and click again in the button above to redeploy the SignerECS stack.
 ```
 $ aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
 ```
