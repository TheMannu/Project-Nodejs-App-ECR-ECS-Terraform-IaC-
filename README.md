# Infrastructure as Code with Terraform on AWS: ECR and ECS Implementation

### Introduction
This guide walks you through implementing infrastructure for Amazon Elastic Container Registry (ECR) and Amazon Elastic Container Service (ECS) using Terraform on AWS.

### Prerequisites
1. An AWS account with appropriate permissions to create IAM users, S3 buckets, DynamoDB tables, ECR repositories, ECS clusters, etc.
2. Terraform installed on your local machine.

3. Basic understanding of AWS services, Terraform, and Docker

### Setting Up Terraform Configuration

1. Create a main.tf file with Terraform block and AWS provider configuration.
- Create a **main.tf** file, and paste this block
```
terraform {
  required_version = "~>1.3"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~>4.8"
    }
  }
}
```

2. Create a **providers.tf** file and configure the AWS provider block.

```
provider "aws" {
  region = "us-east-1"
}
```

3. Create a module folder for Terraform remote state using S3 and DynamoDB for state locking.


- Inside this **modules** folder, Crate a another folder **tf-state** folder.
   - Inside this **tf-state** folder, Create a File **tf-state.tf**
   - Add resource blocks for S3 bucket and DynamoDB table in modules/TFState/tf-state.tf.
```
resource "aws_s3_bucket" "terraform_state" {
  bucket        = var.bucket_name
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "terraform_bucket_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state_crypto_conf" {
  bucket = aws_s3_bucket.terraform_state.bucket
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = var.table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

   - Inside this **tf-state** folder, Create another File **variables.tf**
   - Define variables for bucket name and table name in variables.tf.

```
variable "bucket_name" {
  description = "Remote S3 Bucket Name"
  type        = string
  validation {
    condition     = can(regex("^([a-z0-9]{1}[a-z0-9-]{1,61}[a-z0-9]{1})$", var.bucket_name))
    error_message = "Bucket Name must not be empty and must follow S3 naming rules."
  }
}

variable "table_name" {
  description = "Remote DynamoDB Table Name"
  type        = string
}
```
- Now move to **main.tf** file 
- Create a module block in main.tf for tf-state pointing to the tf-state folder.Add this block of code 

~~~
module "tf-state" {
  source      = "./modules/tf-state"
  bucket_name = local.bucket_name
  table_name  = local.table_name
}
~~~

- Now Create another file adjacent to main.tf file named **locals.tf** 
- Define local variables for bucket name and table name in locals.tf file. And paste this block
~~~
locals {
  bucket_name = "iamtwo-s3-bucket"
  table_name  = "iamtwo-db-table"
} 
~~~


###  Initialize Terraform and Provision Resources
1. Run `terraform init` to initialize Terraform and download providers.

2. Run `terraform validate` to check the configuration.
3. Run `terraform plan` to review the planned infrastructure changes.
4. Run `terraform apply` to apply the configuration and provision S3 bucket and DynamoDB table.With server side encryption and bucket versioning

### Varify for S3 and Dynamodb table.
```
aws s3 ls
aws dynamodb list-tables
```

### Now move to the **main.tf** file and add backend block for s3 bucket
```
backend "s3" {
   bucket         = "iamtwo-s3-bucket"
   key            = "tf-infra/terraform.tfstate"
   region         = "us-east-1"
   dynamodb_table = "iamtwo-db-table"
   encrypt        = true
}
```
Again run **terraform init** comands to configure the state files to the s3 and enter the key value **yes**.

### Step 3: Setting Up ECR Repository

- Create an `ecr` folder inside the `modules` directory.

   - Create an `ecr.tf` file in the `modules/ecr` folder for defining the ECR repository resource.
```
resource "aws_ecr_repository" "demo_app_ecr_repo" {
  name = var.ecr_repo_name
}
```
   - Create an `variables.tf` file in the `modules/ecr` folder for defining the variable definitation for the ECR repository.
```
variable "ecr_repo_name" {
  description = "ECR Repo Name"
  type        = string
}
```

   - Create an `outputs.tf` file in the `modules/ecr` folder for output block for defining the output url of  ECR repository.*will be sent to ecs module as a variable*
```
output "repository_url" {
  value = aws_ecr_repository.demo_app_ecr_repo.repository_url
}
```
### Now move to the **main.tf** file and add ECR module and pass variables.

```
module "ecrRepo" {
  source = "./modules/ecr"

