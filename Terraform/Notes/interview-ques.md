What is Terraform and how does it work internally?
Terraform is an infrastructure as code tool that allows you to define and provision infrastructure using declarative configuration files. It works by reading configuration files, building a dependency graph, and comparing desired state with the current state stored in a state file. It then creates an execution plan and applies only the required changes. It is used to automate infrastructure provisioning and ensure consistency across environments. A key limitation is state management complexity, especially in team environments. In real-world usage, you must secure and lock the state file to avoid concurrent modification issues.

What is a Terraform state file and why is it important?
The Terraform state file stores the current mapping of resources defined in configuration to real-world infrastructure. It helps Terraform determine what changes need to be applied during plan and apply operations. It is used to track resource metadata like IDs and dependencies. A limitation is that state files can become large and sensitive, containing secrets or infrastructure details. In practice, storing state remotely with locking mechanisms like S3 and DynamoDB prevents corruption during concurrent operations.

What is the difference between terraform plan and terraform apply?
Terraform plan generates an execution plan by comparing the desired configuration with the current state without making changes. Terraform apply executes that plan and creates, updates, or deletes resources accordingly. Plan is used for preview and validation, while apply performs actual infrastructure changes. A limitation is that the plan can become outdated if infrastructure changes between plan and apply. In real-world scenarios, teams often use CI pipelines to ensure plan and apply happen close together to avoid drift.

What are providers in Terraform?
Providers are plugins that allow Terraform to interact with APIs of cloud platforms or services like AWS or Azure. They define resource types and operations such as create, update, and delete. Providers work by translating Terraform configuration into API calls. They are used to manage infrastructure across multiple platforms in a unified way. A limitation is dependency on provider versions, which can introduce breaking changes. In practice, version pinning is important to maintain stability in production environments.

What are resources in Terraform?
Resources are the fundamental building blocks in Terraform that represent infrastructure components like virtual machines or databases. They are defined in configuration files and mapped to real-world objects via providers. Terraform manages their lifecycle including creation, updates, and deletion. They are used to declaratively define infrastructure. A limitation is that some resource changes may force recreation instead of in-place updates. In real-world cases, this can cause downtime if not handled with lifecycle rules like create_before_destroy.

What is a module in Terraform?
A module is a reusable collection of Terraform configurations that encapsulates resources and logic. Modules work by allowing input variables and output values to make them reusable across different environments. They are used to standardize infrastructure patterns and reduce duplication. A limitation is that deeply nested modules can become hard to debug and maintain. In practice, keeping modules simple and well-documented helps teams reuse them effectively.

What are variables and outputs in Terraform?
Variables allow parameterization of Terraform configurations so values can be customized without changing code. Outputs expose information about created resources such as IP addresses or IDs after deployment. Variables work by being passed at runtime or defined in files, while outputs are stored in state and accessible externally. They are used for flexibility and integration between modules. A limitation is that sensitive variables must be handled carefully to avoid leaks. In real-world usage, marking variables as sensitive and avoiding logging them is critical.

What is Terraform backend and why is it used?
A backend defines where Terraform stores its state file and how operations like locking are handled. It works by configuring storage options like local, S3, or remote services. Backends are used to enable collaboration and prevent state conflicts. A limitation is that backend changes require migration steps and can disrupt workflows. In practice, remote backends with locking are preferred for team environments to ensure consistency.

What is resource dependency and how does Terraform manage it?
Resource dependency defines the order in which resources should be created or modified. Terraform automatically infers dependencies through references and builds a dependency graph. It ensures correct execution order during apply operations. This is used to avoid issues like creating a resource before its prerequisite exists. A limitation is that implicit dependencies may not always be detected correctly. In real-world cases, explicit depends_on is used to enforce order when required.

What is Terraform lifecycle and when would you use it?
Lifecycle is a block in Terraform that controls how resources are created, updated, or destroyed. It includes options like create_before_destroy and prevent_destroy. It works by modifying default behavior of resource operations. It is used to handle cases where downtime must be avoided or accidental deletion should be prevented. A limitation is that misuse can lead to orphaned resources or unexpected costs. In practice, lifecycle rules must be carefully tested to ensure they align with infrastructure requirements.

What is Terraform workspace and how is it used?
A workspace allows multiple instances of the same Terraform configuration to be managed separately. It works by maintaining separate state files for each workspace. It is used for managing environments like dev, staging, and production. A limitation is that workspace usage can become confusing in large projects. In real-world setups, many teams prefer separate state backends or directories instead of relying heavily on workspaces.

What is drift in Terraform and how do you handle it?
Drift occurs when the actual infrastructure differs from the Terraform state due to manual changes or external updates. Terraform detects drift during plan by comparing state with real infrastructure. It is handled by reapplying configurations or importing changes. Drift management is used to maintain consistency and predictability. A limitation is that frequent drift can make infrastructure unreliable. In practice, restricting manual changes and enforcing IaC-only updates reduces drift.

What is Terraform import and when would you use it?
Terraform import allows existing infrastructure resources to be brought under Terraform management. It works by mapping an existing resource to a Terraform configuration and updating the state file. It is used when adopting Terraform for already provisioned infrastructure. A limitation is that import does not generate configuration automatically. In real-world cases, engineers must manually write matching configuration to avoid inconsistencies.

What is remote state and how is it shared?
Remote state refers to storing Terraform state in a shared location like S3 or Terraform Cloud. It works by enabling multiple users or systems to access and update the same state safely. It is used for collaboration and integration across teams. A limitation is managing access control and state security. In practice, using encryption and IAM policies ensures that only authorized users can access the state.

What are provisioners in Terraform and should they be used?
Provisioners are used to execute scripts or commands on resources after creation. They work by running local or remote scripts during resource lifecycle events. They are used for tasks like configuration or bootstrapping. A limitation is that provisioners are not idempotent and can lead to unpredictable results. In real-world scenarios, it is recommended to avoid provisioners and use configuration management tools instead.

What is Terraform locking and why is it important?
Terraform locking prevents multiple users from modifying the state simultaneously. It works by using backend-supported locking mechanisms like DynamoDB for S3. It is used to avoid race conditions and state corruption. A limitation is that locks can become stuck if operations fail. In practice, teams need procedures to safely release locks without damaging the state.

What are data sources in Terraform?
Data sources allow Terraform to fetch information from existing infrastructure instead of creating new resources. They work by querying provider APIs and returning data for use in configurations. They are used to integrate with already existing systems. A limitation is dependency on external system availability. In real-world use, failures in data sources can block deployments, so fallback strategies are important.

What is Terraform graph and why is it useful?
Terraform graph visualizes the dependency graph of resources. It works by generating a directed graph of resource relationships. It is used for debugging and understanding execution order. A limitation is that large graphs become difficult to interpret. In real-world scenarios, it is mainly used during troubleshooting complex dependencies.

What is versioning in Terraform and why does it matter?
Versioning in Terraform applies to providers, modules, and Terraform itself. It works by specifying version constraints in configuration files. It is used to ensure compatibility and avoid breaking changes. A limitation is that strict versioning can prevent upgrades. In practice, teams test upgrades in lower environments before updating production configurations.

What are common performance considerations in Terraform?
Terraform performance depends on resource count, provider efficiency, and API rate limits. It works by executing operations in parallel where possible based on dependencies. It is used to optimize infrastructure deployment speed. A limitation is that excessive parallelism can hit API limits or cause failures. In real-world usage, tuning parallelism and batching changes improves performance and reliability.
