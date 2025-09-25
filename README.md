# CircleCI + AWS + OPA Integration Lab

**Lab Objective**: Set up a complete CI/CD pipeline with CircleCI, AWS integration using OIDC, and OPA policy validation that demonstrates policy violations.

**Time Required**: 2-3 hours

**Prerequisites**: 
- GitHub account
- AWS account with administrative access
- Basic familiarity with terminal/command line
- Text editor (VS Code, nano, etc.)
- - **Note:** _If nano doesn't work for you via the terminal when creating/editing files (config files, policy files, etc.), you can always copy and paste the code in the lab step directly into that file, [once created](https://docs.github.com/en/repositories/working-with-files/managing-files/creating-new-files), in github and commit the changes._
  - **Note:** _If you get stuck at anytime in this lab, or something isn't working as it should, ChatGPT or Claude is your best friend :-)_ 

---
## 0) Bash navigation cheat sheet (use whenever you’re unsure)

```bash
pwd                     # show current directory
ls -la                  # list files and show hidden .git/
cd ~/code               # go to 'code' directory in your home
cd <folder>             # change into a folder
cd ..                   # up one level
cd -                    # back to previous directory
mkdir -p path/to/dir    # create a directory (nested ok)
```

## Lab Setup: Create Your Working Directory

Before starting, create a proper working environment:

```bash
# Navigate to your preferred development location
cd ~/Documents  # or wherever you keep projects

# Create and enter your lab directory
mkdir circleci-aws-opa-lab
cd circleci-aws-opa-lab

# Initialize git repository
git init

# Create initial project structure
mkdir -p policies/security tests/ terraform/
touch README.md .gitignore

# Your working directory structure should look like:
# circleci-aws-opa-lab/
# ├── .git/                    # Hidden git directory (created by git init)
# ├── policies/
# │   └── security/
# ├── tests/
# ├── terraform/
# ├── README.md
# └── .gitignore
```

If you haven't already, I'd recommend setting up SSH first (recommended so you’re not prompted to authenticate with every push):
**Very Important**:  If you are prompted to create a passphrase, you need to store it somewhere like a password manager.  **You may need it again!  Even if you store it in your apple keychain**

```bash
# Generate an SSH key (press Enter through prompts unless you want a custom path)
ssh-keygen -t ed25519 -C "<you>@email.com"

# Start the agent and add the key
# macOS:
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# Linux:
# eval "$(ssh-agent -s)"
# ssh-add ~/.ssh/id_ed25519

# Add the public key to GitHub → Settings → SSH and GPG keys → New SSH key
cat ~/.ssh/id_ed25519.pub

# Test connectivity
ssh -T git@github.com
```

> Tip: On macOS the `--apple-use-keychain` flag stores your passphrase so pushes don’t reprompt you.

---
**Important**: All subsequent terminal commands in this lab assume you are in the `circleci-aws-opa-lab` directory unless otherwise specified.

---

## Step 1: Set Up GitHub Repository

### 1.1 Create Repository on GitHub

