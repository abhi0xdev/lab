What is AWS and how is it used in DevOps?
AWS is a cloud platform that provides on-demand infrastructure services like compute, storage, and networking. In DevOps, it is used to automate infrastructure provisioning, deployment pipelines, and monitoring systems. It works by offering managed services such as EC2, S3, and CodePipeline that integrate together. It is used because it reduces manual work and enables scalability and automation. A limitation is vendor lock-in, and in real-world usage, misconfigured resources can lead to high unexpected costs.

What is EC2 and how do you use it in deployments?
EC2 is a virtual server service that allows you to run applications on AWS infrastructure. It works by launching instances with chosen OS images and configurations, which can be accessed via SSH. It is used to host applications, run scripts, or act as part of a deployment pipeline. A limitation is that instance management requires manual scaling unless automated. In practice, improper security group settings can expose instances to the internet.

What are Security Groups in AWS?
Security Groups are virtual firewalls attached to EC2 instances that control inbound and outbound traffic. They work by defining rules based on ports, protocols, and IP ranges. They are used to secure access to instances and restrict unnecessary exposure. A limitation is that they are stateful and sometimes confusing for beginners when debugging connectivity issues. In real-world scenarios, overly permissive rules like 0.0.0.0/0 can create serious security risks.

What is S3 and where is it used in DevOps workflows?
S3 is an object storage service used to store files like logs, artifacts, and backups. It works by storing data in buckets with unique keys and supports versioning and lifecycle policies. It is used in DevOps for artifact storage, static website hosting, and backup strategies. A limitation is eventual consistency in some operations, which can affect immediate reads. In practice, incorrect bucket permissions can lead to public data leaks.

What is IAM and why is it important?
IAM is a service that manages access control for AWS resources using users, roles, and policies. It works by attaching JSON policies that define allowed or denied actions. It is used to enforce least privilege access and secure resources. A limitation is complexity in managing policies at scale. In real-world use, overly broad permissions like AdminAccess can lead to security breaches.

What is a CI/CD pipeline in AWS?
A CI/CD pipeline automates the process of building, testing, and deploying code changes. It works using services like CodePipeline, CodeBuild, and CodeDeploy integrated together. It is used to reduce manual intervention and ensure faster, reliable releases. A limitation is that debugging pipeline failures can be complex. In practice, missing rollback strategies can cause downtime during failed deployments.

What is CodeBuild and how does it work?
CodeBuild is a fully managed build service that compiles source code and runs tests. It works by executing commands defined in a buildspec file inside a container environment. It is used to automate build processes without managing servers. A limitation is limited customization compared to self-hosted solutions. In real-world scenarios, incorrect environment variables can cause build failures.

What is CodeDeploy and where is it used?
CodeDeploy is a deployment service that automates application deployments to EC2 or Lambda. It works by using deployment groups and lifecycle hooks defined in an appspec file. It is used to ensure consistent deployments and reduce downtime. A limitation is complexity in setting up deployment strategies like blue-green. In practice, incorrect hook scripts can fail deployments midway.

What is CloudFormation?
CloudFormation is an infrastructure-as-code service that allows defining AWS resources using templates. It works by reading JSON or YAML templates and provisioning resources accordingly. It is used to automate infrastructure creation and ensure consistency. A limitation is that debugging stack failures can be difficult. In real-world use, improper dependency definitions can cause stack creation errors.

What is Terraform and how is it different from CloudFormation?
Terraform is an infrastructure-as-code tool that supports multiple cloud providers including AWS. It works by using HCL language and maintaining a state file for resource tracking. It is used for cross-cloud deployments and better modularization. A limitation is state file management which can lead to conflicts. In practice, storing state remotely with locking is necessary to avoid corruption.

What is Auto Scaling in AWS?
Auto Scaling automatically adjusts the number of EC2 instances based on demand. It works by defining scaling policies triggered by metrics like CPU utilization. It is used to maintain performance and reduce cost by scaling dynamically. A limitation is delayed scaling response depending on metrics and cooldown periods. In real-world use, incorrect thresholds can cause frequent scaling or outages.

What is a Load Balancer in AWS?
A Load Balancer distributes incoming traffic across multiple targets like EC2 instances. It works by routing requests based on rules and health checks. It is used to improve availability and fault tolerance. A limitation is additional cost and configuration complexity. In practice, improper health check configuration can remove healthy instances from rotation.

What is CloudWatch?
CloudWatch is a monitoring and logging service for AWS resources. It works by collecting metrics, logs, and events from services and applications. It is used for alerting, troubleshooting, and performance monitoring. A limitation is limited log retention unless configured. In real-world scenarios, missing alarms can delay detection of critical failures.

What is VPC in AWS?
VPC is a virtual network that isolates AWS resources within a defined IP range. It works by creating subnets, route tables, and gateways for traffic control. It is used to design secure and scalable network architectures. A limitation is complexity in networking setup. In practice, incorrect route table configuration can block connectivity.

What is Docker and how is it used with AWS?
Docker is a containerization platform that packages applications with dependencies. It works by creating container images that run consistently across environments. It is used with AWS services like ECS or EKS for scalable deployments. A limitation is managing container security and image size. In real-world use, unoptimized images can increase deployment time.

What is ECS vs EKS?
ECS is AWS’s native container orchestration service, while EKS is a managed Kubernetes service. ECS works with task definitions and is simpler to set up. EKS uses Kubernetes APIs and provides more flexibility and ecosystem support. ECS is used for simplicity, while EKS is used for portability and advanced orchestration. A limitation of EKS is higher complexity and cost, and in practice, cluster misconfiguration can impact workloads.

What is Blue-Green Deployment?
Blue-Green deployment is a strategy where two environments are maintained for release. It works by switching traffic from the old version to the new version after validation. It is used to reduce downtime and enable rollback. A limitation is increased resource usage since two environments are needed. In real-world use, data synchronization between environments must be handled carefully.

What is AWS Lambda?
Lambda is a serverless compute service that runs code without managing servers. It works by triggering functions based on events like API calls or file uploads. It is used for event-driven architectures and cost efficiency. A limitation is execution time limits and cold start latency. In practice, large dependencies can increase cold start times.

What is API Gateway?
API Gateway is a service to create and manage APIs. It works by routing HTTP requests to backend services like Lambda or EC2. It is used to expose services securely and manage traffic. A limitation is added latency compared to direct calls. In real-world use, improper throttling settings can lead to abuse or outages.

What is Secrets Manager and why is it used?
Secrets Manager stores and manages sensitive data like passwords and API keys. It works by encrypting secrets and allowing controlled access via IAM. It is used to avoid hardcoding credentials in code. A limitation is additional cost compared to simpler storage. In practice, improper rotation policies can leave secrets exposed for long periods.
