Infrastructure-provision
=========

This is Ansible role which can be used to automatically build and deploys simple web application called "WeatherGirl" to Amazon Web Services. 

The web application is composed of two docker containers, [*wg_frontend*](https://github.com/DavidDevOp/WeatherGirl-frontend) and [*wg_backend*](https://github.com/DavidDevOp/WeatherGirl-backend) which are build automatically from source codes stored locally.  The source codes are publicly available.

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
        


Warning
------------
The code provided is in the developement stage. For making it production-ready, further steps need to be considered:

- Using nginx web server to handle multiple requests. Currently (python flask web server is used).
- Using a Load ballancer for the web server.
- Restrict public access to the S3 bucket (requires to use boto3 python API for acessing AWS services, instead of the presently used http requests)
- Use AWS IAM user/role for the front- and back- end tasks to access the AWS resources, namely the MySQL database se and, for more flexible access-rights control. 
- Adopt a suitable policy for management of secret password(s) (e.g. AWS Secrets Manager). Currently container environment variables are used.

Requirements
------------
Toolchain:

- aws-cli >= 2.1.3
- docker >= 18.09.7
- ansible >= 2.10.3
- ansible-galaxy modules:
- community.general # as installed with ansible, needef for some docker operations
- community.aws (github repository merge pull #284 from main branch, see the note below)

    ansible galaxy collection community.aws version from "main" branch at gthub repository required for proper function of the script. The release version (2.10.3) of the community.aws.ecs_taskdefinition module does not function properly. Version that worked was form merge pull #284 
    https://github.com/ansible-collections/community.aws/pull/284
    At the time of publication, the correct version could installed locally by run the following command:
        ansible-galaxy collection install git+https://github.com/ansible-collections/community.aws.git


Python modules:

- python >=3.7
- boto3 >= 1.16.17


Role Variables
--------------

Environment variables of the Ansible Master Node (cluster build config) used in the role:

- AWS_ACESS_KEY_ID: Credentials for the DevOps user in the AWS account, with privileges to wide-enough manipulate the AWS resources (Creqte RDS instance, Create IAM roles, create VPN, subnets, InternetGateway, NAt Gateway, Edit route tables, Create ECS tasks, run ECS tasks,...)
- AWS_SECRET_ACCESS_KEY: ... the AWS credential secret ...
- WG_QUERY_SECRET: (Your_OpenWeatherMap_secret_API_key) - basic access available free of charge for registered users at openweathermap.org
- WG_DATABASE_USER: (username of the MySQL database server user) arbitrary MySQL server username - if new database instance will be created, this will be the name of the admin user
- WG_DATABASE_PASS: (password of the MySQL database server user)
- WG_S3_BUCKET_NAME: Provide a name of the S3 bucket to be created and used in this application, e.g. "muyawskybl"
- WG_DB_INSTANCE_NAME: Provide a name of the MySQL database to be created and used in this application, e.g. "muysqldb"
- WG_BACKEND_SOURCE_DIR: /home/r/wg_robot
- WG_FRONTEND_SOURCE_DIR=/home/r/web_pages
 
The description of role variables:
### vars file for infrastructure-provision
cluster_name: weather_girl
s3bucket_name: "{{ lookup('env', 'WG_S3_BUCKET_NAME') }}" # this should be taken from an environment variable
db_instance_name: "{{ lookup('env', 'WG_DB_INSTANCE_NAME') }}" # this should be taken from an environment variable
db_database_name: "cities" # the name of the database in the DB instance
size_of_cluster: 1 # how many running frontend containers are required. Only "1" os supported at this moment.
download_rate_expression: "rate(60 minutes)" # CloudWatchEvent rule task scheduling expression for downloading the data

### backend docker image parameters
be_source_folder: "{{ lookup('env', 'WG_BACKEND_SOURCE_DIR') }}"
be_repository_name: "wg_backend" # name of the ECR repository
be_source_image_name: "wg_backend" # local source image name to push
be_source_image_tag: "latest" # source image tag to use for push

### frontend docker image parameters
fe_source_folder: "{{ lookup('env', 'WG_FRONTEND_SOURCE_DIR') }}"
fe_repository_name: "wg_frontend" # name of the ECR repository
fe_source_image_name: "wg_frontend" # local source image name to push
fe_source_image_tag: "latest" # source image tag to use for push

### configuration of the VPC and network environment:
cloud_config:
  region: '{{ region }}'
  vpc_name: "{{ cluster_name }}-vpc"
  vpc_cidr: '10.0.0.0/16'
  subnets:
    - { subnet_name: '{{ cluster_name }}_subnet_public', subnet_cidr: '10.0.0.0/24', zone: '{{ region }}a' } # this subnet is for the web server
    - { subnet_name: '{{ cluster_name }}_subnet_private', subnet_cidr: '10.0.1.0/24', zone: '{{ region }}a' } # private SN for the DB and the backend task
    - { subnet_name: '{{ cluster_name }}_subnet_private_altAZ', subnet_cidr: '10.0.2.0/24', zone: '{{ region }}b' } # private SN for the DB in another Availability Zone
  internet_gateway: true                                           
  admin_username: ec2-user
  security_groups:
    - name: scalable-http
      rules:
        - {"proto":"tcp", "from_port":"80", "to_port":"80", "cidr_ip":"0.0.0.0/0"} # access web server (in production mode,)
        - {"proto":"tcp", "from_port":"3306", "to_port":"3306", "cidr_ip":"0.0.0.0/0"} # for access to the database from anywhere (on the private subnet)
        - {"proto":"tcp", "from_port":"443", "to_port":"443", "cidr_ip":"0.0.0.0/0"} # SSH access from the Internet
        - {"proto":"tcp", "from_port":"5000", "to_port":"5000", "cidr_ip":"0.0.0.0/0"} # access the web server in development stage
      description: "http - {{ cluster_name }}"


Dependencies
------------

There are no dependencies on other roles hosted on Galaxy.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

        - name: Build the Cloud Environment infrastructure
          hosts: localhost
          connection: local
          gather_facts: False
          vars_files:
          roles:
              - role: infrastructure-provision
                vars:
                  region: us-east-1
      
License
-------

MIT

Author Information
------------------

The code was written by David Rais, see my other projects at [github.com/daveraees](https://github.com/daveraees)