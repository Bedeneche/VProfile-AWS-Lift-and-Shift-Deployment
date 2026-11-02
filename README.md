# VProfile AWS Lift and Shift

## Project overview

In this project, I deploy the VProfile Java web application on AWS using a lift and shift approach. I run the application and its backend services on four EC2 instances, put an Application Load Balancer (ALB) in front of the application, and use Amazon S3 to store the application package. I use Route 53 to give the backend services names that the application can resolve inside the VPC.

The four instances are App01 for the web application, DB01 for the database, MC01 for Memcached, and RMQ01 for RabbitMQ. The ALB is an AWS managed service, not another EC2 instance. I created an AMI from App01, used it to create a launch template named `vprofile app`, and used that launch template for my Auto Scaling group. I also enabled stickiness on the target group so requests from a client stay associated with the same registered target for the configured stickiness duration.

This repository contains the VProfile application, bootstrap scripts, Ansible examples, and a Jenkins pipeline. It does not currently contain code to create the AWS resources. I create the VMs, security groups, ALB, Route 53 records, S3 bucket, IAM roles, and Auto Scaling group separately in AWS.

## How I designed the environment

| Component | How I use it |
| --- | --- |
| ALB and target group | The ALB receives HTTP requests on port 80 from my allowed client IPv4 address and forwards them to app instances on port 8080. I enabled target group stickiness. |
| App01 | Runs the VProfile WAR on Ubuntu with Java 17 and Tomcat 10. |
| DB01 | Runs MariaDB/MySQL-compatible database service and stores the `accounts` schema on port 3306. |
| MC01 | Runs Memcached on port 11211. |
| RMQ01 | Runs RabbitMQ AMQP on port 5672. |
| S3 bucket | Holds the WAR file that I build and deploy. |
| Route 53 | Resolves the backend hostnames to their private IP addresses. I can also create a public alias for the application that points to the ALB. |

For an internet-facing ALB, I place it in public subnets across at least two Availability Zones. I place the EC2 instances in private subnets where practical. The ALB needs network access to App01 on port 8080, and App01 needs access to the backend service ports.

## Security groups I use

I replace `YOUR_PUBLIC_IPV4/32` with my current public IPv4 CIDR. For application-to-application traffic, I use security group references so that changes to private IP addresses do not require rule updates.

| Security group | Inbound access |
| --- | --- |
| ALB SG | TCP 80 from `YOUR_PUBLIC_IPV4/32` for my restricted lab access. If I want the site to be public, I change the source accordingly. |
| App SG | TCP 8080 from the ALB SG and TCP 22 from `YOUR_PUBLIC_IPV4/32` for SSH administration. |
| Backend SG | TCP 3306, 11211, and 5672 from the App SG, plus TCP 22 from `YOUR_PUBLIC_IPV4/32` for SSH administration. |

I avoid exposing the database, Memcached, and RabbitMQ ports to the internet. An all-TCP rule between backend instances is broader than the application needs, so I only add it if my lab specifically requires backend-to-backend traffic and scope it to the backend SG. For a temporary lab I use the requested HTTP-only ALB listener; for a public deployment I would configure HTTPS and a certificate.

## What is in this repository

- `src/` contains the Java application, JSP views, resources, tests, and SQL database dumps.
- `src/main/resources/application.properties` contains the connection settings packaged into the WAR.
- `userdata/tomcat_ubuntu.sh` installs Java and Tomcat 10 on Ubuntu.
- `userdata/mysql.sh` installs MariaDB and initializes the database.
- `userdata/memcache.sh` installs and starts Memcached.
- `userdata/rabbitmq.sh` and `al2023rmq.repo` install RabbitMQ on Amazon Linux 2023.
- `userdata/application.properties` is an example configuration using short Route 53 names.
- `ansible/` contains older Ansible setup and deployment examples.
- `Jenkinsfile` builds, tests, analyzes, and publishes the application to Nexus.
- `pom.xml` defines the Maven WAR build.

## What I need before deployment

- An AWS account with access to create EC2 instances, security groups, an ALB and target group, S3, IAM roles, and Route 53 records.
- Four EC2 instances in a VPC with network rules that allow the connections described above.
- Java 17 and Maven 3 on the machine where I build the application. The POM sets the Java source and target versions to 17.
- AWS CLI credentials for the identity uploading the artifact, and an EC2 instance profile that lets App01 download it from S3.
- Route 53 records for `db01`, `mc01`, and `rmq01` that App01 can resolve inside the VPC, or updated application properties that match the names I create.

