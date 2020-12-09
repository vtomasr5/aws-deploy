# AWS Deploy

![Travis (.com) branch](https://img.shields.io/travis/com/jmsantorum/aws-deploy/main)
![Docker Pulls](https://img.shields.io/docker/pulls/jmsantorum/aws-deploy)

`aws-deploy` simplifies deployments on Amazon AWS by providing a convenience CLI tool for complex actions, which are executed pretty often.

##Key Features

- support for complex task definitions (e.g. multiple containers & task role)
- easily redeploy the current task definition (including `docker pull` of eventually updated images)
- deploy new versions/tags or all containers or just a single container in your task definition
- scale up or down by adjusting the desired count of running tasks
- add or adjust containers environment variables
- run one-off tasks from the CLI
- support for ECS & Code Deploy

## TL;DR

Deploy a new version of your service:

    $ aws-deploy ecs deploy my-cluster my-service --tag 1.2.3

Redeploy the current version of a service:

    $ aws-deploy ecs deploy my-cluster my-service

Scale up or down a service:

    $ aws-deploy ecs scale my-cluster my-service 4

Updating a cron job:

    $ aws-deploy ecs cron my-cluster my-task my-rule

Update a task definition (without running or deploying):

    $ aws-deploy ecs update my-cluster my-task

Deploy a new version of your application:

    $ aws-deploy code-deploy deploy my-application my-deployment-group --tag 1.2.3

Redeploy the current version of your application:

    $ aws-deploy code-deploy deploy my-application my-deployment-group

## Run via Docker

Instead of installing **aws-deploy** locally, which requires a Python environment, you can run **aws-deploy** via Docker. All versions are available on Docker Hub: https://hub.docker.com/repository/docker/jmsantorum/aws-deploy

Running **aws-deploy** via Docker is easy as:

    docker run jmsantorum/aws-deploy:1.0.0
    
In this example, the stable version 1.0.0 is executed. Alternatively you can use Docker tags ``master`` or ``latest`` for the latest stable version or Docker tag ``develop`` for the newest development version of **aws-deploy**.

Please be aware, that when running **aws-deploy** via Docker, the configuration - as described below - does not apply. You have to provide credentials and the AWS region via the command as attributes or environment variables:

    docker run jmsantorum/aws-deploy:1.0.0 aws-deploy ecs deploy my-cluster my-service --aws-region eu-west-1 --aws-access-key-id ABC --aws-secret-access-key ABC

## Configuration

As **aws-deploy** is based on boto3 (the official AWS Python library), there are several ways to configure and store the
authentication credentials. Please read the boto3 documentation for more details
(http://boto3.readthedocs.org/en/latest/guide/configuration.html#configuration). The simplest way is by running:

    $ aws configure

Alternatively you can pass the AWS credentials (via `--aws-access-key-id` and `--aws-secret-access-key`) or the AWS
configuration profile (via `--aws-profile`) as options when you run `aws-deploy`.

## Actions

Currently the following group of actions are supported:

### ECS

#### deploy

Redeploy a service either without any modifications or with a new image, environment variable and/or command definition.

    $ aws-deploy ecs deploy CLUSTER SERVICE [OPTIONS]

#### scale

Scale a service up or down and change the number of running tasks.

    $ aws-deploy ecs scale CLUSTER SERVICE DESIRED_COUNT [OPTIONS]

#### run

Run a one-off task based on an existing task-definition and optionally override command and/or environment variables.

     $ aws-deploy ecs run CLUSTER TASK [COUNT] [OPTIONS]
     
#### update

Update a task definition by creating a new revision to set a new image, environment variable and/or command definition, etc.

    $ aws-deploy ecs update TASK [OPTIONS]

#### cron (scheduled task)

Update a task definition and update a events rule (scheduled task) to use the new task definition.

    $ aws-deploy ecs cron CLUSTER TASK RULE [OPTIONS]

### Code Deploy

#### deploy

Redeploy a service either without any modifications or with a new image, environment variable and/or command definition.

    $ aws-deploy code-deploy deploy APPLICATION_NAME DEPLOYMENT_GROUP_NAME [OPTIONS]

## Usage

For detailed information about the available actions, arguments and options, run:

    $ aws-deploy --help
    $ aws-deploy ecs --help
    $ aws-deploy code-deploy --help

## Examples

All examples assume, that authentication has already been configured.

### Deployment

#### Simple Redeploy

To redeploy a service without any modifications, but pulling the most recent image versions, run the follwing command.
This will duplicate the current task definition and cause the service to redeploy all running tasks.::

    $ aws-deploy ecs deploy my-cluster my-service


#### Deploy a new tag

To change the tag for **all** images in **all** containers in the task definition, run the following command::

    $ aws-deploy ecs deploy my-cluster my-service -t 1.2.3


#### Deploy a new image

To change the image of a specific container, run the following command::

    $ aws-deploy ecs deploy my-cluster my-service --image webserver nginx:1.11.8

This will modify the **webserver** container only and change its image to "nginx:1.11.8".


#### Deploy several new images

The `-i` or `--image` option can also be passed several times::

    $ aws-deploy ecs deploy my-cluster my-service -i webserver nginx:1.9 -i application my-app:1.2.3

This will change the **webserver**'s container image to "nginx:1.9" and the **application**'s image to "my-app:1.2.3".

#### Deploy a custom task definition

To deploy any task definition (independent of which is currently used in the service), you can use the ``--task`` parameter. The value can be:

A fully qualified task ARN::

    $ aws-deploy ecs deploy my-cluster my-service --task arn:aws:ecs:eu-central-1:123456789012:task-definition/my-task:20

A task family name with revision::

    $ aws-deploy ecs deploy my-cluster my-service --task my-task:20

Or just a task family name. It this case, the most recent revision is used::

    $ aws-deploy ecs deploy my-cluster my-service --task my-task

.. important::
   ``ecs`` will still create a new task definition, which then is used in the service.
   This is done, to retain consistent behaviour and to ensure the ECS agent e.g. pulls all images.
   But the newly created task definition will be based on the given task, not the currently used task.


#### Set an environment variable

To add a new or adjust an existing environment variable of a specific container, run the following command::

    $ aws-deploy ecs deploy my-cluster my-service -e webserver SOME_VARIABLE SOME_VALUE

This will modify the **webserver** container definition and add or overwrite the environment variable `SOME_VARIABLE` with the value "SOME_VALUE". This way you can add new or adjust already existing environment variables.


#### Adjust multiple environment variables

You can add or change multiple environment variables at once, by adding the `-e` (or `--env`) options several times::

    $ aws-deploy ecs deploy my-cluster my-service -e webserver SOME_VARIABLE SOME_VALUE -e webserver OTHER_VARIABLE OTHER_VALUE -e app APP_VARIABLE APP_VALUE

This will modify the definition **of two containers**.
The **webserver**'s environment variable `SOME_VARIABLE` will be set to "SOME_VALUE" and the variable `OTHER_VARIABLE` to "OTHER_VALUE".
The **app**'s environment variable `APP_VARIABLE` will be set to "APP_VALUE".


#### Set environment variables exclusively, remove all other pre-existing environment variables

To reset all existing environment variables of a task definition, use the flag ``--exclusive-env`` ::

    $ aws-deploy ecs deploy my-cluster my-service -e webserver SOME_VARIABLE SOME_VALUE --exclusive-env

This will remove **all other** existing environment variables of **all containers** of the task definition, except for the variable `SOME_VARIABLE` with the value "SOME_VALUE" in the webserver container.


#### Set a secret environment variable from the AWS Parameter Store

.. important::
    This option was introduced by AWS in ECS Agent v1.22.0. Make sure your ECS agent version is >= 1.22.0 or else your task will not deploy.

To add a new or adjust an existing secret of a specific container, run the following command::

    $ aws-deploy ecs deploy my-cluster my-service -s webserver SOME_SECRET KEY_OF_SECRET_IN_PARAMETER_STORE

You can also specify the full arn of the parameter::

    $ aws-deploy ecs deploy my-cluster my-service -s webserver SOME_SECRET arn:aws:ssm:<aws region>:<aws account id>:parameter/KEY_OF_SECRET_IN_PARAMETER_STORE

This will modify the **webserver** container definition and add or overwrite the environment variable `SOME_SECRET` with the value of the `KEY_OF_SECRET_IN_PARAMETER_STORE` in the AWS Parameter Store of the AWS Systems Manager.


#### Set secrets exclusively, remove all other pre-existing secret environment variables

To reset all existing secrets (secret environment variables) of a task definition, use the flag ``--exclusive-secrets`` ::

    $ aws-deploy ecs deploy my-cluster my-service -s webserver NEW_SECRET KEY_OF_SECRET_IN_PARAMETER_STORE --exclusive-secret

This will remove **all other** existing secret environment variables of **all containers** of the task definition, except for the new secret variable `NEW_SECRET` with the value coming from the AWS Parameter Store with the name "KEY_OF_SECRET_IN_PARAMETER_STORE" in the webserver container.

#### Modify a command

To change the command of a specific container, run the following command::

    $ aws-deploy ecs deploy my-cluster my-service --command webserver "nginx"

This will modify the **webserver** container and change its command to "nginx". If you have 
a command that requries arugments as well, then you can simply specify it like this as you would normally do:

    $ aws-deploy ecs deploy my-cluster my-service --command webserver "ngnix -c /etc/ngnix/ngnix.conf"

This works fine as long as any of the arguments do not contain any spaces. In case arguments to the
command itself contain spaces, then you can use the JSON format:

$ aws-deploy ecs deploy my-cluster my-service --command webserver '["sh", "-c", "while true; do echo Time files like an arrow $(date); sleep 1; done;"]'

More about this can be looked up in documentation.
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definitions




#### Set a task role

To change or set the role, the service's task should run as, use the following command::

    $ aws-deploy ecs deploy my-cluster my-service -r arn:aws:iam::123456789012:role/MySpecialEcsTaskRole

This will set the task role to "MySpecialEcsTaskRole".

#### Ignore capacity issues

If your cluster is undersized or the service's deployment options are not optimally set, the cluster
might be incapable to run blue-green-deployments. In this case, you might see errors like these:

    ERROR: (service my-service) was unable to place a task because no container instance met all of
    its requirements. The closest matching (container-instance 123456-1234-1234-1234-1234567890) is
    already using a port required by your task. For more information, see the Troubleshooting
    section of the Amazon ECS Developer Guide.

There might also be warnings about insufficient memory or CPU.

To ignore these warnings, you can run the deployment with the flag ``--ignore-warnings``::

    $ aws-deploy ecs deploy my-cluster my-service --ignore-warnings

In that case, the warning is printed, but the script continues and waits for a successful
deployment until it times out.

#### Deployment timeout

The deploy and scale actions allow defining a timeout (in seconds) via the ``--timeout`` parameter.
This instructs ecs-deploy to wait for ECS to finish the deployment for the given number of seconds.

To run a deployment without waiting for the successful or failed result at all, set ``--timeout`` to the value of ``-1``.

### Scaling


#### Scale a service

To change the number of running tasks and scale a service up and down, run this command::

    $ aws-deploy ecs scale my-cluster my-service 4

### Running a Task


#### Run a one-off task

To run a one-off task, based on an existing task-definition, run this command::

    $ aws-deploy ecs run my-cluster my-task

You can define just the task family (e.g. ``my-task``) or you can run a specific revision of the task-definition (e.g.
``my-task:123``). And optionally you can add or adjust environment variables like this::

    $ aws-deploy ecs run my-cluster my-task:123 -e my-container MY_VARIABLE "my value"


#### Run a task with a custom command

You can override the command definition via option ``-c`` or ``--command`` followed by the container name and the
command in a natural syntax, e.g. no conversion to comma-separation required::

    $ aws-deploy ecs run my-cluster my-task -c my-container "python some-script.py param1 param2"

The JSON syntax explained above regarding modifying a command is also applicable here.


#### Run a task in a Fargate Cluster

If you want to run a one-off task in a Fargate cluster, additional configuration is required, to instruct AWS e.g. which
subnets or security groups to use. The required parameters for this are:

- launchtype
- securitygroup
- subnet
- public-ip

Example::

    $ aws-deploy ecs run my-fargate-cluster my-task --launchtype=FARGATE --securitygroup sg-01234567890123456 --subnet subnet-01234567890123456 --public-ip

You can pass multiple ``subnet`` as well as multiple ``securitygroup`` values. the ``public-ip`` flag determines, if the task receives a public IP address or not.
Please see ``ecs run --help`` for more details.

## Troubleshooting

If the service configuration in ECS is not optimally set, you might be seeing timeout or other errors during the deployment.

### Timeout

The timeout error means, that AWS ECS takes longer for the full deployment cycle then ecs-deploy is told to wait. The deployment itself still might finish successfully, if there are no other problems with the deployed containers.

You can increase the time (in seconds) to wait for finishing the deployment via the ``--timeout`` parameter. This time includes the full cycle of stopping all old containers and (re)starting all new containers. Different stacks require different timeout values, the default is 300 seconds.

The overall deployment time depends on different things:

- the type of the application. For example node.js containers tend to take a long time to get stopped. But nginx containers tend to stop immediately, etc.
- are old and new containers able to run in parallel (e.g. using dynamic ports)?
- the deployment options and strategy (Maximum percent > 100)?
- the desired count of running tasks, compared to
- the number of ECS instances in the cluster
