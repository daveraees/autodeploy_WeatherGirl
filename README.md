autodeploy_WeatherGirl
=========

This is repository contains an Ansible playbook which can be used to automatically build and deploy simple web application called "WeatherGirl" to Amazon Web Services. 

The web application is composed of two docker containers, [*wg_frontend*](https://github.com/daveraees/WeatherGirl-frontend) and [*wg_backend*](https://github.com/daveraees/WeatherGirl-backend) which are build automatically from source codes stored locally.  The source codes are publicly available.

With the provided AWS account access credentials for an IAM DevOps role, it creates all the services required for running the WeatherGirl application:
- network environment 
    - dedicated VPC, one public and two private subnets, 
    - attaches NAT gateway in a pair of private subnets, 
    - Internet Gateway in public subnet,
    - route tables, 
- MySQL database instance
    - in pair of private subnets
- S3 bucket
    - for storage of the weather information files
- A cluster in AWS Elastic Container Service (ECS), where it registers
    - background taks 
        - to periodicaly download of the weather information from public OpenWeatherMap service 
        - brief description: ECS-Fargate launch, with Scheduled Task triggered at fixed rate, running in one of the private subnets
    - web server running the (*wg_frontend*) container, that displays the downloaded weather information
        - in public subnet, has public "Elastic IP", and is accessible from the Internet at port 5000 
        

Requirements
------------
Toolchain:

- aws-cli >= 2.1.3
- docker >= 18.09.7
- ansible >= 2.10.3
- ansible-galaxy modules:
- community.general # as installed with ansible, needef for some docker operations
- community.aws (github repository merge pull #284 from main branch, see the note below)

    ansible galaxy collection community.aws version from "main" branch at GitHub.com repository is required for proper function of the script. The release version (2.10.3) of the community.aws.ecs_taskdefinition module does not function properly. Version that worked was form merge pull #284 
    https://github.com/ansible-collections/community.aws/pull/284
    At the time of publication, the correct version could installed locally by run the following command:
        ansible-galaxy collection install git+https://github.com/ansible-collections/community.aws.git


Python modules:

- python >= 3.7
- boto3 >= 1.16.17


Role Variables
--------------

Environment variables of the Ansible Master Node (cluster build config) used in the "autodeploy" role:

- AWS_ACCESS_KEY_ID: Credentials for the DevOps user in the AWS account, with privileges to wide-enough manipulate the AWS resources (Creqte RDS instance, Create IAM roles, create VPN, subnets, InternetGateway, NAt Gateway, Edit route tables, Create ECS tasks, run ECS tasks,...)
- AWS_SECRET_ACCESS_KEY: ... the AWS credential secret ...
- WG_QUERY_SECRET: (Your_OpenWeatherMap_secret_API_key) - basic access available free of charge for registered users at openweathermap.org
- WG_CITY_COUNT_LIMIT: Maximum number of cities to queue for weather info periodic downloads. Maximu is about 60, due to request limit on the OpenWeatherMap data provider (max 1000 requests per day for free subscription)
- WG_DATABASE_USER: (username of the MySQL database server user) arbitrary MySQL server username - if new database instance will be created, this will be the name of the admin user
- WG_DATABASE_PASS: (password of the MySQL database server user)
- WG_S3BUCKET_NAME: Provide a name of the S3 bucket to be created and used in this application, e.g. "muyawskybl"
- WG_DB_INSTANCE_NAME: Provide a name of the MySQL database to be created and used in this application, e.g. "muysqldb"
- WG_BACKEND_SOURCE_DIR: /absolute/path/to/clone/of/backend/repository/directory
- WG_FRONTEND_SOURCE_DIR: /absolute/path/to/clone/of/frontend/repository/directory
 
The description of variables in file /deploy/roles/autodeploy/vars/main.yml:
### vars file for autodeploy
- cluster_name: weather_girl
- s3bucket_name: "{{ lookup('env', 'WG_S3BUCKET_NAME') }}" # this should be taken from an environment variable
- db_instance_name: "{{ lookup('env', 'WG_DB_INSTANCE_NAME') }}" # this should be taken from an environment variable
- db_database_name: "cities" # the name of the database in the DB instance
- size_of_cluster: 1 # how many running frontend containers are required. Only "1" os supported at this moment.
- download_rate_expression: "rate(60 minutes)" # CloudWatchEvent rule task scheduling expression for downloading the data

#### backend docker image parameters
- be_source_folder: "{{ lookup('env', 'WG_BACKEND_SOURCE_DIR') }}"
- be_repository_name: "wg_backend" # name of the ECR repository
- be_source_image_name: "wg_backend" # local source image name to push
- be_source_image_tag: "latest" # source image tag to use for push

#### frontend docker image parameters
- fe_source_folder: "{{ lookup('env', 'WG_FRONTEND_SOURCE_DIR') }}"
- fe_repository_name: "wg_frontend" # name of the ECR repository
- fe_source_image_name: "wg_frontend" # local source image name to push
- fe_source_image_tag: "latest" # source image tag to use for push

#### configuration of the VPC and network environment:
- cloud_config: # dictionary with the cloud configuration, please see the 

Dependencies
------------

There are no dependencies on other roles hosted on Galaxy. 

The playbook uses *docker* commnad to manipulate container images, therefore it must be able to invoke the commands. Docker command needs superuser privileges. You can setup your Linux system following the guide: [Post-installation steps for Linux](https://docs.docker.com/engine/install/linux-postinstall/)

Test the playbook
----------------
The sample playbook *deploy/deploy.yml* should be edited prior to execution. The AWS region for the cluster deployment should be specified in the *region* variable. This example playbook deploys the WeatherGirl app to the us-east-1 region (N. Virginia, US).

        - name: Build the Cloud Environment infrastructure
          hosts: localhost
          connection: local
          gather_facts: False
          vars_files:
          roles:
              - role: autodeploy
                vars:
                  region: eu-west-1

The playbook *deploy/deploy.yml* can be executed with the following command from the playbook's directory:

        $ ansible-playbook deploy.yml

Warning
------------
The code provided is in the developement stage, the author accepts no reliability for potential damage caused by using it. For making it production-ready, further steps need to be considered:

- Using nginx web server to handle multiple requests. Currently (python flask web server is used).
- Using a Load ballancer for the web server.
- Adjust public access to the S3 bucket (it has currently *public-read* permission)
- Use AWS IAM user/role for the front- and back- end tasks to access the AWS resources, namely the MySQL database se and, for more flexible access-rights control. 
- Adopt a suitable policy for management of secret password(s) (e.g. AWS Secrets Manager). Currently container environment variables are used.
      
License
-------

MIT

Author Information
------------------

The code was written by David Rais, see my other projects at [github.com/daveraees](https://github.com/daveraees)
