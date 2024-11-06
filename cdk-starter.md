# AWS CDK Starter

---

## Pipeline

*inputs*
1. `ordered_region_list`
2. `ordered_environment_list`

The cdk pipeline is defined as an AWS code deploy pipeline in `pipeline-stack.ts`. 
The pipeline is self mutating, meaning changes to the pipeline stack will update the representation of the pipeline at deployment.

The pipeline stack contains the highest level resources. These include resources directly shared between stages and meta resources used to interact with the pipeline.

### Stages
The pipeline has two core stages (not including required build steps) 

#### Shared Stage (Application Network) `{region}-shared`

An instance of the Application Network stage contains all resources that are shared at the regional level for a given application. These resources are used by all (gamma, prod, ...) resources. For example, a single ECR repository houses all application images which are then referenced by the respective stages.
Additionally the network stage contains resources that are prerequisites for downstream stages. The deployment lambda is able to queue requests from Github while the service infrastructure provisioning is in progress.

Promotion to each instance of the Application network stage can be completed in parallel for all regions.

#### Pipeline Stage (Application Environment) `{region}-{environment}`

An instance of the Pipeline Stage contains a set of resources required to run the application. Any number of environments can be specified for a region and they will be sequentially executed. Validation can be configured between stages to prevent the propagation of changes. It is common to have stages for different levels of testing. ie `environments = [beta, gamma, prod]`

Promotion to each instance of the Pipeline stage should be sequential to reduce blast radius of any deployed defects

#### Example Pipeline Stage Configuration
*inputs:* 
1. `ordered_region_list = ['eu', 'na']`
2. `ordered_environment_list = ['gamma', 'prod']`

*output:*
```
│pipeline build stages│ ─┬─> │na-shared│ ──> │eu-gamma│ ──> │eu-prod│ ──> │na-gamma│ ──> │na-prod│
                         └─> │eu-shared│ 
```
![[application-pipeline.svg]]

## Resources

#### Pipeline Stack

- code commit repository - pipeline source repo
- code pipeline - application pipeline
- code build step - build pipeline (self mutating) / application infra source

**Stage:** ApplicationNetworkUtilityStage
**Step:** ManualApprovalStep : Check if the ECR repo is empty

**Stage:** Pipeline Stage (gamma, prod, etc.)

- sns - pipeline approval topic
- sqs - pipeline approval queue
- subscription - queue subs to topic
- [*ApprovePipelineActionLambda*](ApprovePipelineActionLambda)
- lambda event source (sqs) - lambda subs to queue

### ApplicationNetworkStage (_region_-shared)

#### ApplicationNetworkUtilityStack

- log group - cloud trail log group
- cloud trail - cloudwatch cloudtrail
- ecr - image repository
- *ApprovePipelineActionLambda*
- cloud trail rule - ecr image pushed
- cloud trail rule target - lambda subs to ecr image pushed rule

- sns fifo - deployment topic
- sqs fifo - deployment queue
- sqs fifo - deployment queue dlq
- subscription - queue subs to topic
- ssm parameter (write) - deployment queue arn
- sns topic - deployment status topic
- ssm parameter (write) - deployment status topic arn
- *GitHubUploadLambda*
- sns grant publish to github upload lambda

### PipelineStage (_region_-_stage_)

#### Data Stack

- *postgres rds*

#### Application Stack

- *SpringStarterFargateService*
- *BlueGreenPublicEcsHttpAlb*
- *BlueGreenEcsDeploymentGroup*
- ssm parameter (read) - deployment status notification topic arn
- sns (topic from ssm parameter) - deployment status notification topic
- *BeforeInstallVerificationLambda*
- sns grant publish to before install verification lambda
- *AfterAllowTrafficVerificationLambda*
- sns grant publish to after allow traffic verification lambda

- ssm parameter (read) - deployment queue arn
- sqs (queue from ssm parameter) - deployment queue
- *EcsCodeDeployLambda*
- lambda event source (sqs) - lambda subs to queue

- cfn output - dns name
- cfn output - code deploy lambda
- cfn output - ecr repository
- cfn output - db credential secret name

## Connections

INT = Internet
ALB = Application Load Balancer
ECS = Fargate Service
RDS = Postgres Database Instance
NGW = NAT Gateway

1. INT -80---> ALB
2. ALB -8080-> ECS
3. ECS -5432-> RDS
4. ECS -Any--> NGW

