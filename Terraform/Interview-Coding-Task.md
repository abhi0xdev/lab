Below are **real Terraform interview coding tasks** you’re very likely to face at a mid-level (especially for DevOps/Data Engineer roles). Each includes **problem → approach → solution → what interviewer looks for → common mistakes**.

---

# 🔹 Task 1: Create a Reusable EC2 Module

### 🧠 Problem

Design a reusable Terraform module to create EC2 instances with:

* AMI
* Instance type
* Tags

---

### ✅ Expected Approach

* Use **variables**
* Keep module generic
* Return useful outputs

---

### 💻 Solution

**modules/ec2/main.tf**

```hcl id="ec2main01"
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type

  tags = var.tags
}
```

**modules/ec2/variables.tf**

```hcl id="ec2vars01"
variable "ami" {}
variable "instance_type" {}
variable "tags" {
  type = map(string)
}
```

**modules/ec2/outputs.tf**

```hcl id="ec2out01"
output "instance_id" {
  value = aws_instance.this.id
}
```

**Root module**

```hcl id="ec2root01"
module "web" {
  source = "./modules/ec2"

  ami           = "ami-123"
  instance_type = "t2.micro"
  tags = {
    Name = "web-server"
  }
}
```

---

### 🎯 What Interviewer Looks For

* Clean module structure
* Reusability
* Proper variable usage

---

### ❌ Common Mistakes

* Hardcoding values
* No outputs
* Not using maps for tags

---

# 🔹 Task 2: Create Multiple EC2 Instances Dynamically

### 🧠 Problem

Create multiple EC2 instances with different names and types.

---

### ✅ Approach

* Use `for_each` (preferred over count)

---

### 💻 Solution

```hcl id="multiec201"
variable "instances" {
  default = {
    web1 = "t2.micro"
    web2 = "t2.small"
  }
}

resource "aws_instance" "web" {
  for_each = var.instances

  ami           = "ami-123"
  instance_type = each.value

  tags = {
    Name = each.key
  }
}
```

---

### 🎯 Interview Insight

* Must explain **why `for_each` > `count`**

  * Prevents unnecessary recreation
  * Stable resource identity

---

# 🔹 Task 3: Setup Remote Backend with Locking

### 🧠 Problem

Store Terraform state remotely using S3 and enable locking.

---

### 💻 Solution

```hcl id="backend01"
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "tf-lock"
    encrypt        = true
  }
}
```

---

### 🎯 What Interviewer Checks

* Understanding of:

  * State locking
  * Team collaboration
  * Security (encryption)

---

### ❌ Mistake

* Forgetting DynamoDB → no locking

---

# 🔹 Task 4: Use Data Source for Dynamic AMI

### 🧠 Problem

Fetch latest Amazon Linux AMI dynamically.

---

### 💻 Solution

```hcl id="datasrc01"
data "aws_ami" "latest" {
  most_recent = true

  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.latest.id
  instance_type = "t2.micro"
}
```

---

### 🎯 Interview Insight

* Avoid hardcoding AMIs
* Shows production awareness

---

# 🔹 Task 5: Conditional Resource Creation

### 🧠 Problem

Create EC2 only if a flag is true.

---

### 💻 Solution

```hcl id="cond01"
variable "create_instance" {
  default = true
}

resource "aws_instance" "web" {
  count = var.create_instance ? 1 : 0

  ami           = "ami-123"
  instance_type = "t2.micro"
}
```

---

### 🎯 Insight

* Demonstrates environment-based deployments

---

# 🔹 Task 6: Create VPC + Subnet + EC2 (Dependency Graph)

### 🧠 Problem

Build basic infra:

* VPC
* Subnet
* EC2 inside subnet

---

### 💻 Solution

```hcl id="vpc01"
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
}
```

---

### 🎯 Interview Focus

* Understand implicit dependencies
* No need for `depends_on`

---

# 🔹 Task 7: Handle Drift (Import Existing Resource)

### 🧠 Problem

An EC2 instance already exists. Bring it under Terraform.

---

### 💻 Solution

```bash id="import01"
terraform import aws_instance.web i-123456
```

---

### 🎯 Insight

* Real-world scenario
* Must explain:

  * Import doesn’t create config
  * You must write matching config manually

---

# 🔹 Task 8: Prevent Accidental Deletion

### 🧠 Problem

Protect critical resource from destruction.

---

### 💻 Solution

```hcl id="lifecycle01"
resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t2.micro"

  lifecycle {
    prevent_destroy = true
  }
}
```

---

### 🎯 Interview Insight

* Production safety awareness

---

# 🔹 Task 9: Output Values for Integration

### 🧠 Problem

Expose EC2 public IP for downstream usage.

---

### 💻 Solution

```hcl id="output01"
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

### 🎯 Use Case

* CI/CD pipelines
* Monitoring setup

---

# 🔹 Task 10: Debug a Failing Terraform Run

### 🧠 Problem

`terraform apply` fails due to dependency or provider issue.

---

### ✅ Approach (Expected Answer)

1. Run `terraform plan`
2. Validate config → `terraform validate`
3. Enable debug logs:

```bash id="debug01"
export TF_LOG=DEBUG
```

4. Check:

   * Credentials
   * State file
   * Resource conflicts

---

### 🎯 Interview Insight

* Structured debugging approach matters more than commands

---

# 🔥 Bonus: Design-Level Task (Very Important)

### 🧠 Problem

Design Terraform structure for:

* Dev / Staging / Prod environments
* Reusable infra

---

### ✅ Strong Answer

* Use:

  * Modules (VPC, EC2, DB)
  * Separate folders or workspaces
  * Remote backend
  * CI/CD pipeline

---

### Example Structure

```
terraform/
  modules/
    vpc/
    ec2/
  envs/
    dev/
    prod/
```

---

# 🚨 What Differentiates Candidates

### ❌ Weak Answer

* Only writes code
* No explanation

### ✅ Mid-Level Answer

* Explains:

  * Why using module
  * Why remote state
  * Trade-offs

### 🔥 Strong Candidate

* Mentions:

  * Drift handling
  * Security
  * CI/CD integration
  * Scaling issues

---


