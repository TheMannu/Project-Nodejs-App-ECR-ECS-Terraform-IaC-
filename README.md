# Infrastructure as Code with Terraform on AWS: ECR and ECS Implementation

### Introduction
This guide walks you through implementing infrastructure for Amazon Elastic Container Registry (ECR) and Amazon Elastic Container Service (ECS) using Terraform on AWS.

### Prerequisites
1. An AWS account with appropriate permissions to create IAM users, S3 buckets, DynamoDB tables, ECR repositories, ECS clusters, etc.
2. Terraform installed on your local machine.
3. Basic understanding of AWS services, Terraform, and Docker


### Step 1: Setting Up Terraform Configuration
1. Create a main.tf file with Terraform block and AWS provider configuration.
2. Create a providers.tf file and configure the AWS provider block.
3. Create a module for Terraform remote state using S3 and DynamoDB for state locking.
   - Add resource blocks for S3 bucket and DynamoDB table in modules/TFState/tfstate.tf.
   - Define variables for bucket name and table name in variables.tf.
   - Create a module block in main.tf for TF state pointing to the TF State folder.
   - Define local variables for bucket name and table name in locals.tf.

### Step 2: Initialize Terraform and Provision Resources
1. Run `terraform init` to initialize Terraform and download providers.
2. Run `terraform validate` to check the configuration.
3. Run `terraform plan` to review the planned infrastructure changes.
4. Run `terraform apply` to apply the configuration and provision S3 bucket and DynamoDB table.

### Step 3: Setting Up ECR Repository
1. Create an `ECR` folder inside the `modules` directory.
2. Create an `ECR.tf` file in the `modules/ECR` folder for defining the ECR repository resource.
3. Define variables and outputs for the ECR repository.
4. Add an output block for the repository URL in outputs.tf.
5. Update main.tf to include the ECR module and pass variables.
6. Define local variables for the ECR repo name in locals.tf.
7. Initialize Terraform again to pick up the new module.
8. Run `terraform validate` and `terraform plan` to verify and plan changes.
9. Run `terraform apply` to create the ECR repository.

### Step 4: Dockerize Application and Push to ECR
1. Dockerize your application by creating a Dockerfile.
2. Build the Docker image and push it to your ECR repository using AWS CLI commands.

### Step 5: Setting Up ECS Cluster
1. Create an `ECS` folder inside the `modules` directory.
2. Create an `ecs.tf` resource block in the `modules/ECS` folder to define ECS cluster,subnets,    task definition, IAM roles, load balancer, etc.
3. Define variables for ECS module in variables.tf.
5. Update main.tf to include the ECS module and pass variables.
6. Define local variables for ECS module in locals.tf.
4. Initialize Terraform to pick up the new module.
5. Run `terraform validate` and `terraform plan` to verify and plan changes.
6. Run `terraform apply` to create the ECS cluster and associated resources.

### Step 7: Testing the Infrastructure
1. Verify the ECS cluster, task, and ALB configurations in the AWS Management Console.
2. Access the application via the load balancer DNS name in ec2 dashboard-load balencer to test it's functionality.



## Hello World

![Screenshot (304) (Small)](https://github.com/TheMannu/nodejsApp/assets/84488161/c3370898-e2cf-421d-a88d-83035f4b3b90)

### Step 8: Cleanup
1. Run `terraform destroy` to clean up and delete all created resources.
   
# [linkedIn] (www.linkedin.com/in/ashwank)



