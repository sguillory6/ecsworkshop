---
title: "Deploy ECS Cluster Auto Scaling"
disableToc: true
hidden: true
---
 
### Enable a Capacity Provider per AZ on the cluster

This lab assumes you have completed the previous lab on EC2 ECS Cluster Auto Scaling. While will be updating
the infracture we deployed in that lab, we will work with a different CDK script that has the necessary changes.
Navigate back to the repo where we create and manage the platform but enter the sub-directory cdk-az for this lab.

```bash
cd ~/environment/container-demo/cdk-az
```

You should be familiar the section of code that says `###### CAPACITY PROVIDERS SECTION #####` at this point.
Modify that section to look like this:


```python
        ###### CAPACITY PROVIDERS SECTION #####
        # Adding EC2 capacity to the ECS Cluster
        self.asg1 = self.ecs_cluster.add_capacity(
            "ECSEC2Capacity1",
            instance_type=aws_ec2.InstanceType(instance_type_identifier='t3.small'),
            min_capacity=0,
            max_capacity=10,
            vpc_subnets=aws_ec2.SubnetSelection(availability_zones=['us-east-1a'])
        )
        
        self.asg2 = self.ecs_cluster.add_capacity(
            "ECSEC2Capacity2",
            instance_type=aws_ec2.InstanceType(instance_type_identifier='t3.small'),
            min_capacity=0,
            max_capacity=10,
            vpc_subnets=aws_ec2.SubnetSelection(availability_zones=['us-east-1b'])
        )

        self.asg3 = self.ecs_cluster.add_capacity(
            "ECSEC2Capacity3",
            instance_type=aws_ec2.InstanceType(instance_type_identifier='t3.small'),
            min_capacity=0,
            max_capacity=10,
            vpc_subnets=aws_ec2.SubnetSelection(availability_zones=['us-east-1c'])
        )

        core.CfnOutput(self, "EC2AutoScalingGroupName1", value=self.asg1.auto_scaling_group_name, export_name="EC2ASGName1")
        core.CfnOutput(self, "EC2AutoScalingGroupName2", value=self.asg2.auto_scaling_group_name, export_name="EC2ASGName2")
        core.CfnOutput(self, "EC2AutoScalingGroupName3", value=self.asg3.auto_scaling_group_name, export_name="EC2ASGName3")
	##### END CAPACITY PROVIDER SECTION #####
```

Note that we are now adding three auto scaling groups to the cluster, but we are tying each ASG to a single, distinct AZ. 

Now, update the cluster using the cdk.

```bash
cdk deploy --require-approval never
```