## How I build and upload the application

From the repository root, I build the WAR with Maven:

```sh
mvn clean package
```

The build creates `target/vprofile-v2.war`. I upload it to a versioned key in my S3 bucket, replacing the bucket name with my own:

```sh
aws s3 cp target/vprofile-v2.war s3://YOUR_ARTIFACT_BUCKET/vprofile/vprofile-v2.war
```

I use an IAM identity and role for the upload rather than placing AWS access keys in scripts or on the EC2 instances. App01 needs permission to read only the artifact objects it deploys. I add `s3:ListBucket` only if my deployment method needs to list bucket contents.

## How I deploy the application

1. I create the VPC and subnets, then create the ALB, application, and backend security groups using the rules above.
2. I launch DB01, MC01, and RMQ01 with the backend SG attached. I run the matching bootstrap scripts from `userdata/` and check that each service is active and listening on its private interface and port.
3. I initialize the `accounts` database using `src/main/resources/db_backup.sql`. I make sure the database username and password match the application configuration.
4. I create private Route 53 records for `db01`, `mc01`, and `rmq01`, pointing to the backend private IPs, and associate the private hosted zone with my VPC.
5. I launch App01 with the application SG and an instance profile that can read the WAR from S3. I run `userdata/tomcat_ubuntu.sh`, configure the application properties, and deploy the WAR as `ROOT.war` under `/var/lib/tomcat10/webapps/` on Ubuntu's packaged Tomcat 10.
6. I verify that App01 responds at `http://APP01_PRIVATE_IP:8080/` from an allowed network path. I check the Tomcat logs and confirm that the application can connect to the database, Memcached, and RabbitMQ.
7. I create an HTTP target group on port 8080, register App01 for the initial check, and configure a health check path that returns a successful response. I create an ALB listener on port 80 that forwards traffic to this target group.
8. I confirm that the target is healthy and open the ALB DNS name from my allowed client IP. I can create a Route 53 public alias pointing my application name to the ALB.
9. I create an AMI from App01, create the `vprofile app` launch template from that AMI, and use the launch template for my Auto Scaling group. I attach the target group to the group so instances can receive ALB traffic. I verify that replacement instances start and pass the ALB health check.

For the Auto Scaling group, I avoid baking changing secrets or environment-specific addresses into the AMI. I prefer to provide configuration at launch and let each instance retrieve the artifact through its instance role.

## Application settings and items I review

The application configuration packaged in the WAR is `src/main/resources/application.properties`. `userdata/application.properties` is an example. The source properties currently use `db01.vprofile.in`, `mc01.vprofile.in`, and `rmq01.vprofile.in`. I make sure my private DNS zone uses those names, or change the properties to match my own records and rebuild the WAR.

Some repository scripts contain example credentials such as `admin123` and `test/test`. I treat these as lab values and replace them before deployment. I do not commit real credentials. The current `userdata/mysql.sh` clones another repository branch and imports its dump; for a controlled deployment, I use the database dump from this checkout and confirm that the database grants match the application host. I review each script before running it, especially where it configures broad bind addresses or default credentials.

## How I check the deployment

- [ ] App01 resolves all three backend service names to private IP addresses.
- [ ] App01 can connect to DB01 on port 3306, MC01 on port 11211, and RMQ01 on port 5672.
- [ ] Tomcat is active and listening on port 8080, and the WAR is deployed as the root context.
- [ ] The ALB target group reports App01 as healthy on port 8080.
- [ ] Target group stickiness is enabled with the duration I selected.
- [ ] I can reach the ALB on port 80 from the client IPv4 address I allowed.
- [ ] App01 can download the WAR from S3 using its instance profile, without static AWS keys.
- [ ] A replacement instance launched from the `vprofile app` launch template can retrieve the artifact, start Tomcat, and pass the ALB health check.

## Current project limitations

The repository does not include infrastructure-as-code templates or an AWS inventory, so it cannot create or confirm the live AWS resources by itself. The included Ansible playbooks use older Tomcat 8 paths and Nexus URLs, which do not match my Tomcat 10 and S3 deployment flow without changes. The Jenkins pipeline also publishes to Nexus; it does not currently upload to S3 or deploy to EC2. The repository does not define an HTTPS listener, certificate setup, or automated AWS health checks.
