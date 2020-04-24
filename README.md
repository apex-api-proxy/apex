![Apex Banner](https://github.com/apex-api-proxy/apex/blob/master/apex_banner.png)

<h1 align="center">Apex API Proxy</h1>

Welcome to Apex, an API proxy that provides one place to log and control microservices. Apex is built with Node.js/Express.js, Redis, TimescaleDB and React, and deploys in containers using Docker and Docker Compose. This repository contains the YAML file that specifies how to configure and deploy containers for the five core elements of Apex; to deploy locally, simply clone the repo and run `docker-compose up` from the root directory. To deploy using AWS Elastic Container Service (ECS), follow the detailed instructions below.

### 🏠 [Homepage](https://apex-api-proxy.github.io/)

## Table of Contents

* [Apex Usage](#apex-usage)
* [Install and Deploy](#install-and-deploy)
* [Deploying with AWS ECS](#deploying-with-aws-ecs)
* [Show Your Support](#show-your-support)

## Apex Usage

Apex provides an admin user interface for adding and editing service information, updating custom configuration, and basic log analysis. By default, the UI is accessible via port 1991.

To begin using Apex:

* Register your service for authentication and service discovery by other services
	* Choose a unique service name for your service
	* Choose any password
	* Enter the address for your service - this can be an IP address, or domain name
	  * By default, Apex will send a request with HTTPS to port 443 on your IP address
	* Click on submit, then save your token in a safe place
* Update your requests to other services so they are routed through Apex
	* Address all requests to the Apex IP address on port 1989 (http, not https)
	* For the `X-Apex-Authorization` header, include the value `Basic $YOUR_TOKEN`
	* For the `X-Apex-Responding-Service-Name` header, include the name of the service you want your request to be routed to
* If you wish to enable custom configuration e.g. for retry logic, you can do so in the ‘Configuration’ section of the admin UI. If custom configuration is specified between two services, Apex requires the following fields in the JSON file:
	* `timeout` (integer, in ms)
	* `max-retry-attempts` (integer)
	* `backoff` (integer, in ms)
	* You may add any other values required to support custom proxy middleware.
* By default, Apex adds a `X-Apex-Correlation-ID` header to all requests and responses. However, if you wish to enable tracing for workflows that span 3 or more services, then you must also update your code to propagate the `X-Apex-Correlation-ID` received in requests/responses to your service, onto subsequent requests/responses sent by your service.

## Install and Deploy

Apex can be deployed manually, but we recommend using [Docker](https://docs.docker.com/install/) and [Docker-Compose](https://docs.docker.com/compose/install/). To deploy with Docker, both of these must be installed.

To deploy Apex locally (in an on-premise server, DigitalOcean Droplet, etc.), clone _this_ repo. Then run `docker-compose up` from the root directory; Docker will download the necessary images from Docker Hub to build the service containers.

To deploy using AWS Elastic Container Service, follow the instructions below.

## Deploying with AWS ECS

You may follow the official AWS documentation for setting up ECS: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-ec2.html

If you prefer, you may reference the step-by-step walkthrough below (updated April 2020). Feel free to replace fields such as `region` with the one that works best for you:

1. Generate AWS Access Key ID and AWS Secret Access Key for your current AWS user, using the AWS IAM service. Keep these in a safe place in your system
   - https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey
2. Install the AWS CLI
   - https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
3. Configure the AWS CLI with your AWS Access Key ID and AWS Secret Access Key
   - https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html (scroll to bottom for OS-specific instructions)
4. Install the ECS CLI
   - https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
5. Configure the ECS CLI (1) - create a cluster configuration
   - `ecs-cli configure --cluster apex-ec2 --default-launch-type EC2 --config-name apex-ec2 --region us-east-2`
6. Configure the ECS CLI (2) - create a profile (replace $AWS_ACCESS_KEY_ID and $AWS_SECRET_ACCESS_KEY with your own)
   - `ecs-cli configure profile --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY --profile-name apex-ec2-profile`
7. Clone the apex-api-proxy/apex repo and change to the Apex root directory
   - `git clone https://github.com/apex-api-proxy/apex.git`
   - `cd apex`
8. Create keypair for EC2 instances for your current AWS user, and save it to the apex directory. This is used later for SSH-ing into the EC2 instance running our containers
   - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair
   - Name it apex-ec2.pem
   - Choose the pem format (used for SSH on Linux / Mac), rather than the ppk format (used for Windows)
9. Create your ECS cluster
   - `ecs-cli up --keypair apex-ec2 --capability-iam --size 1 --instance-type t2.medium --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
10. Obtain your EC2 instance’s public IP address
    - Locate and click into the ECS cluster you just created in the AWS console https://us-east-2.console.aws.amazon.com/ecs/home?region=us-east-2#/clusters
    - Click on the ECS instances tab
    - Click on the EC2 instance in the first row of results
    - In the EC2 instance info page, on the information panel in the bottom half, you will find the instance’s public IPv4 address, and its public DNS. Either will work
    - In the same information panel, also make a note of the VPC ID for the next step
11. Update the security group for your EC2 instance to allow incoming traffic on ports 1989, 1990 and 1991, as well as SSH on port 22
    - In the EC2 console, click on ‘Security Groups’ under the ‘Network & Security’ section
    - Click on the security group whose VPC ID matches the one from the previous step. Choose the one whose description says ECS Allowed Ports
    - Click on ‘Edit inbound rules’
    - Add 3 ‘Custom TCP’ rules, one for each of the ports 1989, 1990 and 1991. For the ‘Source’ field, choose ‘Anywhere’ or ‘My IP’, depending on your authorization needs
    - Also add 1 rule for SSH, and again choose the appropriate ‘Source’
    - Click on ‘Save rules’
12. View the Apex admin panel at http://$EC2_IP_ADDRESS:1991 (remember to use http, rather than https)
13. Deploy compose file to the cluster
    - `ecs-cli compose --file docker-compose-aws-ecs-ec2.yml --ecs-params ecs-params-ec2.yml up --create-log-groups --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
14. View running containers. You can see the task ID for your newly created containers in the output
    - `ecs-cli ps --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
15. View the container logs (replace \$YOUR_TASK_ID with the task ID in the output from the previous step)
    - `ecs-cli logs --task-id $YOUR_TASK_ID --follow --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
16. Another way to view the logs is on the ECS console
    - Go back to the console page for your current cluster: https://us-east-2.console.aws.amazon.com/ecs/home?region=us-east-2#/clusters/apex-ec2/services
    - Click on the ‘Tasks’ tab, then into the first row of results
    - In the table of containers at the bottom, expand any one row, find the ‘View logs in CloudWatch’ link and click on it
17. SSH into EC2 instance, for more control using the Docker CLI (replace the IP address in the below command with that of your cluster)
    - Change the file permissions of your keypair: `sudo chmod 400 apex-ec2.pem`
    - SSH into your EC2 instance: ssh -i "apex-ec2.pem" ec2-user@ec2-18-189-141-42.us-east-2.compute.amazonaws.com
18. Stop running containers, in preparation for the next step
    - `ecs-cli compose --file docker-compose-aws-ecs-ec2.yml --ecs-params ecs-params-ec2.yml down --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
19. Start a service with the same containers, so that containers restart themselves
    - `ecs-cli compose --file docker-compose-aws-ecs-ec2.yml --ecs-params ecs-params-ec2.yml service up --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
20. When done with service, remove it
    - `ecs-cli compose --file docker-compose-aws-ecs-ec2.yml --ecs-params ecs-params-ec2.yml service rm --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`
21. Finally, remove the ECS cluster
    - `ecs-cli down --force --cluster-config apex-ec2 --ecs-profile apex-ec2-profile`

## Show your support

Give a ⭐️ if you liked this project!

---