These changes to this section of code will provide the necessary components to balance our tasks across AZs regardless of the state of the infrastructure.
We can now associate a capacity provider with each ASG, giving them equal weights, so that tasks will be scheduled accordingly. For more information, see
the [official cdk documentation.](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs/Cluster.html#aws_cdk.aws_ecs.Cluster.add_capacity)

Once the deployment is complete, let's move back to the capacity provider demo repo, this time to the ec2-az directory, and start work on setting up balanced
AZ scheduling strategy for our tasks.

```bash
cd ~/environment/ecsdemo-capacityproviders/ec2-az
```

### Enable Balanced AZ Cluster Auto Scaling

As we did in the previous section, we are going to once again create a capacity provider. This time however, we will be creating three capacity providers
to enable balanced AZ  managed cluster auto scaling.

Before we do that however, there is a manual step we have to perform in the console. As you did in the previous lab, please go to the EC2 service page
in the console and navigate to the Auto Scaling Groups section. You should see three Auto Scaling Groups whose names start with ecsworkshop-base. As you
did in the previous lab, enable Instance Protection on all three ASGs.

Now let's create those Capacity Providers.

```bash
# Get the required cluster values needed when creating the on-demand capacity provider
export asg_name1=$(aws cloudformation describe-stacks --stack-name ecsworkshop-base | jq -r '.Stacks[].Outputs[] | select(.ExportName | contains("EC2ASGName1"))| .OutputValue')
export asg_arn1=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $asg_name1 | jq .AutoScalingGroups[].AutoScalingGroupARN)
export capacity_provider_name1=$(echo "EC2$(date +'%s')-1")

# Creating capacity provider
aws ecs create-capacity-provider \
     --name $capacity_provider_name1 \
     --auto-scaling-group-provider autoScalingGroupArn="$asg_arn1",managedScaling=\{status="ENABLED",targetCapacity=80\},managedTerminationProtection="ENABLED" \
     --region $AWS_REGION

An error occurred (ClientException) when calling the CreateCapacityProvider operation: 
The managed termination protection setting for the capacity provider is invalid. To enable
 managed termination protection for a capacity provider, the Auto Scaling group must have 
 instance protection from scale in enabled.
 
 
# Get the required cluster values needed when creating the spot capacity provider
export asg_name2=$(aws cloudformation describe-stacks --stack-name ecsworkshop-base | jq -r '.Stacks[].Outputs[] | select(.ExportName | contains("EC2ASGName2"))| .OutputValue')
export asg_arn2=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $asg_name2 | jq .AutoScalingGroups[].AutoScalingGroupARN)
export capacity_provider_name2=$(echo "EC2$(date +'%s')-2")

# Creating capacity provider
aws ecs create-capacity-provider \
     --name $capacity_provider_name2 \
     --auto-scaling-group-provider autoScalingGroupArn="$asg_arn2",managedScaling=\{status="ENABLED",targetCapacity=80\},managedTerminationProtection="ENABLED" \
     --region $AWS_REGION

# Get the required cluster values needed when creating the spot capacity provider
export asg_name3=$(aws cloudformation describe-stacks --stack-name ecsworkshop-base | jq -r '.Stacks[].Outputs[] | select(.ExportName | contains("EC2ASGName3"))| .OutputValue')
export asg_arn3=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $asg_name3 | jq .AutoScalingGroups[].AutoScalingGroupARN)
export capacity_provider_name3=$(echo "EC2$(date +'%s')-3")

# Creating capacity provider
aws ecs create-capacity-provider \
     --name $capacity_provider_name3 \
     --auto-scaling-group-provider autoScalingGroupArn="$asg_arn3",managedScaling=\{status="ENABLED",targetCapacity=80\},managedTerminationProtection="ENABLED" \
     --region $AWS_REGION
```

- *Note*: If you get an error that the capacity provider already exists because you've created it in the workshop before, just move on to the next step.

- *Note*: If you get an error that the managed termination protection setting for the capacity provider is invalid, go back and make sure that Instance
Protection is set to "Protect From Scale In" on both of your ASGs.

- In order to create a capacity provider with cluster auto scaling enabled, we need to have an auto scaling group created prior. We did this earlier in
this section when we added the EC2 capacity to the ECS cluster. We run a couple of cli calls to get the autoscale group details which is required for the
next command where we create the capacity provider.

- The next command is creating a capacity provider via the AWS CLI. Let's look at the parameters and explain their purpose:

  - `--name`: This is the human readable name for the capacity provider that we are creating.
  - `--auto-scaling-group-provider`: There is quite a bit here, let's unpack one by one:
  
    - `autoScalingGroupArn`: The ARN of the auto scaling group for the cluster autoscaler to use.
    - `managedScaling`: This is where we enable/disable cluster auto scaling. We also set `targetCapacity`, which determines at what point in cluster utilization do we want the auto scaler to take action.
    - `managedTerminationProtection`: Enable this parameter if you want to ensure that prior to an EC2 instance being terminated (for scale-in actions), the auto scaler will only terminate instances that are not running tasks.

Now that we have the capacity providers created, we need to associate them with the ECS Cluster.

```bash
aws ecs put-cluster-capacity-providers \
--cluster container-demo \
--capacity-providers $capacity_provider_name1 $capacity_provider_name2 $capacity_provider_name3 \
--default-capacity-provider-strategy capacityProvider=$capacity_provider_name1,weight=1 capacityProvider=$capacity_provider_name2,weight=1 capacityProvider=$capacity_provider_name3,weight=1
```

You will get a json response indicating that the cluster update has been applied. Note that we are associating one capacity provider per ASG of the 
the cluster, assigning capacity provider a weight of 1. This should result in tasks being distributing evenly across the ASGs and thus evenly across
the AZs. Now it's time to deploy a service and test this functionality out!


### Deploy an EC2 backed ECS service

First, as we've done previously, we will run a `diff` on what presently exists, and what will be deployed via the cdk.

```bash
cdk diff
```

Review the changes, you should see all new resources being created for this service as it hasn't been deployed yet. So on that note, let's deploy it!

```bash
cdk deploy --require-approval never
```

Once the service is deployed, take note of the load balancer URL output. Copy that and paste it into the browser.


### Examine the current deployment

What we did above was deploy a service that runs three tasks. With the current EC2 instances that are registered to the cluster, there is more
than enough capacity to run our service.

Navigate to the console, and select the container-demo cluster. Click the ECS Instances tab, and review the current capacity. Note that the three tasks are
distributed across three instances spread across three AZs.

![clustercapacity](/images/ec2-ecs-az-3.png)

As you can see, we have plenty of capacity to support a few more tasks. But what happens if we need to run more tasks than what we have current capacity to run?

- As operators of the cluster, we have to think about how to scale the backend EC2 infrastructure that runs our tasks (of course, this is for EC2 backed
tasks, with Fargate, this is not a concern of the operator as the EC2 backend is obfuscated). 
- We also have to be mindful of scaling the application. It's a good practice to enable autoscaling on the services to ensure the application can meet the
demand of it's end users.

This poses a challenge when operating an EC2 backed cluster, as scaling needs to be considered in two places. With the cluster autoscaling being enabled,
now the orchestrator will scale the backend infrastucture to meet the demand of the application. This empowers teams that need EC2 backed tasks, to think
"application first", rather than think about scaling the infrastructure.


### Scale the service beyond the current capacity available

We'll do it live! Go back into the deployment configuration (`~/environment/ecsdemo-capacityproviders/ec2-az/app.py`), and do the following:

Change the desired_count parameter from `3` to `12`.

```python
        self.load_balanced_service = aws_ecs_patterns.ApplicationLoadBalancedEc2Service(
            self, "EC2CapacityProviderService",
            service_name='ecsdemo-capacityproviders-ec2',
            cluster=self.base_platform.ecs_cluster,
            cpu=256,
            memory_limit_mib=512,
            #desired_count=3,
            desired_count=12,
            public_load_balancer=True,
            task_image_options=self.task_image,
        )
```

Now, save the changes, and let's deploy.

```bash
cdk deploy --require-approval never
```

Let's walk through what we did and what is happening.

- We are modifying our task count for the service to go from three, to twelve. This will stretch us beyond the capacity that presently
exists for the cluster.
- The capacity providers assigned to the cluster will recognize that the target capacity of the total cluster resources is above 80%.
This will trigger an autoscaling event to scale EC2 to get the capacity back to 80% or under.
- The nine new tasks should be distributed evenly across the three AZs

Over the course of the next couple of minutes, behind the scenes a target tracking scaling policy is triggering a Cloudwatch alarm to
enable the auto scale group to scale out. Shortly after, we will begin to see new EC2 instances register to the cluster. This will then
be followed by the tasks getting scheduled on to those instances, with the tasks being evenly distributed across the three AZs according
to our equal weights assigned to the capacity providers.

Once done, the console should now look something like below:

All tasks should be in a `RUNNING` state.
![running](/images/ec2-ecs-az-12.png)

### Scale the service back down to three

Now that we've seen cluster auto scaling scale out the backend EC2 infrastructure, let's drop the count back to three and watch as it will
scale our infrastructure back in.

We'll do it live! Go back into the deployment configuration (`~/environment/ecsdemo-capacityproviders/ec2-az/app.py`), and do the following:

Change the desired_count parameter from `12` to `3`.

```python
        self.load_balanced_service = aws_ecs_patterns.ApplicationLoadBalancedEc2Service(
            self, "EC2CapacityProviderService",
            service_name='ecsdemo-capacityproviders-ec2',
            cluster=self.base_platform.ecs_cluster,
            cpu=256,
            memory_limit_mib=512,
            desired_count=3,
            #desired_count=12,
            public_load_balancer=True,
            task_image_options=self.task_image,
        )
```

Now, save the changes, and let's deploy.

```bash
cdk deploy --require-approval never
```

That's it. Now, over the course of the next few minutes, the cluster autoscaler will recognize that we are well above capacity requirements, and
will scale the EC2 instances back in.


#### Review

What we accomplished in this section of the workshop is the following:

- Created three capacity providers for EC2 backed tasks that has managed cluster auto scaling enabled.
- We used weights to employ a strategy of balancing tasks across the three AZs regardless of the state of the infrastructure
- We deployed a service with three tasks and plenty of backend capacity and observed that the tasks were deployed according to our strategy
- We then scaled out to twelve tasks. This caused the managed cluster auto scaling to trigger a scale out event to have the backend infrastructure
meet the availability requirements of the tasks.
- We then scaled the service back to three, and watched as the cluster autoscaler scaled the EC2 instances back in to ensure that we weren't over provisioned.

#### Cleanup

Run the cdk command to delete the service (and dependent components) that we deployed.

```bash
cdk destroy -f
```

Next, go back to the ECS Cluster in the console. In the top right, select `Update Cluster`.

![updatecluster](/images/cp_update_cluster.png)

Under `Default capacity provider strategy`, click the `x` next to all of the strategies until there are no more left to remove. Once you've done that, click `Update`.

![deletecapprovider](/images/cp_delete_default.png)


That's it! Great job! That's it for the workshop today. I hope you had fun and learned a few things about ECS Cluster Auto Scaling. 