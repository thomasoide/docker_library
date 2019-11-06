# Deploying Library with Docker, AWS ECS Fargate and AWS ECR

This README documents how to deploy Library, the NYT's open-source documentation site, using Docker. This documentation is also targeted for people with MacOS/Linux machines. I have not deployed, nor have any clue, how deploy Library to Docker with Windows.

This documentation assumes that you've already completed the previous steps to access Google Cloud Platform, Google Drive API, OAuth, etc. If you haven't set that up yet, see the [NYT's documentation](https://nyt-library-demo.herokuapp.com/get-started) and follow it step by step.

Once you have your version of Library working locally, you're ready to put it on the internet.

## 1. Fork

The first thing you'll need is your own fork of the [Library repository](https://github.com/nytimes/library).

You'll be using the included Dockerfile (with a few modifications) to build the Docker image that you'll eventually push to AWS Elastic Container Registry. If you already have a fork of the repository, fetch from upstream to get the most up-to-date version of Library.

## 2. Customize

Library isn't ready straight out of the box. The first thing you'll want to do is to populate `./custom/` directory with your desired customizations to your library instance. The Times wrote documentation on the [custom directory here](https://github.com/nytimes/library/tree/master/custom).

The next thing you'll want to do is edit the Dockerfile:

```Dockerfile
FROM node:10

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install NPMs
COPY package.json* package-lock.json* /usr/src/app/
RUN npm i --production

COPY . /usr/src/app
RUN npm run build

CMD ["npm", "run", "start"]
```

The final thing you'll need to do (if you haven't done so already), is to populate a .env file with environment variables. You can find an example .env file in the README of Library, linked at the top of this document.

## 3. AWS ECR

You obviously won't be able to build a Docker image without Docker. You can download Docker [here](https://hub.docker.com/?overlay=onboarding).

This will give you a Docker Desktop app as well as access to Docker's CLI (which is what we'll use to deploy Library).

You'll also need an AWS account, your `.aws` directory configured with an access key and a secret access key and the AWS CLI installed globally.

+ [Configuring ~/.aws](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).
+ [Installing the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-macos.html).

Once you've done all that, you're ready to push a docker image to the ECR.

### ECR console to-do

1. Navigate to the ECR page on the AWS console `Home > Compute > ECR`.
2. Click `Create repository` in the top right corner of the screen.
3. Give your repository a name. It doesn't matter what it is.
4. Navigate to the repository page, and click `View Push Commands` in the top right corner.

### ECR Command Line to-do

1. Navigate to the directory where your Library fork lives.
2. Authenticate your Docker client to ECR using the command given by AWS. You will need to retype this command every 12 hours since that's the amount of time it takes for the authorization token to expire. Another important note about this command, if you have multiple AWS profiles in your `~/.aws/credentials` file, you'll need to use the `--profile` flag to a specify a specific profile. Note that AWS ECR will not let you push to a repository if you're using the wrong account.
3. Execute each of the next three push commands given to you by AWS. The first command, `docker build -t your_local_repo_name .`, builds a local Docker image on your computer. The second coomand, `docker tag your_local_repo_name:latest your_aws_id.dkr.ecr.your_region.amazonaws.com/your_remote_repo_name:latest`, tags the local image and points it at the repository URL. The third command, `docker push your_aws_id.dkr.ecr.your_region.amazonaws.com/your_remote_repo_name:latest`, pushes the image to AWS ECR.

## 4. AWS ECS

Amazon Elastic Container Service is how AWS hosts and runs Docker container on clusters. You can host services and tasks with AWS Fargate (the serverless architecture) or on a cluster of EC2 instances. We'll walk through using Fargate here.

1. From the AWS ECS Console, click on the Task Definitions tab, and create a new Task Definition (`Amazon ECS > Task Definitions > Create new Task Definition`). Select the FARGATE launch type.
2. Fill out the required fields. Make sure you give the task the `ecsTaskExecutionRole`.
3. Add a container to the task. You'll need the *image* URI not the repository URI (the app will not work if you use the repository URI). The memory hard limit should be the same as the Task memory you set for the task as a whole.
4. While in the container menu, open the port that your app listens on locally.
5. Finally, if your app has any environment variables that are necessary for it to function, put those in the `Environment variables` section. These are injected into the application at buildtime. If you want, you can also import the values from AWS Systems Manager Parameter store keys.
6. Create a new cluster from the AWS ECS homepage.

## 5. Adding a URL

You'll likely want to put your Library instance behind a URL instead of using a public IP address. But it's not as simple as simply buying a domain, creating a hosted zone and pointing a subdomain at the task's public IP.

Everytime you update a service/task in AWS Fargate, the public IP of the service changes. This means that every time you push up new updates to Library, you'll also have to change the IP where the subdomain points.

One way to solve this problem is by using Route 53 in tandem with an Application Load Balancer (I'm sure there are other ways to solve this issue).

### Creating the ALB

1. Open the EC2 Dashboard and select Load Balancers from the left-hand menu. Select "Application Load Balancer" and add a VPC, subnets and security group. Be sure to open port 443 if you want to use HTTPS instead of HTTP.
2. Do not add any Target Groups. This will come later.

### Routing

These next steps seem like a lot, but in reality it's not that bad. Shouldn't take more than a few minutes.

1. Navigate to Route 53 and click Hosted Zone that you want Library to point at. Create a new subdomain on that hosted zone and give it a name.
2. Make sure that Alias is set to `yes` and alias it to the load balancer you just created.
3. Now you'll create a target group. Navigate to the Target Group console, which is located under the EC2 service.
4. Give the target group a name and the correct protocol (HTTP/HTTPS). Make sure the Target type is `IP`.
5. In "Advanced health check settings", be sure to add 302 to the list of success codes. If you don't do this, Library will constantly destroy and rebuild itself within your ECS cluster.
6. Navigate back to your load balancer and under the `Listeners` tab, click on the link that says "View/edit rules." Add a rule. It should read `I Host is __your_subdomain_here__.com THEN Forward to __your_target_group_here__`. This will help create the connection between the ECS service and the load balancer.

## 5. To the Interwebs!

1. Create a new cluster for your app to live on (if you haven't created one already).
3. Open the cluster and create a new service. Select the task definition that you previously created.
2. Set `Number of tasks` to 1, since Library is being run off a single Docker container.
3. Set VPC, subnets and security groups. If you want the app to be public, you can let AWS create these elements for you. If you have internal security group rules that IP restrict your app, you can add those here. Make sure that the "Auto-assign public IP" option is `ENABLED`.  
5. Once you set the VPC and subnets, you'll be able to access load balancing options. Choose the `Application Load Balancer` option, and choose the ALB you created from the dropdown menu. Then select your container and click the blue button that says `Add to load balancer`. Uncheck the box labeled `Enable service discovery integration`.
5. Select the stock AWS options and let AWS provision the service. Once your service task says "RUNNING", you should be able to go to the DNS you created and see your Library instance.

Congratulations, you just deployed Library with Docker! 