  ecr_repo_name = local.ecr_repo_name
}
```
### Now move to the **locals.tf** file and add local variables for the ecr-repo-name.

```
 ecr_repo_name = "demo-app-ecr-repo"
```

- Initialize Terraform again to pick up the new module.
- Run `terraform validate` and `terraform plan` to verify and plan changes.
- Run `terraform apply` to create the ECR repository.

*we can varify by looking **demo-app-ecr-repo** into Amazon ECR section of Amazon console but it will not contain any image*

### Setting up the application
   Copy the code from repository from **App > src *

#### Varify the Application
To check if application is working fine 
   - Move inside the **App/src** folder 
      - Run command `npm install` to install all the dependencies.

      - Run command `node server.js` to start the server.

      - Open new terminal and Run command `curl http://localhost:3000/`,It will the output of the application in JSON format.

### Dockerize Application and Push to ECR

- Dockerize your application by creating a Dockerfile 
   - App > Dockerfile  paste the  block
```
FROM node:alpine

WORKDIR /src

COPY ./src/package.json .
RUN npm install

COPY ./src/ .

EXPOSE 3000

CMD ["node", "server.js"]
```

2. Build the Docker image and push it to our ECR repository using AWS CLI commands.
- Locate the ECR repository in the amazon ecr console and click the **view push commands**
   - Follow the instructions and Run the commands in terminal at **/App** to :-
      - Login the ECR repository **/** location
      - Build the Docker Image at **/App** location
      - Tag the Docker Image at **/App** location
      - and Push the image to the repository at **/App** location
   - Varify the repository in our ECR at amazon ecr console.

### Setting Up ECS Cluster
- Create an `ecs` folder inside the `modules` directory add this block.
   - Create an `ecs.tf` resource block in the `modules/ecs` folder to define ECS cluster,subnets,task definition, IAM roles, load balancer, etc.
```

resource "aws_default_subnet" "default_subnet_c" {
  availability_zone = var.availability_zones[2]
}

resource "aws_ecs_task_definition" "demo_app_task" {
  family                   = var.demo_app_task_famliy
  container_definitions    = <<DEFINITION
  [
    {
      "name": "${var.demo_app_task_name}",
      "image": "${var.ecr_repo_url}",
      "essential": true,
      "portMappings": [
        {
          "containerPort": ${var.container_port},
          "hostPort": ${var.container_port}
        }
      ],
      "memory": 512,
      "cpu": 256
    }
  ]
  DEFINITION
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  memory                   = 512
  cpu                      = 256
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
}

resource "aws_iam_role" "ecs_task_execution_role" {
  name               = var.ecs_task_execution_role_name
  assume_role_policy = data.aws_iam_policy_document.assume_role_policy.json
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_alb" "application_load_balancer" {
  name               = var.application_load_balancer_name
  load_balancer_type = "application"
  subnets = [
    "${aws_default_subnet.default_subnet_a.id}",
    "${aws_default_subnet.default_subnet_b.id}",
    "${aws_default_subnet.default_subnet_c.id}"
  ]
  security_groups = ["${aws_security_group.load_balancer_security_group.id}"]
}

resource "aws_security_group" "load_balancer_security_group" {
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_lb_target_group" "target_group" {
  name        = var.target_group_name
  port        = var.container_port
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = aws_default_vpc.default_vpc.id
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_alb.application_load_balancer.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.target_group.arn
  }
}

resource "aws_ecs_service" "demo_app_service" {
  name            = var.demo_app_service_name
  cluster         = aws_ecs_cluster.demo_app_cluster.id
  task_definition = aws_ecs_task_definition.demo_app_task.arn
  launch_type     = "FARGATE"
  desired_count   = 1

  load_balancer {
    target_group_arn = aws_lb_target_group.target_group.arn
    container_name   = aws_ecs_task_definition.demo_app_task.family
    container_port   = var.container_port
  }

  network_configuration {
    subnets          = ["${aws_default_subnet.default_subnet_a.id}", "${aws_default_subnet.default_subnet_b.id}", "${aws_default_subnet.default_subnet_c.id}"]
    assign_public_ip = true
    security_groups  = ["${aws_security_group.service_security_group.id}"]
  }
}

resource "aws_security_group" "service_security_group" {
  ingress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = ["${aws_security_group.load_balancer_security_group.id}"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
   - Create a `data.tf` file in the `modules/ecs` folder and add a data block for aws-iam-policy 

```
data "aws_iam_policy_document" "assume_role_policy" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}
```
   - Create a `variables.tf` file in the `modules/ecs` folder and Define variables for ECS module in variables.tf.
```

