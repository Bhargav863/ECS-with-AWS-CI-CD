# CI/CD on AWS Using GitHub, Amazon ECR, ECS Fargate, CodeBuild, and CodePipeline

This guide walks through deploying a static website to Amazon ECS on AWS
Fargate and automating future deployments with GitHub, AWS CodeBuild,
and AWS CodePipeline.

## 1. Launch and prepare an Amazon Linux 2023 EC2 instance

1.  Launch an **Amazon Linux 2023** EC2 instance (for this lab,
    `t3.micro`).
2.  Configure the security group to allow:
    -   SSH (`22`) from your IP address.
    -   HTTP (`80`) from the internet, if you want to test the website
        directly from this EC2 instance.
3.  Connect to the instance and install Docker:

``` bash
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

Check Docker:

``` bash
docker images
docker ps -a
```

> If your EC2 user is not permitted to run Docker commands, use `sudo`
> with the Docker commands, or add the user to the Docker group and
> reconnect.

## 2. Install Git and clone the website repository

Install Git:

``` bash
sudo yum install git -y
```

Clone the project and move into its directory:

``` bash
git clone https://github.com/ReyazShaik/website.git
cd website
```

## 3. Build and run the Docker image

The repository already contains a `Dockerfile`. Inspect it:

``` bash
cat Dockerfile
```

The Dockerfile:

``` dockerfile
FROM nginx:latest

COPY . /usr/share/nginx/html/

EXPOSE 80
```

**What these instructions do:**

-   `FROM nginx:latest` uses the official NGINX image as the base image.
-   `COPY . /usr/share/nginx/html/` copies the files from the build
    context (the current project directory) into NGINX's default web
    root.
-   `EXPOSE 80` documents that the containerized web server listens on
    port 80. It does not publish the port on the EC2 host by itself.

Check existing images:

``` bash
docker images
```

Build the image and tag it as `website:latest`:

``` bash
docker build -t website:latest .
```

Verify that the image was created:

``` bash
docker images
```

Run a container (optional local test):

``` bash
docker run -d --name website-test -p 80:80 website:latest
```

Check the container:

``` bash
docker ps -a
```

Open the EC2 instance's public IPv4 address in a browser to test the
website. Ensure inbound TCP port `80` is allowed by the EC2 security
group.

> **Note:** Run the container only if you want to test the image
> directly on EC2. If port 80 is already occupied, stop and remove the
> test container before starting another service on that port:
>
> ``` bash
> docker stop website-test
> docker rm website-test
> ```

## 4. Push the Docker image to Amazon ECR

### 4.1 Create an ECR repository

1.  Open **Amazon Elastic Container Registry (ECR)** in the AWS Console.
2.  Create a **private repository** named `website`.
3.  Copy the repository URI. The example used in this lab is:

``` text
289857919920.dkr.ecr.ap-south-1.amazonaws.com/website
```

Replace the account ID, region, and repository name if yours are
different.

### 4.2 Attach an IAM role to the EC2 instance

Create an IAM role for EC2 and attach a suitable ECR permissions policy.
For this image-push lab, the instance needs permissions to authenticate
to ECR and push images. Prefer a least-privilege policy scoped to the
target repository rather than granting broad administrator access.

Attach the role to the EC2 instance through **EC2 → Instances → Actions
→ Security → Modify IAM role**.

### 4.3 Authenticate Docker to ECR

For the example region `ap-south-1`:

``` bash
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin \
  289857919920.dkr.ecr.ap-south-1.amazonaws.com
```

If the AWS CLI is not installed, install it and configure the instance
role before running this command.

### 4.4 Build, tag, and push the image

From the directory containing the Dockerfile:

``` bash
docker build -t website:latest .

docker tag website:latest \
  289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest

docker push \
  289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest
```

Open the `website` repository in ECR and verify that the image appears.

## 5. Create an Amazon ECS cluster using Fargate

1.  Open **Amazon Elastic Container Service (ECS)**.
2.  Create a cluster named `websitecluster`.
3.  Select **AWS Fargate** as the infrastructure option. Fargate is
    serverless compute for containers; AWS manages the underlying
    infrastructure.
4.  Container Insights / monitoring is optional for this lab.

### Fargate vs. ECS on EC2

-   **Fargate:** AWS manages the underlying compute infrastructure. You
    specify task CPU and memory.
-   **EC2:** You manage the EC2 container instances that provide compute
    capacity for ECS tasks.

## 6. Create an ECS task definition

Create a task definition before running tasks or creating a service.

Use the following example settings:

  ----------------------------------------------------------------------------------------------------
  Setting                             Value
  ----------------------------------- ----------------------------------------------------------------
  Task definition family              `website-task-df`

  Launch / infrastructure type        AWS Fargate

  Operating system                    Linux

  CPU                                 `0.25 vCPU`

  Memory                              `0.5 GB`

  Container name                      `website`

  Container image                     `289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest`

  Container port                      `80`

  Port protocol                       TCP
  ----------------------------------------------------------------------------------------------------

-   Configure a task role only if the application running inside the
    container needs permission to access other AWS services.
-   Configure the task execution role so ECS can pull the image from ECR
    and publish logs to CloudWatch Logs. The standard
    `ecsTaskExecutionRole` is commonly used.
-   Enable CloudWatch logging for the container.
-   Leave other settings at their defaults for this basic lab, then
    create the task definition.

## 7. Optional: Run standalone ECS tasks

> **Optional lab exercise --- do not do this if you only want to deploy
> through the ECS service.**

1.  Open `websitecluster` → **Tasks** → **Run new task**.
2.  Select **Fargate**.
3.  Select the task definition family `website-task-df` and its
    revision.
4.  Set the desired number of tasks (for example, `2`).
5.  Configure the VPC, subnets, and security group.
6.  Enable a public IP only if the tasks need direct internet access for
    this test.
7.  Run the task.

Open a task's public IP in a browser to test it. The task security group
must allow inbound TCP port `80`.

> A task's public IP is not a stable customer-facing endpoint. For a
> production-style deployment, use an ECS service with a load balancer.

## 8. Create an Application Load Balancer and target group

You can create these resources before creating the ECS service, or use
the ECS service creation workflow to create/configure them if that
option is available in your console.

For an Application Load Balancer (ALB):

1.  Create an **IP-type** target group because Fargate tasks use
    `awsvpc` networking.
2.  Set the target group's protocol and port to HTTP `80`.
3.  Configure the health check path as `/`.
4.  Create an internet-facing ALB in the required VPC and public
    subnets.
5.  Add an HTTP listener on port `80` and forward traffic to the target
    group.
6.  Ensure the ALB security group allows inbound HTTP `80` from the
    intended clients.
7.  Ensure the ECS task security group allows inbound TCP port `80` from
    the ALB security group.

If you use HTTPS, configure an HTTPS listener and certificate in
addition to the HTTP configuration.

## 9. Create an ECS service

Create a service in `websitecluster` using the task definition created
earlier.

Example settings:

  Setting                   Value
  ------------------------- ---------------------------------------------------
  Launch / infrastructure   Fargate
  Application type          Service
  Task definition family    `website-task-df`
  Service name              `websiteservice`
  Desired tasks             `4` (example)
  Public IP                 Enable only if required by your networking design
  Load balancer             Application Load Balancer
  Target group              The IP-type target group created for this service

Networking:

-   Select the correct VPC and subnets.
-   Choose the ECS task security group.
-   For a public-subnet lab setup, enable public IP assignment if
    required.
-   For a production design, private subnets with suitable outbound
    connectivity are generally preferred.

### Optional ECS Service Auto Scaling

Configure service auto scaling if required:

  Setting          Example
  ---------------- -------------------------------------
  Minimum tasks    `2`
  Maximum tasks    `6`
  Scaling policy   Target tracking
  Metric           ECS service average CPU utilization
  Target value     `70%`

Create the service and wait until the tasks are running and the target
group reports healthy targets. Access the website using the ALB DNS
name, not an individual task IP.

## 10. Configure AWS CodeBuild

CodeBuild will build the Docker image and push it to ECR when source
code changes.

1.  Open **AWS CodeBuild** and choose **Create build project**.
2.  Configure the project:

  -----------------------------------------------------------------------------
  Setting                             Value
  ----------------------------------- -----------------------------------------
  Project name                        `website`

  Project type                        Default

  Source provider                     GitHub

  Repository                          `https://github.com/ReyazShaik/website`

  Buildspec                           Use the `buildspec.yml` file in the
                                      repository

  Artifacts                           No artifacts, if the pipeline only needs
                                      the image definitions file from the build
                                      output configuration described below
  -----------------------------------------------------------------------------

3.  Create or select a CodeBuild service role.
4.  Ensure the CodeBuild role has the permissions required to:
    -   Authenticate to ECR and push images to the `website` repository.
    -   Write build logs to CloudWatch Logs.
    -   Read Secrets Manager secrets only if the build actually uses
        them.
5.  Start a build and inspect the phase details.

If the first build fails during ECR login or image push, check the
CodeBuild service role's ECR permissions, repository policy, region, and
account/repository URI. After correcting the permissions, run the build
again and verify the new image in ECR.

> Avoid attaching AdministratorAccess or broad Secrets Manager write
> permissions just to make the build pass. Grant only the permissions
> the build needs.

### Important: `buildspec.yml` and image definitions

The repository's `buildspec.yml` must contain the correct AWS account
ID, region, ECR repository URI, image tag, and commands to build and
push the image.

For ECS CodePipeline deployments, the build must also produce an
`imagedefinitions.json` file in the format expected by the ECS deploy
action. The container name in that file must exactly match the container
name in the ECS task definition (`website` in this guide).

