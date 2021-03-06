## Deploy the Alert Logic Agent Container for ECS

## Before You Begin
To deploy the Alert Logic Agent Container for ECS, you need your unique registration key. 

**To find your unique registration key:**

1. In the Alert Logic console, click the Support Information icon.
2. Click "Details."
3. Copy your unique registration key.

In addition, be sure you install the AWS command line interface (CLI), and ensure you point it to, and configure it for, the appropriate AWS account. For more information about the AWS CLI, see https://aws.amazon.com/cli/. 

## Deploy the Alert Logic Agent Container Task Definition
To deploy the Agent Container for ECS, you must download the task definition file from this repository, and then edit the file to include your Alert Logic unique registration key.

**To deploy and edit the task definition file:**
1. Download the al-agent-ecs.json task definition file from this repository.

2. In the task definition file, update the ```value``` variable with your unique registration key.
   ```
   "environment": [
     {
       "name": "KEY",
       "value": "your_registration_key_here"
     }
   ]
   ```
3. In the Linux command line, type the following command to register your task definition:
   ```
   aws ecs register-task-definition --cli-input-json file://path//to/task-definition/al-agent-ecs.json
   ```
   
## Modify your IAM Policy
Log into the AWS console to ensure the IAM role you use for ECS has the permissions required by the Agent Container. To do so, access the IAM policy document to verify it contains the permissions specified below. 

**To view or modify an IAM policy:** 

1. In the AWS Console, click **IAM,** located under **Security, Identity & Compliance**.
2. From the IAM Management Console, click **Policies**.
3. Select the policy attached to the IAM role you use for ECS, and then click **Edit Policy**.
3. Click **JSON**.
4. Review the policy document for the following permissions. Copy and paste any of the following permissions the policy document lacks into the JSON window:
   ```
         {
             "Effect": "Allow",
             "Action": [
                 "ecs:RegisterContainerInstance",
                 "ecs:DeregisterContainerInstance",
                 "ecs:DiscoverPollEndpoint",
                 "ecs:Submit*",
                 "ecs:Poll",
                 "ecs:StartTask",
                 "ecs:StartTelemetrySession"
             ],
             "Resource": [
                 "*"
             ]
         }
	```
5. Click **Review policy**.
6. On the Review Policy page, click **Save changes**.
   
## Create a Startup Script for Your ECS Instances
You can manually create a startup script to ensure the agent starts when the host starts. 

**To create the startup script:**
1. Copy the following command:
   ```
   #!/bin/bash
   mset -o pipefail

   cluster="MY_CLUSTER" # Enter your cluster name here

   task_def="al-agent-task"
   touch /etc/ecs/ecs.config || {
       echo "Error: it seems like we are not running on an ECS-optimized instance" >&2
       exit 2
   }
   set -ex
   echo ECS_CLUSTER=$cluster >> /etc/ecs/ecs.config
   start ecs
   yum install -y aws-cli jq
   instance_arn=$( curl -f http://localhost:51678/v1/metadata | jq -re .ContainerInstanceArn | awk -F/ '{print $NF}')
   az=$(curl -f http://169.254.169.254/latest/meta-data/placement/availability-zone)
   region=${az:0:${#az} - 1}
   echo "cluster=$cluster az=$az region=$region aws ecs start-task --cluster \
   $cluster --task-definition $task_def --container-instances $instance_arn --region $region" >> /etc/rc.local
   ```
2. Paste the command into the Linux command line. 

## Add Startup to ECS Cluster Instances using CloudFormation 
If you use CloudFormation, you can copy and paste the following into a template to ensure the agent starts when the host starts. 

   ```
   "UserData": {
      "Fn::Base64": {
        "Fn::Join": [
          "",
          [
            "#!/bin/bash -xe\n",
            "echo ECS_CLUSTER=",
            {
              "Ref": "YourECSClusterName"
            },
            " >> /etc/ecs/ecs.config\n",
            "start ecs || true\n",
            "sleep 30\n",
            "instance_arn=$(curl -s http://localhost:51678/v1/metadata | jq -r '. | .ContainerInstanceArn' | awk -F/ '{print $NF}' )\n",
            "az=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)\n",
            "cluster=\"", { "Ref": "YourECSClusterName" }, "\"\n",
            "region=${az:0:${#az} - 1}\n",
            "task_def=\"al-agent-task\"\n",
            "echo \"\n",
            "aws ecs start-task --cluster $cluster --task-definition $task_def --container-instances $instance_arn --region $region >> /etc/rc.local\n"
          ]
        ]
      }
   }
   ```
