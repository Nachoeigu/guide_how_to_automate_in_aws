# Step by step: How to upload your Python Docker Image in the Cloud and execute it in schedule with AWS

## First, we should create this inside the project:
- A **requirements.txt** file, with the needed libraries for the project.
For example:
```
requests
pandas
unidecode
python-dotenv
datetime
```

- A **Dockerfile**, with the instruction about how docker should work.
For example
```
#Python base image
FROM python:3.8

#We define the workdirectory in the docker image
WORKDIR /app 

#We copy the files inside our local directory into the virtual work directory created in the line above
COPY . /app

#We update pip for installations
RUN pip install --upgrade pip

#We execute this line to install the dependencies of the project in the docker image
RUN pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable: you can hardcode them or put its input names. If you let them empty, you can pass it in the execution of the image with the flag -e, or in AWS with the Task Definition with the key "secrets"
ENV chat_id -693888464 #Your code should refer this variable like 'os.getenv("chat_id")' exactly.
ENV telegram_bot '' #Your code should refer this variable like 'os.getenv("telegram_bot")' exactly.

# Run app.py when the container launches
CMD ["python", "flights/main.py"]
```

- A **.dockerignore** file, where we will exclude private files.
For example
```
*.csv
.env
/venv
...
...
...
```


## Building and testing Docker image:
You should build the docker image. First, we build the image without the flag platform so we can test it locally:
```
docker build -t <desired_docker_image_name>:latest .
```
Then you can run your image with:
```
docker run <your_docker_image>:latest
```
If everything runs correctly, it means that the docker image was set up propertly.
You are ready for building the docker image again, but now for production purposes.
We remove the local testing image (then I will explain why).
```
docker rmi <your_docker_image>:latest
```
```
docker ps -l #You see the containers
docker rm <the_container_id_where_the_image_ran> #We remove the used container
```
Finally we are ready to build the final Docker image with the following command:
```
docker build --platform: linux/amd64 -t <desired_docker_image_name>:latest .
```
This is the reason we built again: we put the platform linux/amd64 because we will run the image in AWS Fargate, where has the host linux/amd64, and if we want to run the Docker image in our local host we will receive an error because the platform doesnt match.

## Create an IAM user in AWS:
You should create an account in Amazon Web Service, create a user (IAM) and obtain your Access Key ID and the Secret Access Key.

While creating your IAM user, add the following permissions:

- **AmazonEC2ContainerRegistryFullAccess**: it allows you to manage and maintain your Docker container images and repositories.

- **AmazonECS_FullAccess**:  it allows you to create, manage, and deploy containers using ECS.

- **AWSLambda_FullAccess**: It allows you to manage Lambda services, like creating Lambda functions.

- **SecretsManagerReadWrite**: It allows you to create, modify or delete secret variables.

- **AmazonECSTaskExecutionRolePolicy**: It allows you to run tasks.

- **AmazonEC2ContainerServiceforEC2Role**: It allows you to run and manage services and clusters.