1. Go to [github.com](https://github.com)
2. Click "New" repository
3. **Repository name**: `circleci-aws-opa-lab`
4. Make it **Public** (required for free CircleCI)
5. **Do NOT** initialize with README (you already created one locally)
6. Click "Create repository"

### 1.2 Connect Local Repository to GitHub

**Make sure you are in your lab directory**:
```bash
# Verify you're in the right place
pwd
# Should show: /Users/YOUR_USERNAME/Documents/circleci-aws-opa-lab (or your chosen path)

# Add content to README, (Note: the "echo" command creates the readme file and adds the text between the quotes to it)
echo "# CircleCI AWS OPA Integration Lab" > README.md

# Add basic gitignore, (Note: the "cat" command creates the .gitignore file)
cat > .gitignore << EOF
**Note:** In the terminal you'll see the words "heredoc"
on the command line after you enter this cat command,
just copy and paste the below text up to, and including,
the word "EOF" into the terminal and press enter.
This is the text you need in the .gitignore file this command just created

# AWS credentials (never commit these)
aws-configs/
*.pem
*.key

# Terraform
*.tfstate
*.tfstate.*
.terraform/

# OS files
.DS_Store
EOF

# Stage and commit files
git add .
git commit -m "Initial commit: Lab setup"

# Connect to GitHub (replace YOUR_USERNAME with your actual GitHub username)
git remote add origin git@github.com:YOUR_USERNAME/circleci-aws-opa-lab.git
git branch -M main
git push -u origin main
```

---

## Step 2: Set Up CircleCI Account and Connect Repository

### 2.1 Create CircleCI Account

1. Go to [circleci.com](https://circleci.com)
2. Click "Sign Up" and choose "Sign up with GitHub"
3. Authorize CircleCI to access your GitHub account

### 2.2 Connect Your Repository

1. In CircleCI dashboard, click "Projects"
2. Click on "Create Project" in the top right of the window
3. Choose "Build, test, and deploy your software application"
4. Name the project "circleci-aws-opa-lab"
5. Click "setup a pipeline"
6. It's ok to go with the default pipeline name "build-and-test"
7. Click "Next: choose a repo"
8. Find `circleci-aws-opa-lab` in the list
9. Click "Next: set up your config"
10. Click "Prepare config file" 
11. CircleCI will create a basic `.circleci/config.yml` file
12. Click "Next: set up your triggers"
13. "All pushes" is fine
14. Click "Next: review and finish setup"
15. Click "Commit config and run"

### 2.3 Create CircleCI Configuration

**In your local lab directory**, create the CircleCI configuration:

```bash
# Make sure you're in the lab directory
cd ~/Documents/circleci-aws-opa-lab  # Adjust path as needed

# Create CircleCI directory (the . makes it hidden)
mkdir -p .circleci

# Create basic configuration file
cat > .circleci/config.yml << 'EOF'
version: 2.1

jobs:
  test:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Hello World
          command: echo "Hello from CircleCI!"

workflows:
  test-workflow:
    jobs:
      - test
EOF

# Commit and push the configuration
git add .circleci/config.yml
git commit -m "Add basic CircleCI configuration"
git push origin main
```

**Important Notes**:
- The `.circleci` directory starts with a dot, making it a hidden directory
- You must be in your Git repository directory (with the `.git` folder) to run `git` commands
- Check that you're in the right directory with `ls -la` - you should see `.git` listed

### 2.4 Verify CircleCI is Working

1. Go to CircleCI dashboard
2. Click on "Pipelines"
3. You should see your pipeline running with a green "Success"
4. Click on "test", then "Hello World" to see "Hello from CircleCI!" in the logs

---

## Step 3: Set Up AWS Integration with OIDC

### 3.1 Find Required IDs

You need three pieces of information. Let's gather them:

#### AWS Account ID
**Method 1 - AWS Console**:
1. Log into AWS Console
2. Click your username in top-right corner
3. Your 12-digit Account ID is shown in the dropdown

**Method 2 - AWS CLI** (if installed):
```bash
aws sts get-caller-identity
# Look for "Account": "123456789012"
```

#### CircleCI Organization ID
1. Go to CircleCI → Organization Settings
2. Click "Overview" in left sidebar
3. Copy the "Organization ID" (format: `a1b2c3d4-e5f6-7890-abcd-ef1234567890`)

#### CircleCI Project ID
1. Go to your CircleCI project
2. Click "Project Settings" (gear icon)
3. Click "Overview" in left sidebar
4. Copy the "Project ID" (format: `b2c3d4e5-f6g7-8901-bcde-f23456789012`)

**Write these down - you'll need them multiple times**:
```
AWS Account ID: ________________
CircleCI Org ID: ________________
CircleCI Project ID: ________________
```

### 3.2 Create AWS IAM Policy

**In your lab directory**, create the policy file:

```bash
# Create directory for AWS configuration files
mkdir -p aws-configs

# Create IAM policy (save in aws-configs directory)
cat > aws-configs/circleci-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "iam:GetRole",
        "iam:PassRole",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Your project structure now looks like (remember you can always open finder and confirm the poject structure):
# circleci-aws-opa-lab/
# ├── .circleci/
# │   └── config.yml
# ├── aws-configs/
# │   └── circleci-policy.json    # AWS policy stored here
# ├── policies/
# ├── tests/
# └── terraform/
```

**Create the policy in AWS**:
```bash
# Make sure you're in the lab directory
pwd  # Should show your lab directory path

# Create the policy in AWS
aws iam create-policy \
  --policy-name CircleCILabPolicy \
  --policy-document file://aws-configs/circleci-policy.json

# Note the PolicyArn in the output - you'll need it
```

### 3.3 Set Up OIDC Identity Provider

**In AWS Console**:
1. Go to **IAM** → **Identity providers** → **Add provider**
2. **Provider type**: OpenID Connect
3. **Provider URL**: `https://oidc.circleci.com/org/YOUR_CIRCLECI_ORG_ID`
   - Replace `YOUR_CIRCLECI_ORG_ID` with your actual Org ID from Step 3.1
4. **Audience**: `YOUR_CIRCLECI_ORG_ID` (same value)
5. Click **Add provider**

### 3.4 Create IAM Role with Trust Policy

**In your lab directory**, create the trust policy:

```bash
# Create trust policy file (replace the placeholders with your actual values)
cat > aws-configs/circleci-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.circleci.com/org/YOUR_CIRCLECI_ORG_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.circleci.com/org/YOUR_CIRCLECI_ORG_ID:aud": "YOUR_CIRCLECI_ORG_ID"
        },
        "StringLike": {
          "oidc.circleci.com/org/YOUR_CIRCLECI_ORG_ID:sub": "org/YOUR_CIRCLECI_ORG_ID/project/YOUR_CIRCLECI_PROJECT_ID/user/*"
        }
      }
    }
  ]
}
EOF

# IMPORTANT: Edit the file to replace placeholders with your actual values


# Replace ALL instances of:
# YOUR_AWS_ACCOUNT_ID with your 12-digit AWS account ID
# YOUR_CIRCLECI_ORG_ID with your CircleCI organization ID  
# YOUR_CIRCLECI_PROJECT_ID with your CircleCI project ID
```

**Create the IAM role**:
```bash
# Create the role
aws iam create-role \
  --role-name CircleCILabRole \
  --assume-role-policy-document file://aws-configs/circleci-trust-policy.json

# Attach the policy to the role (replace YOUR_AWS_ACCOUNT_ID)
aws iam attach-role-policy \
  --role-name CircleCILabRole \
  --policy-arn arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/CircleCILabPolicy
```

### 3.5 Configure Environment Variables in CircleCI

**About Environment Variables**: These are key-value pairs that your CI/CD pipeline can read. They're like settings that tell your code how to behave without hardcoding values.

1. Go to CircleCI → Your Project → Project Settings
2. Click "Environment Variables" in left sidebar
3. Click "Add Environment Variable"

**Add this variable**:
- **Name**: `AWS_DEFAULT_REGION`
- **Value**: `us-east-1` (or your preferred AWS region)

**Environment Variable Rules**:
- Names must start with letter or underscore (`_`)
- No spaces or special characters in names
- Values can contain most characters
- If you need a `$` in the value, escape it as `\$`

---

## Step 4: Update CircleCI Configuration for AWS Integration

### 4.1 Update CircleCI Configuration

**In your lab directory**, replace the CircleCI config:

```bash
# Update the CircleCI configuration
cat > .circleci/config.yml << 'EOF'
version: 2.1

jobs:
  test-aws-connection:
    docker:
      - image: cimg/base:stable
    environment:
      AWS_DEFAULT_REGION: us-east-1
    steps:
      - checkout
      
      - run:
          name: Install AWS CLI
          command: |
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip
            sudo ./aws/install
      
      - run:
          name: Configure OIDC and Test AWS Connection
          command: |
            # Set up OIDC authentication
            export AWS_ROLE_ARN="arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/CircleCILabRole"
            export AWS_WEB_IDENTITY_TOKEN_FILE="/tmp/web-identity-token"
            echo $CIRCLE_OIDC_TOKEN > $AWS_WEB_IDENTITY_TOKEN_FILE
            
            # Test AWS connection
            echo "Testing AWS connection..."
            aws sts get-caller-identity
            echo "AWS connection successful!"

workflows:
  test:
    jobs:
      - test-aws-connection
EOF

# Replace YOUR_AWS_ACCOUNT_ID with your actual 12-digit AWS account ID
```

### 4.2 Test AWS Integration

```bash
# Commit and push the updated configuration
git add .circleci/config.yml
git commit -m "Add AWS OIDC integration"
git push origin main
```

**Verify in CircleCI**:
1. Go to CircleCI dashboard
2. Watch the pipeline run
3. You should see a green "Successful!" in the pipeline for this job and a green check mark next to "test AWS connection"

---

## Step 5: Set Up OPA (Open Policy Agent)

### 5.1 Create OPA Policies

**In your lab directory**, create security policies:

```bash
# Create a basic security policy for S3
cat > policies/security/s3.rego << 'EOF'
package aws.s3.security

# Deny S3 buckets without encryption
deny[msg] {
    input.resource_type == "aws_s3_bucket"
    not input.server_side_encryption_configuration
    msg := "S3 buckets must have server-side encryption enabled"
}

# Deny S3 buckets that allow public read access
deny[msg] {
    input.resource_type == "aws_s3_bucket"
    input.acl == "public-read"
    msg := "S3 buckets must not have public-read ACL"
}

# Warn about S3 buckets without versioning
warn[msg] {
    input.resource_type == "aws_s3_bucket"
    not input.versioning
    msg := "Consider enabling versioning for S3 buckets"
}
EOF

# Create policy tests
cat > tests/s3_test.rego << 'EOF'
package aws.s3.security

# Test that encrypted S3 bucket is allowed
test_encrypted_bucket_allowed {
    count(deny) == 0 with input as {
        "resource_type": "aws_s3_bucket",
        "server_side_encryption_configuration": {
            "rule": {
                "apply_server_side_encryption_by_default": {
                    "sse_algorithm": "AES256"
                }
            }
        }
    }
}

# Test that unencrypted S3 bucket is denied
test_unencrypted_bucket_denied {
    count(deny) > 0 with input as {
        "resource_type": "aws_s3_bucket"
    }
}

# Test that public-read bucket is denied
test_public_read_bucket_denied {
    count(deny) > 0 with input as {
        "resource_type": "aws_s3_bucket",
        "acl": "public-read",
        "server_side_encryption_configuration": {}
    }
}
EOF
```

### 5.2 Update CircleCI Configuration to Include OPA

```bash
# Update CircleCI config to include OPA validation
cat > .circleci/config.yml << 'EOF'
version: 2.1

jobs:
  policy-validation:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      
      - run:
          name: Install Dependencies
          command: |
            # Install OPA
            curl -L -o opa https://openpolicyagent.org/downloads/v0.58.0/opa_linux_amd64_static
            chmod +x opa
            sudo mv opa /usr/local/bin/
            
            # Install jq for JSON processing
            sudo apt-get update
            sudo apt-get install -y jq
      
      - run:
          name: Test OPA Policies
          command: |
            echo "Running OPA policy tests..."
            if [ -d "tests/" ]; then
              opa test policies/ tests/
            else
              echo "No tests directory found, skipping policy tests"
            fi
            echo "Policy tests completed successfully!"
      
      - run:
          name: Validate Sample Resource
          command: |
            # Create a sample S3 bucket configuration (compliant)
            cat > compliant-s3.json \<< 'JSON'
            {
              "resource_type": "aws_s3_bucket",
              "bucket": "compliant-bucket",
              "server_side_encryption_configuration": {
                "rule": {
                  "apply_server_side_encryption_by_default": {
                    "sse_algorithm": "AES256"
                  }
                }
              },
              "versioning": {
                "enabled": true
              }
            }
            JSON
            
            # Test compliant resource
            echo "Testing compliant S3 bucket..."
            RESULT=$(opa eval -d policies/ -i compliant-s3.json "data.aws.s3.security.deny[x]" --format json)
            echo "Policy evaluation result: $RESULT"
            
            if echo "$RESULT" | jq -e '.result | length == 0' > /dev/null; then
              echo "✅ Compliant resource passed validation"
            else
              echo "❌ Compliant resource failed validation"
              echo "Violations found: $RESULT"
              exit 1
            fi
            
            # Create a non-compliant S3 bucket configuration
            cat > non-compliant-s3.json \<< 'JSON'
            {
              "resource_type": "aws_s3_bucket",
              "bucket": "non-compliant-bucket"
            }
            JSON
            
            # Test non-compliant resource
            echo "Testing non-compliant S3 bucket..."
            VIOLATIONS=$(opa eval -d policies/ -i non-compliant-s3.json "data.aws.s3.security.deny[x]" --format json)
            echo "Violations result: $VIOLATIONS"
            
            if echo "$VIOLATIONS" | jq -e '.result | length > 0' > /dev/null; then
              echo "✅ Non-compliant resource correctly flagged:"
              echo "$VIOLATIONS" | jq '.result'
            else
              echo "❌ Non-compliant resource should have been flagged but wasn't"
              exit 1
            fi

  test-aws-connection:
    docker:
      - image: cimg/base:stable
    environment:
      AWS_DEFAULT_REGION: us-east-1
    steps:
      - checkout
      
      - run:
          name: Install AWS CLI
          command: |
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip
            sudo ./aws/install
      
      - run:
          name: Configure OIDC and Test
          command: |
            # Replace YOUR_AWS_ACCOUNT_ID with your actual AWS account ID
            export AWS_ROLE_ARN="arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/CircleCILabRole"
            export AWS_WEB_IDENTITY_TOKEN_FILE="/tmp/web-identity-token"
            echo $CIRCLE_OIDC_TOKEN > $AWS_WEB_IDENTITY_TOKEN_FILE
            
            echo "Testing AWS connection..."
            aws sts get-caller-identity
            echo "AWS connection successful!"

workflows:
  security-pipeline:
    jobs:
      - policy-validation
      - test-aws-connection:
          requires:
            - policy-validation
EOF


# Replace YOUR_AWS_ACCOUNT_ID with your actual AWS account ID
```

### 5.3 Test OPA Integration

```bash
# Commit and push OPA configuration
git add .
git commit -m "Add OPA policy validation"
git push origin main
```

**Verify in CircleCI**:
1. Watch the pipeline run
2. The "policy-validation" job should run first
3. You should see policy tests pass and validation results

---

## Step 6: Create Infrastructure that Violates Policy

### 6.1 Create Terraform Configuration

**In your lab directory**, create infrastructure code:

```bash
# Create a Terraform configuration that violates our S3 policy
cat > terraform/main.tf << 'EOF'
provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

# This S3 bucket VIOLATES our policy (no encryption, public-read ACL)
resource "aws_s3_bucket" "policy_violation_bucket" {
  bucket = "circleci-lab-violation-bucket-${random_string.suffix.result}"
}

resource "aws_s3_bucket_acl" "policy_violation_bucket_acl" {
  bucket = aws_s3_bucket.policy_violation_bucket.id
  acl    = "public-read"  # This violates our policy
}

# Random suffix to ensure unique bucket names
resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

output "bucket_name" {
  value = aws_s3_bucket.policy_violation_bucket.bucket
}
EOF
```

### 6.2 Update CircleCI to Validate Terraform and Deploy (Job will fail in CircleCI)
```bash
# Update CircleCI config to validate terraform and deploy
cat > .circleci/config.yml << 'EOF'

version: 2.1

jobs:
  policy-validation:
    docker:
      - image: cimg/python:3.9
    steps:
      - checkout
      
      - run:
          name: Install Dependencies
          command: |
            # Install OPA
            curl -L -o opa https://openpolicyagent.org/downloads/v0.58.0/opa_linux_amd64_static
            chmod +x opa
            sudo mv opa /usr/local/bin/
            
            # Install jq for JSON processing
            sudo apt-get update
            sudo apt-get install -y jq
      
      - run:
          name: Test OPA Policies
          command: |
            echo "Running OPA policy tests..."
            if [ -d "tests/" ]; then
              opa test policies/ tests/
            else
              echo "No tests directory found, skipping policy tests"
            fi
            echo "Policy tests passed!"
      
      - run:
          name: Validate Terraform Configuration
          command: |
            echo "Validating Terraform resources against policies..."
            
            # Create JSON representation of our violating S3 bucket
            cat > terraform-resources.json \<< 'JSON'
            [
              {
                "resource_type": "aws_s3_bucket",
                "resource_name": "policy_violation_bucket",
                "bucket": "circleci-lab-violation-bucket-12345"
              },
              {
                "resource_type": "aws_s3_bucket_acl",
                "resource_name": "policy_violation_bucket_acl",
                "bucket": "circleci-lab-violation-bucket-12345",
                "acl": "public-read"
              }
            ]
            JSON
            
            # Validate each resource and track violations
            TOTAL_VIOLATIONS=0
            FOUND_VIOLATIONS=false
            
            # Process each resource individually
            for i in $(jq -r 'keys | .[]' terraform-resources.json); do
              resource=$(jq -c ".[$i]" terraform-resources.json)
              resource_name=$(echo "$resource" | jq -r '.resource_name')
              
              echo "Validating resource: $resource_name"
              
              # Write resource to temporary file for OPA
              echo "$resource" > /tmp/resource_$i.json
              
              VIOLATIONS=$(opa eval -d policies/ -i /tmp/resource_$i.json "data.aws.s3.security.deny[x]" --format json)
              WARNINGS=$(opa eval -d policies/ -i /tmp/resource_$i.json "data.aws.s3.security.warn[x]" --format json)
              
              # Check if violations exist
              if echo "$VIOLATIONS" | jq -e '.result | length > 0' > /dev/null 2>&1; then
                echo "❌ POLICY VIOLATIONS FOUND:"
                echo "$VIOLATIONS" | jq '.result'
                echo "VIOLATIONS_FOUND=true" >> /tmp/violations_status
              elif [ "$VIOLATIONS" != "" ]; then
                echo "✅ No violations found for this resource"
              fi
              
              # Check if warnings exist
              if echo "$WARNINGS" | jq -e '.result | length > 0' > /dev/null 2>&1; then
                echo "⚠️  POLICY WARNINGS:"
                echo "$WARNINGS" | jq '.result'
              fi
              
              # Clean up temp file
              rm -f /tmp/resource_$i.json
            done
            
            # Check if any violations were found
            if [ -f /tmp/violations_status ] && grep -q "VIOLATIONS_FOUND=true" /tmp/violations_status; then
              echo "Found policy violations in Terraform configuration"
              echo "❌ Build failed due to policy violations - this is expected!"
              exit 1
            else
              echo "✅ No policy violations found"
            fi

  deploy-compliant-infrastructure:
    docker:
      - image: cimg/base:stable
    environment:
      AWS_DEFAULT_REGION: us-east-1
    steps:
      - checkout
      
      - run:
          name: Install AWS CLI and Terraform
          command: |
            # Install AWS CLI
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip
            sudo ./aws/install
            
            # Install Terraform
            wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
            unzip terraform_1.6.0_linux_amd64.zip
            sudo mv terraform /usr/local/bin/
      
      - run:
          name: Create and Deploy Compliant Infrastructure
          command: |
            export AWS_ROLE_ARN="arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/CircleCILabRole"
            export AWS_WEB_IDENTITY_TOKEN_FILE="/tmp/web-identity-token"
            echo $CIRCLE_OIDC_TOKEN > $AWS_WEB_IDENTITY_TOKEN_FILE
            
            # Create terraform directory if it doesn't exist
            mkdir -p terraform
            cd terraform/
            
            # Create compliant version
            cat > main-compliant.tf \<< 'TERRAFORM'
            provider "aws" {
              region = var.aws_region
            }
            
            variable "aws_region" {
              description = "AWS region"
              type        = string
              default     = "us-east-1"
            }
            
            resource "aws_s3_bucket" "compliant_bucket" {
              bucket = "circleci-lab-compliant-${random_string.suffix.result}"
            }
            
            resource "aws_s3_bucket_server_side_encryption_configuration" "compliant_encryption" {
              bucket = aws_s3_bucket.compliant_bucket.id
            
              rule {
                apply_server_side_encryption_by_default {
                  sse_algorithm = "AES256"
                }
              }
            }
            
            resource "aws_s3_bucket_public_access_block" "compliant_pab" {
              bucket = aws_s3_bucket.compliant_bucket.id
              block_public_acls       = true
              block_public_policy     = true
              ignore_public_acls      = true
              restrict_public_buckets = true
            }
            
            resource "aws_s3_bucket_versioning" "compliant_versioning" {
              bucket = aws_s3_bucket.compliant_bucket.id
              versioning_configuration {
                status = "Enabled"
              }
            }
            
            resource "random_string" "suffix" {
              length  = 8
              special = false
              upper   = false
            }
            
            output "bucket_name" {
              value = aws_s3_bucket.compliant_bucket.bucket
            }
            TERRAFORM
            
            echo "Initializing Terraform..."
            terraform init
            
            echo "Planning compliant deployment..."
            terraform plan -out=tfplan
            
            echo "Deploying compliant infrastructure..."
            terraform apply -auto-approve tfplan
            
            echo "✅ Compliant infrastructure deployed successfully!"
            
            # Clean up to avoid AWS charges
            echo "Cleaning up resources..."
            terraform destroy -auto-approve
            echo "✅ Resources cleaned up"

workflows:
  security-pipeline:
    jobs:
      - policy-validation
      - deploy-compliant-infrastructure:
          requires:
            - policy-validation
EOF


# Replace YOUR_AWS_ACCOUNT_ID with your actual AWS account ID

# Commit and push the non-compliant version
git add .
git commit -m "Add non-compliant infrastructure that fails policy validation"
git push origin main

**Note**: This build will fail, that's intentional:
-The S3 security policy correctly detected that the bucket lacks server-side encryption
-The policy provided a clear, actionable error message
-The pipeline failed fast when policy violations were detected
```

### 7.3 Verify Complete Success
To experience a job that passes compliance checks you first need to create the "compliant-deployment branch
```bash
# Create and switch to the new branch
git checkout -b compliant-deployment

# Verify you're on the new branch
git branch
```

### 7.4 Edit the circleci config file to include the compliant infrastructure
```bash
# Update CircleCI config to validate terraform and deploy
cat > .circleci/config.yml << 'EOF'

version: 2.1

jobs:
  policy-validation:
    docker:
      - image: cimg/python:3.9
    steps:
      - checkout
      
      - run:
          name: Install Dependencies
          command: |
            # Install OPA
            curl -L -o opa https://openpolicyagent.org/downloads/v0.58.0/opa_linux_amd64_static
            chmod +x opa
            sudo mv opa /usr/local/bin/
            
            # Install jq for JSON processing
            sudo apt-get update
            sudo apt-get install -y jq
      
      - run:
          name: Test OPA Policies
          command: |
            echo "Running OPA policy tests..."
            if [ -d "tests/" ]; then
              opa test policies/ tests/
            else
              echo "No tests directory found, skipping policy tests"
            fi
            echo "Policy tests passed!"
      
      - run:
          name: Validate Terraform Configuration
          command: |
            echo "Validating COMPLIANT Terraform resources against policies..."
            
            # Create JSON representation of compliant S3 bucket
            cat > terraform-resources.json \<< 'JSON'
            [
              {
                "resource_type": "aws_s3_bucket",
                "resource_name": "compliant_bucket",
                "bucket": "circleci-lab-compliant-bucket-12345",
                "server_side_encryption_configuration": {
                  "rule": {
                    "apply_server_side_encryption_by_default": {
                      "sse_algorithm": "AES256"
                    }
                  }
                },
                "versioning": {
                  "enabled": true
                },
                "tags": {
                  "Environment": "prod",
                  "Owner": "security-team@company.com",
                  "CostCenter": "CC-1234",
                  "Project": "SecurityCompliance",
                  "DataClassification": "internal"
                }
              },
              {
                "resource_type": "aws_s3_bucket_public_access_block",
                "resource_name": "compliant_bucket_pab",
                "bucket": "circleci-lab-compliant-bucket-12345",
                "block_public_acls": true,
                "block_public_policy": true,
                "ignore_public_acls": true,
                "restrict_public_buckets": true
              }
            ]
            JSON
            
            # Validate each resource and track violations
            TOTAL_VIOLATIONS=0
            FOUND_VIOLATIONS=false
            
            # Process each resource individually
            for i in $(jq -r 'keys | .[]' terraform-resources.json); do
              resource=$(jq -c ".[$i]" terraform-resources.json)
              resource_name=$(echo "$resource" | jq -r '.resource_name')
              
              echo "Validating resource: $resource_name"
              
              # Write resource to temporary file for OPA
              echo "$resource" > /tmp/resource_$i.json
              
              VIOLATIONS=$(opa eval -d policies/ -i /tmp/resource_$i.json "data.aws.s3.security.deny[x]" --format json)
              WARNINGS=$(opa eval -d policies/ -i /tmp/resource_$i.json "data.aws.s3.security.warn[x]" --format json)
              TAG_VIOLATIONS=$(opa eval -d policies/ -i /tmp/resource_$i.json "data.compliance.tagging.deny[x]" --format json)
              
              # Check if violations exist
              if echo "$VIOLATIONS" | jq -e '.result | length > 0' > /dev/null 2>&1; then
                echo "❌ S3 SECURITY POLICY VIOLATIONS FOUND:"
                echo "$VIOLATIONS" | jq '.result'
                echo "VIOLATIONS_FOUND=true" >> /tmp/violations_status
              fi
              
              # Check tagging violations
              if echo "$TAG_VIOLATIONS" | jq -e '.result | length > 0' > /dev/null 2>&1; then
                echo "❌ TAGGING POLICY VIOLATIONS FOUND:"
                echo "$TAG_VIOLATIONS" | jq '.result'
                echo "VIOLATIONS_FOUND=true" >> /tmp/violations_status
              fi
              
              # Check if warnings exist
              if echo "$WARNINGS" | jq -e '.result | length > 0' > /dev/null 2>&1; then
                echo "⚠️  POLICY WARNINGS:"
                echo "$WARNINGS" | jq '.result'
              fi
              
              if [ ! -f /tmp/violations_status ]; then
                echo "✅ No violations found for resource: $resource_name"
              fi
              
              # Clean up temp file
              rm -f /tmp/resource_$i.json
            done
            
            # Check if any violations were found
            if [ -f /tmp/violations_status ] && grep -q "VIOLATIONS_FOUND=true" /tmp/violations_status; then
              echo "Found policy violations in Terraform configuration"
              echo "❌ Build failed due to policy violations"
              exit 1
            else
              echo "✅ All resources passed policy validation!"
              echo "✅ Ready for deployment"
            fi

  deploy-compliant-infrastructure:
    docker:
      - image: cimg/base:stable
    environment:
      AWS_DEFAULT_REGION: us-east-1
    steps:
      - checkout
      
      - run:
          name: Install AWS CLI and Terraform
          command: |
            # Install AWS CLI
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip -o awscliv2.zip
            sudo ./aws/install
            
            # Install Terraform (clean install)
            rm -rf terraform*  # Remove any existing terraform files or directories
            wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
            unzip -o terraform_1.6.0_linux_amd64.zip
            chmod +x terraform
            sudo mv terraform /usr/local/bin/
            
            # Verify installations
            terraform version
            aws --version
      
      - run:
          name: Create and Deploy Compliant Infrastructure
          command: |
            export AWS_ROLE_ARN="arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/CircleCILabRole"
            export AWS_WEB_IDENTITY_TOKEN_FILE="/tmp/web-identity-token"
            echo $CIRCLE_OIDC_TOKEN > $AWS_WEB_IDENTITY_TOKEN_FILE
            
            # Create terraform directory if it doesn't exist
            mkdir -p terraform
            cd terraform/
            
            # Create compliant version with all required security controls
            cat > main-compliant.tf \<< 'TERRAFORM'
            provider "aws" {
              region = var.aws_region
            }
            
            variable "aws_region" {
              description = "AWS region"
              type        = string
              default     = "us-east-1"
            }
            
            # Compliant S3 bucket with all security controls
            resource "aws_s3_bucket" "compliant_bucket" {
              bucket = "circleci-lab-compliant-${random_string.suffix.result}"
              
              tags = {
                Environment        = "prod"
                Owner             = "security-team@company.com"
                CostCenter        = "CC-1234"
                Project           = "SecurityCompliance"
                DataClassification = "internal"
              }
            }
            
            # Server-side encryption (required by policy)
            resource "aws_s3_bucket_server_side_encryption_configuration" "compliant_encryption" {
              bucket = aws_s3_bucket.compliant_bucket.id
            
              rule {
                apply_server_side_encryption_by_default {
                  sse_algorithm = "AES256"
                }
              }
            }
            
            # Block all public access (required by policy)
            resource "aws_s3_bucket_public_access_block" "compliant_pab" {
              bucket = aws_s3_bucket.compliant_bucket.id
              
              block_public_acls       = true
              block_public_policy     = true
              ignore_public_acls      = true
              restrict_public_buckets = true
            }
            
            # Versioning enabled (recommended by policy)
            resource "aws_s3_bucket_versioning" "compliant_versioning" {
              bucket = aws_s3_bucket.compliant_bucket.id
              versioning_configuration {
                status = "Enabled"
              }
            }
            
            # Lifecycle configuration for cost optimization
            resource "aws_s3_bucket_lifecycle_configuration" "compliant_lifecycle" {
              bucket = aws_s3_bucket.compliant_bucket.id
              
              rule {
                id     = "transition_to_ia"
                status = "Enabled"
                
                transition {
                  days          = 30
                  storage_class = "STANDARD_IA"
                }
                
                transition {
                  days          = 90
                  storage_class = "GLACIER"
                }
              }
            }
            
            resource "random_string" "suffix" {
              length  = 8
              special = false
              upper   = false
            }
            
            output "bucket_name" {
              value = aws_s3_bucket.compliant_bucket.bucket
            }
            
            output "bucket_arn" {
              value = aws_s3_bucket.compliant_bucket.arn
            }
            
            output "compliance_status" {
              value = "COMPLIANT - All NIST 800-53 controls implemented"
            }
            TERRAFORM
            
            echo "Initializing Terraform..."
            terraform init
            
            echo "Planning compliant deployment..."
            terraform plan -out=tfplan
            
            echo "Deploying compliant infrastructure..."
            terraform apply -auto-approve tfplan
            
            echo "✅ Compliant infrastructure deployed successfully!"
            echo "✅ All NIST 800-53 security controls have been implemented:"
            echo "  - SC-28: Data at rest encryption enabled"
            echo "  - AC-3: Public access blocked"
            echo "  - CP-9: Versioning enabled for data protection"
            echo "  - CM-8: Proper resource tagging for inventory"
            
            # Show the outputs
            terraform output
            
            # Clean up to avoid AWS charges
            sleep 30  # Give a moment to verify deployment
            echo "Cleaning up resources..."
            terraform destroy -auto-approve
            echo "✅ Resources cleaned up successfully"

workflows:
  security-pipeline:
    jobs:
      - policy-validation
      - deploy-compliant-infrastructure:
          requires:
            - policy-validation
EOF

# Replace YOUR_AWS_ACCOUNT_ID with your actual AWS account ID

# Commit and push the non-compliant version
git add .
git commit -m "Add compliant infrastructure that passes policy validation"
git push -u origin compliant-deployment
```
Watch the pipeline run on the compliant-deployment branch - it should now:
1. Pass all policy tests
2. Pass policy validation
3. Deploy compliant infrastructure successfully
4. Clean up resources automatically

---

## Step 8: Lab Verification and Cleanup

### 8.1 Verify Lab Success

**You should have seen:**
1. **Policy Enforcement**: Main branch fails due to policy violations
2. **Policy Tests Pass**: OPA unit tests work correctly
3. **Compliant Deployment**: Compliant branch deploys successfully
4. **Automatic Cleanup**: Resources are destroyed to avoid charges
5. **Security Integration**: OIDC authentication works without long-term keys

### 8.2 Review What You've Accomplished

**Your final project structure:**
```
circleci-aws-opa-lab/
├── .git/                           # Git repository
├── .circleci/
│   └── config.yml                  # CircleCI pipeline configuration
├── aws-configs/
│   ├── circleci-policy.json        # IAM policy for CircleCI
│   └── circleci-trust-policy.json  # OIDC trust policy
├── policies/
│   └── security/
│       └── s3.rego                 # OPA security policies
├── tests/
│   └── s3_test.rego               # Policy unit tests
├── terraform/
│   └── main.tf                     # Infrastructure code
├── README.md
└── .gitignore
```

**Key Achievements:**
- ✅ Set up CircleCI with GitHub integration
- ✅ Configured secure AWS access using OIDC (no long-term keys)
- ✅ Implemented policy-as-code with OPA
- ✅ Created automated policy validation in CI/CD
- ✅ Demonstrated policy violation detection and enforcement
- ✅ Deployed compliant infrastructure automatically
- ✅ Implemented automated resource cleanup

### 8.3 Clean Up AWS Resources (Optional)

**Remove IAM resources if no longer needed:**
```bash
# Navigate back to your lab directory
cd ~/Documents/circleci-aws-opa-lab

# Remove role policy attachment (replace YOUR_AWS_ACCOUNT_ID)
aws iam detach-role-policy \
  --role-name CircleCILabRole \
  --policy-arn arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/CircleCILabPolicy

# Delete the role
aws iam delete-role --role-name CircleCILabRole

# Delete the policy (replace YOUR_AWS_ACCOUNT_ID)
aws iam delete-policy --policy-arn arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/CircleCILabPolicy

# Remove OIDC identity provider (replace YOUR_ACCOUNT_ID and YOUR_ORG_ID)
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/oidc.circleci.com/org/YOUR_ORG_ID
```

---

## Common Troubleshooting

### Git Repository Issues
**Problem**: `fatal: not a git repository`
**Solution**: Ensure you're in the directory with the `.git` folder
```bash
pwd  # Check current directory
ls -la  # Look for .git directory
cd ~/Documents/circleci-aws-opa-lab  # Navigate to correct location
```

### Authentication Issues
**Problem**: GitHub authentication failed
**Solution**: Use Personal Access Token instead of password
1. GitHub → Settings → Developer Settings → Personal Access Tokens
2. Generate new token with repo permissions
3. Use token as password when Git prompts

### CircleCI Environment Variables
**Problem**: Confusion about environment variable format
**Solution**: 
- Name: Use underscores, no spaces (`AWS_DEFAULT_REGION`)
- Value: Plain text (`us-east-1`)
- No need for `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` with OIDC

### OIDC Connection Issues  
**Problem**: AWS connection fails
**Solution**: Verify all IDs are correct in trust policy
- AWS Account ID: 12 digits (find in AWS Console dropdown)
- CircleCI Org ID: UUID format (CircleCI Organization Settings → Overview)
- CircleCI Project ID: UUID format (CircleCI Project Settings → Overview)

### OPA Policy Issues
**Problem**: Policies not working as expected
**Solution**: Test policies locally if you have OPA installed
```bash
# Test policy syntax
opa fmt policies/

# Run policy tests
opa test policies/ tests/

# Test with sample data
echo '{"resource_type":"aws_s3_bucket"}' | opa eval -d policies/ -i - "data.aws.s3.security.deny[x]"
```

---

This lab demonstrates a complete DevSecOps pipeline with policy-as-code enforcement.
              description = "AWS region"
              type        = string
              default     = "us-east-1"
            }
            
            # This S3 bucket COMPLIES with our policy
            resource "aws_s3_bucket" "compliant_bucket" {
              bucket = "circleci-lab-compliant-bucket-${random_string.suffix.result}"
            }
            
            # Enable encryption (complies with policy)
            resource "aws_s3_bucket_server_side_encryption_configuration" "compliant_bucket_encryption" {
              bucket = aws_s3_bucket.compliant_bucket.id
            
              rule {
                apply_server_side_encryption_by_default {
                  sse_algorithm = "AES256"
                }
              }
            }
            
            # Block public access (complies with policy)
            resource "aws_s3_bucket_public_access_block" "compliant_bucket_pab" {
              bucket = aws_s3_bucket.compliant_bucket.id
            
              block_public_acls       = true
              block_public_policy     = true
              ignore_public_acls      = true
              restrict_public_buckets = true
            }
            
            resource "random_string" "suffix" {
              length  = 8
              special = false
              upper   = false
            }
            
            output "bucket_name" {
              value = aws_s3_bucket.compliant_bucket.bucket
            }
            TERRAFORM
            
            # Use the compliant configuration
            cp main-compliant.tf main.tf
            
            echo "Initializing Terraform..."
            terraform init
            
            echo "Planning compliant deployment..."
            terraform plan -out=tfplan
            
            echo "Deploying compliant infrastructure..."
            terraform apply -auto-approve tfplan
            
            echo "✅ Compliant infrastructure deployed successfully!"
            
            # Clean up to avoid AWS charges
            echo "Cleaning up resources..."
            terraform destroy -auto-approve
            echo "✅ Resources cleaned up"

workflows:
  security-pipeline:
    jobs:
      - policy-validation
      # Note: deploy job won't run because policy-validation fails
      # This demonstrates policy enforcement
      - deploy-compliant-infrastructure:
          requires:
            - policy-validation
          filters:
            branches:
              only: manual-deploy  # Only runs on manual-deploy branch
EOF

# Edit to replace YOUR_AWS_ACCOUNT_ID
nano .circleci/config.yml
```

---

## Step 7: Test the Complete Pipeline

### 7.1 Test Policy Enforcement

```bash
# Commit the configuration with policy violations
git add .
git commit -m "Add Terraform with intentional policy violations"
git push origin main
```

**Watch the Pipeline Fail**:
1. Go to CircleCI dashboard
2. The pipeline should fail at the "policy-validation" job
3. You should see policy violations detected
4. The deployment job won't run (policy enforcement working!)

### 7.2 Test with Compliant Configuration

```bash
# Create a branch for compliant deployment
git checkout -b compliant-deployment

# Update the workflow to use compliant validation
cat > .circleci/config.yml << 'EOF'
version: 2.1

jobs:
  policy-validation:
    docker:
      - image: cimg/python:3.9
    steps:
      - checkout
      
      - run:
          name: Install Dependencies
          command: |
            curl -L -o opa https://openpolicyagent.org/downloads/v0.58.0/opa_linux_amd64_static
            chmod +x opa
            sudo mv opa /usr/local/bin/
      
      - run:
          name: Test OPA Policies
          command: |
            echo "Running OPA policy tests..."
            opa test policies/ tests/
            echo "✅ Policy tests passed!"
      
      - run:
          name: Validate Compliant Configuration
          command: |
            echo "🔍 Testing policy enforcement with compliant resources..."
            
            # Test compliant S3 bucket
            cat > compliant-test.json \<< 'JSON'
            {
              "resource_type": "aws_s3_bucket",
              "bucket": "test-bucket",
              "server_side_encryption_configuration": {
                "rule": {
                  "apply_server_side_encryption_by_default": {
                    "sse_algorithm": "AES256"
                  }
                }
              }
            }
            JSON
            
            VIOLATIONS=$(opa eval -d policies/ -i compliant-test.json "data.aws.s3.security.deny[x]")
            if [ "$VIOLATIONS" = "[]" ]; then
              echo "✅ Compliant resource passed validation"
            else
              echo "❌ Compliant resource failed validation:"
              echo "$VIOLATIONS" | jq .
              exit 1
            fi
            
            echo "🎉 Policy validation working correctly!"

  deploy-infrastructure:
    docker:
      - image: cimg/base:stable
    environment:
      AWS_DEFAULT_REGION: us-east-1
    steps:
      - checkout
      
      - run:
          name: Install AWS CLI and Terraform
          command: |
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip
            sudo ./aws/install
            
            wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
            unzip terraform_1.6.0_linux_amd64.zip
            sudo mv terraform /usr/local/bin/
      
      - run:
          name: Deploy Compliant Infrastructure
          command: |
            export AWS_ROLE_ARN="arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/CircleCILabRole"
            export AWS_WEB_IDENTITY_TOKEN_FILE="/tmp/web-identity-token"
            echo $CIRCLE_OIDC_TOKEN > $AWS_WEB_IDENTITY_TOKEN_FILE
            
            cd terraform/
            
            # Create compliant configuration
            cat > main.tf \<< 'TERRAFORM'
            provider "aws" {
              region = var.aws_region
            }
            
            variable "aws_region" {
