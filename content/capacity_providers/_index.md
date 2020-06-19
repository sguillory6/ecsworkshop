+++
title = "Capacity Providers"
chapter = true
weight = 54
+++

# Amazon ECS Cluster Capacity Providers

In this chapter, we will gain an understanding of how Capacity Providers can help us in managing autoscaling for EC2 backed tasks, as well as ways to implement cost savings when running Fargate tasks.
We will implement two capacity provider strategies in our cluster: 

- For a Fargate backed ECS service, we will implement a strategy to deploy that service as a mix between Fargate and Fargate Spot.

- For an EC2 backed ECS service, we will implement Cluster Auto Scaling by increasing the task count of a service beyond the capacity available. This will require the backend EC2 infrastucture to scale to meet the demand, which the ECS cluster autoscaler will handle.

- For an EC2 backed ECS service that employs a mix of on-demand and spot instances, we will implement a strategy to over provision our service by 33% using spot instances for the over provisioned portion to save on costs. This example will demonstrate the use of multiple Capacity Providers in a cluster and the concept of assigning weights to Capacity Providers.

- For an EC2 backed ECS service, we will implement a Cluster Auto Scaling strategy that balances our tasks across availability zones regardless of the state of the architecture, achieving an application-driven approach to development as
opposed to an infrastructure-driven approach.

Let's get started!