## Loging into AWS, via AWS CLI:
Then, download the AWS CLI from [here](https://www.docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) or with "pip install awscli".
Go to the terminal and execute:
```
aws configure
```
In the terminal output you should put your access key, secret key, the region you will use.

## Creating Docker repository in ECR:
After that, create a repository in Elastic Container Registry, where we store Docker images.
```
aws ecr create-repository --repository-name <your_desired_name_for_the_repository>
```

## Authentificate Docker with your ECR:
You will need the account_id of your AWS account and the region. This is needed in order to push images from local to cloud.
```
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<your-region>.amazonaws.com
```

## Push Docker Image inside the ECR instance:
Before pushing, we should tag our image with the path where it will be stored on ECR.
```
docker tag ≤your_desired_docker_image>:latest <account-id>.dkr.ecr.<your-region>.amazonaws.com/<your_docker_repo_name_in_aws>:<desired_tag_of_the_image_we_want_to_run>
```
The *desired_tag_of_the_image_we_want_to_run* usually is "latest".

Then, yes. It is time to push it and save it in ECR. You push the image recently tagged with the route to ECR:
```
docker push <account-id>.dkr.ecr.<your-region>.amazonaws.com/<your_docker_repo_name_in_aws>:<tag_of_the_image_we_want_to_run>
```

## Create secrets in AWS Secret Manager (in case you have to handle enviroment variables)
The AWS Secret Manager works like your .env file. You store your private keys or access tokens.
You can create one with the following command. In this example, in order to create a telegram_token.
```
aws secretsmanager create-secret --name "<your_desired_secret_name>" --secret-string "{\"telegram_token\":\"19082JAKa_/as\"}"
```
The flag name defines the secret name, and the secret string flag defines the key/value pair where you pass the key telegram_token and the value.

## Create a Log Group for mapping the logging of your script
You create a log group in order to handle the logging that your script will show in its executions. 
```
aws logs create-log-group --log-group-name <your_desired_log_group_name>
```
You can see the logs of your executed tasks in AWS CloudWatch.

## Create the Task Definition

This is a template task definition:
```
{
    "family": "<the_desired_name_of_the_task>",
    "executionRoleArn": "arn:aws:iam::<your_account_id>:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc", #We use this mode for AWS Fargate apps
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "1024",
    "memory": "3072",
    "containerDefinitions": [
        {
            "name": "<the_desired_name_you_want_for_the_container>",
            "image": "<your_account_id>.dkr.ecr.<your_region>.amazonaws.com/<the_ecr_repository_name>:<tag_of_the_image_we_want_to_run>",
            "cpu": 0,
            "portMappings": [],
            "essential": true,
            "environment": [],
            "mountPoints": [],
            "volumesFrom": [],
            #Here we pass the sensitive data
            "secrets": [
                {
                    "name": "<the_name_of_your_secret_variable>",
                    "valueFrom": "arn:aws:secretsmanager:<your_region>:<your_account_id>:secret:<the_key_of_the_json_inside_the_secret_variable>"
                }
            ],
            #Here we store the no sensible data
            "environment": [
            {
                "name": "<the_name_of_your_parametrizable_variable>", #It should match with the name in the python script in the os.getenv("...")
                "value": "123456789ABC!" 
            }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "<the_log_group_you_create_for_this_task>",
                    "awslogs-region": "<your_region>",
                    "awslogs-stream-prefix": "<additional_info_you_want_to_add_at_the_beginning_of_the_log>"
                }
            }
        }
    ]
}
```

# Pushing our task definition json file in AWS
We push the configuration file into ECS:
```
aws ecs register-task-definition --cli-input-json file://<name_of_your_task_definition_json_file>.json
```

# Adding the necessary permissions to the role you defined in the Task Definition file.
You need to provide some permissions to the role in order to execute the desired containter.

In the AWS Console page, go to:
- Your profile **<** Identity and Access Managment (IAM) or Security Credentials **<** Access managment **<** Roles **<** ecsTaskExecutionRole **<** Add Permissions

 This is a list of template permissions you should need in a default task (maybe for your case, you need more):
- **AmazonEC2ContainerRegistryFullAccess**: Be able to pull docker images from ECR, this allows the task to access the images stored in your repository.

- **AmazonECS_FullAccess**: Give access to the entire ECS ecosystem (tasks, clusters, services)

- **AmazonECSTaskExecutionRolePolicy**: It allows to stop, describe and manager ECS operations.

- **SecretsManagerReadWrite**: This is useful if your tasks need to access secrets stored in AWS Secrets Manager.

## Create a ECS cluster
We need to create a Elastic Container Service cluster in order to organize and manage the containerized application. With it, we can run our tasks easier.

We will use the flag capacity-providers because we want to create an application based on AWS Fargate:
```
aws ecs create-cluster --cluster-name <your_desired_cluster_name> --capacity-providers FARGATE --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1,base=1
```

## Test your created task:
Before automating our script, we should test the task.

- In the **Task Definition** section, go to your desired task, click over it, in the "Deploy" button, put "Run task". 

- Then configurate your service: placing **your desired cluster**.

- About the service, click **"Launch type"** and select Fargate. 

- The application type is a **"Task"** and put your task.

- Finally you can modify your **Networking** but, with the default arguments, should work.

- Your task will be running soon. Go to **Cloudwatch service** in order to see it logs.

- In the Log section, go to **Log Groups**. Select the applied log group in your task definition.

- You will **see the logs** of the last N executions. Select the last one, which represents the current execution.

## Creating Lambda function in order to trigger the task
In the automation process, we should to create a Lambda function. It will activate our task definition inside our cluster in the AWS Fargate service.

- Go to **Lambda AWS **service in AWS Console.

- Click on **"Create new function"**, select "Author from scratch", put your desired name for the task.

- Then, in the **Runtime** option, you should put the executor (for example, Python 3.8). 

- The architecture for AWS Fargate is **x86_64**. 

- By default, Lambda will create an **Executor Role** with permissions ot upload logs to AWS Cloudwatch Logs

After creating the lambda function, click over it and go to **"Code"** section inside the function.

With this **template** you can create easily the lambda function code that activates a task definition instance:

```
import json
import boto3

ecs = boto3.client('ecs')


def lambda_handler(event, context):
    # Define the parameters for starting the task
    cluster = '<your_desired_cluster_name_to_use>'
    task_definition = '<your_desired_task_definition_to_use>'
    launch_type = 'FARGATE'
    network_configuration = {
        'awsvpcConfiguration': {
            'subnets': ['<your_desired_subnet_to_use>'], #You can find it in AWS Console
            'securityGroups': ['<your_desired_security_group_to_use>'], #You can find it in AWS Console
            'assignPublicIp': 'ENABLED'
        }
    }

    # Start the Fargate task
    response = ecs.run_task(
        cluster=cluster,
        taskDefinition=task_definition,
        launchType=launch_type,
        networkConfiguration=network_configuration
    )

    # Print the response for debugging
    print(response)

    return {
        'statusCode': 200,
        'body': 'Fargate task started successfully.'
    }
```

## Setting the rule for your lambda function
We should set a rule in order to execute the lambda function based on a time logic.

- In the settings of the desired lambda function, go to **"Configuration"** tab.

- Click over the **Trigger** option, and create one based on the source **"EventBridge (Cloudwatch events)"**.

- You can select over **cron() or rate()** logic in order to set in which time you want to execute the lambda function.

## Done

You scheduled your lambda function and it is ready for run when your applied rule achieves its condition.

You will be able to see the logs of the lambda function in **AWS CloudWatch**, in the log group of the lambda function.