Example `imagedefinitions.json`:

``` json
[
  {
    "name": "website",
    "imageUri": "289857919920.dkr.ecr.ap-south-1.amazonaws.com/website:latest"
  }
]
```

Make sure CodeBuild is configured to expose this file as a build
artifact for the pipeline's ECS deploy action. If your buildspec
currently says **no artifacts**, update it so the pipeline can receive
`imagedefinitions.json`.

## 11. Create an AWS CodePipeline for ECS continuous deployment

Create a pipeline to connect GitHub, CodeBuild, and ECS.

### 11.1 Create the pipeline

1.  Open **AWS CodePipeline** and choose **Create pipeline**.
2.  Enter the pipeline name `ECS-CICD`.
3.  Select the pipeline execution mode that suits the lab. The queued
    mode can serialize executions.
4.  Create or select the pipeline service role.

### 11.2 Configure the source stage

1.  Select **GitHub (via GitHub App / connection, V2)** as the source
    provider.
2.  Create a connection, for example `aws-codepipeline-connection`.
3.  Choose **Connect to GitHub** and install/authorize the AWS Connector
    for GitHub.
4.  Select the repository `website`.
5.  Select the repository's actual default branch. The original lab
    notes use `master`; use `main` instead if that is your repository's
    default branch.
6.  Configure the source trigger for pushes to the selected branch. For
    example, choose the push event and add a branch filter for `master`
    (or `main`, as appropriate).

### 11.3 Configure the build stage

1.  Select **AWS CodeBuild** as the build provider.
2.  Choose the CodeBuild project `website`.
3.  Use a single build action.
4.  Skip a separate test stage if this lab does not include one. In a
    real project, add automated tests and quality/security checks where
    appropriate.

### 11.4 Configure the deploy stage

1.  Select **Amazon ECS** as the deploy provider.
2.  Select the AWS Region.
3.  Select cluster `websitecluster`.
4.  Select service `websiteservice`.
5.  Set the image definitions file to:

``` text
imagedefinitions.json
```

6.  Create the pipeline and allow its first execution to complete.

## 12. Verify the CI/CD workflow

1.  Check the latest CodePipeline execution and confirm that the source,
    build, and deploy stages succeed.
2.  Check the latest CodeBuild run and review its logs.
3.  Open the ECR `website` repository and verify the newly pushed image.
4.  Check the ECS service and confirm that the new tasks are running and
    healthy in the ALB target group.
5.  Open the ALB DNS name in a browser and verify the website.

To test continuous deployment, make a small change to `index.html` in
GitHub and push it to the configured branch. CodePipeline should detect
the change, invoke CodeBuild to build and push the image, and then
update the ECS service using the new image definition.

> **Image-tag recommendation:** For a more reliable deployment history,
> use a unique image tag (such as the Git commit ID) rather than relying
> only on `latest`. Ensure the generated `imagedefinitions.json`
> references that exact tag.

## 13. Clean up AWS resources

When you finish the lab, delete resources in an order that avoids
dependencies:

1.  Delete the CodePipeline pipeline.
2.  Delete the ECS service (or set desired tasks to `0` first, then
    delete it).
3.  Stop any standalone ECS tasks.
4.  Deregister task definition revisions if you no longer need them.
5.  Delete the ECS cluster after its services and tasks are removed.
6.  Delete the ALB.
7.  Delete the target group after it is no longer in use by a listener
    or service.
8.  Delete the ECR image(s), then delete the ECR repository if no longer
    needed.
9.  Delete the CodeBuild project if it is no longer needed.
10. Remove the EC2 instance, IAM roles/policies created only for this
    lab, and any other lab-only resources.

Check for related resources such as CloudWatch log groups, security
groups, Elastic IPs, NAT Gateways, and public IPv4 addresses. Some
resources can continue to incur charges even after the main application
is deleted.

------------------------------------------------------------------------

## Lab notes and important corrections

-   The container image repository is created in **Amazon ECR**, not
    ECS.
-   The task definition family used by the service should match the
    family created earlier (`website-task-df` in this guide). The
    original notes referenced `nginx-def` at service creation; use the
    correct family name or your own actual family.
-   Fargate target groups must use **IP** target type, not instance
    target type.
-   The ALB should route to the ECS service's target group. Customers
    should use the ALB DNS name (or a custom domain), not a task's
    public IP.
-   A CodePipeline ECS deploy action needs the `imagedefinitions.json`
    file as an input artifact. A build configured with no artifacts will
    not provide that file to the deploy action.
-   Use least-privilege IAM permissions. The EC2 instance role and
    CodeBuild role have different purposes: the EC2 role pushes an image
    during the manual lab step, while the CodeBuild role pushes images
    during automated builds.
-   The commands and account/repository URI in this document are
    examples from the lab. Replace them with your own values before
    running them.