variable "demo_app_cluster_name" {
  description = "ECS Cluster Name"
  type        = string
}

variable "availability_zones" {
  description = "us-east-1 AZs"
  type        = list(string)
}

variable "demo_app_task_famliy" {
  description = "ECS Task Family"
  type        = string
}

variable "ecr_repo_url" {
  description = "ECR Repo URL"
  type        = string
}

variable "container_port" {
  description = "Container Port"
  type        = number
}

variable "demo_app_task_name" {
  description = "ECS Task Name"
  type        = string
}

variable "ecs_task_execution_role_name" {
  description = "ECS Task Execution Role Name"
  type        = string
}

variable "application_load_balancer_name" {
  description = "ALB Name"
  type        = string
}

variable "target_group_name" {
  description = "ALB Target Group Name"
  type        = string
}

variable "demo_app_service_name" {
  description = "ECS Service Name"
  type        = string
}

```
- Update `main.tf` to include the `ecs` module and pass variables.
```
module "ecsCluster" {
  source = "./modules/ecs"

  demo_app_cluster_name = local.demo_app_cluster_name
  availability_zones    = local.availability_zones

  demo_app_task_famliy         = local.demo_app_task_famliy
  ecr_repo_url                 = module.ecrRepo.repository_url
  container_port               = local.container_port
  demo_app_task_name           = local.demo_app_task_name
  ecs_task_execution_role_name = local.ecs_task_execution_role_name

  application_load_balancer_name = local.application_load_balancer_name
  target_group_name              = local.target_group_name
  demo_app_service_name          = local.demo_app_service_name
}
```
- Update local variables for `ecs` module in `locals.tf`.

```
  demo_app_cluster_name        = "demo-app-cluster"
  availability_zones           = ["us-east-1a", "us-east-1b", "us-east-1c"]
  demo_app_task_famliy         = "demo-app-task"
  container_port               = 3000
  demo_app_task_name           = "demo-app-task"
  ecs_task_execution_role_name = "demo-app-task-execution-role"

  application_load_balancer_name = "cc-demo-app-alb"
  target_group_name              = "cc-demo-alb-tg"

  demo_app_service_name = "cc-demo-app-service"
  ```

- Initialize Terraform to pick up the new module.
- Run `terraform validate` and `terraform plan` to verify and plan changes.
- Run `terraform apply` to create the ECS cluster and associated resources.

### Step 7: Testing the Infrastructure
- Verify the ECS cluster, task, and ALB configurations in the AWS Management Console.
- Access the application via the load balancer DNS name in ec2 dashboard-load balencer to test it's functionality.

### Step 8: Cleanup

1. Run `terraform destroy` to clean up and delete all created resources.

## [linkedIn] (www.linkedin.com/in/ashwank)