## Lambda Constructs
*Shared Role Managed Policies:*
1. Basic Execution Role
2. VPC Access Execution Role


### ApprovePipelineActionLambda
- iam policy statement - ecr permissions and code pipeline execution permissions

*input:* `N/A`
ApprovePipelineActionLambda can be triggered in two ways
1. ecr image upload
2. pipeline approval step status change to 'in-progress'

**when triggered by pipeline approval step:**

check if images are present.
* If no images are present, fail the pipeline manual approval step.
* If images are present, pass the pipeline manual approval step.

**when triggered by ecr image upload:**

trigger a new pipeline execution to clear potential blocked stage

### GitHubUploadLambda
- iam policy statement - ecs list services and sns publish permissions

*input:* 
```
{ 
	"tag": "${{ github.sha }}",
	"emailList": "${{ secrets.APP_EMAIL_LIST }}"
}
```

GitHubUploadLambda is triggered from a [Github workflow](
https://github.com/GabrielCostanzo/prod-ready-spring-starter/blob/mainline/.github/workflows/ecr.yaml)
This lambda is executed after a container image upload to ECR earlier in the workflow.

**When triggered:**
1. if emails provided in Github Secret:`${{ secrets.APP_EMAIL_LIST }}` are not subscribed to the deployment status topic, subscribe emails to the topic.
2. Trigger the deployment SNS topic which will queue a new message with the image tag ie `"tag": "${{ github.sha }}"` indicating the ECR image that should be deployed.

### VerificationLambdas
- iam policy statement - code deploy permissions

1. **BeforeInstallVerificationLambda**
2. **AfterAllowTrafficVerificationLambda**

The verification Lambdas are integrated with the code deploy Application and are specified in the AppSpec file as a targets for the 'Before Install' and 'AfterAllowTraffic' hooks. [AWS documentation for AppSpec hooks](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html#appspec-hooks-ecs)

**When BeforeInstallVerificationLambda Triggered:**
Send a message to all emails subscribed by the GithubUpload Lambda indicating that a new deployment has started with a link to the deployment.

**When AfterAllowTrafficVerificationLambda Triggered:**
Send a message to all emails subscribed by the GithubUpload Lambda indicating that a deployment has finished with a link to the deployment.

### EcsCodeDeployLambda
- iam policy statement - code deploy and ecs permissions

EcsCodeDeployLambda is subscribed to the queue populated by GitHubUploadLambda which publishes messages with a image tag when an ECR image is uploaded. EcsCodeDeployLambda is one of the last resources created in the initial deployment workflow. On initial creation there may already be deployment messages queued. This was a design decision to allow for ECR uploads while the pipeline is still provisioning application resources. The ECS Fargate Service is deployed first with an initial desired count of 0. This prevents the service from triggering failing deployments prior to a desired image being specified.

**When Triggered:**
1. register an ECS task definition with the provided git hash representing the latest ecr image
2. check service to if the desired count is set to zero (init state)
3. if Fargate Service desired count is 0 : update to correct desired count
4. create a code deploy blue green deployment with the new task definition

---

## Custom Constructs

#### rds.DatabaseInstance
**PostgresRds extends rds.DatabaseInstance**
- rds - database instance
- security group - rds sg
- secrets manager secret - master user credentials

#### elbv2.ApplicationLoadBalancer
**BlueGreenPublicEcsHttpAlb**
- elbv2 - application load balancer
- security group - alb sg
- application target group - blue
- applicatoin target group - green

#### codedeploy.EcsDeploymentGroup
**BlueGreenEcsDeploymentGroup**
- codedeploy - ecs deployment group
- codedeploy - ecs application
- iam policy statement - code deploy and s3 permissions

#### ecs.FargateService
**SpringStarterFargateService**
- ecs - fargate service
- security group - ecs sg
- *SpringStarterTaskDefinition*
- ecs - cluster

#### ecs.FargateTaskDefinition
**SpringStarterTaskDefinition extends**
- ecs - fargate task definition
- *EcsTaskExecutionRole*

#### iam.role
**EcsTaskExecutionRole**
- iam - role
- iam service principal - ecs-tasks
- log group - fargate service logs
- iam policy statement - logging permissions
- iam policy statement - credential secret permissions
- iam policy statement - ecr permissions


## Network Configuration


![[network-configuration.svg | 500]]
