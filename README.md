# Terraform with AWS Provider and Ansible Provisioner for Gitlab CI

This Docker images is basen on the official `hashicorp/terraform:0.11.13` Terraform image (light) and extends it with the [Terraform AWS Provider](https://github.com/terraform-providers/terraform-provider-aws/releases) and [Ansible Provisioner by radekg](https://github.com/radekg/terraform-provisioner-ansible).

**It is intended for the use as base image for [GitLab CI pipelines](https://docs.gitlab.com/ce/ci/quick_start/README.html).** You can read my full article about how to use the image on Medium.com: [About Infrastructure on AWS, Automated with Terraform, Ansible and GitLab CI](https://medium.com/@robinflume/about-infrastructure-on-aws-automated-with-terraform-ansible-and-gitlab-ci-5888fe2e85fc).

The image is build as [Docker Multi-Stage Build](https://docs.docker.com/develop/develop-images/multistage-build/), which required Docker Engine `v17.05` or higher.

## Default Versions

The image needs to be build with Docker `build-args`, which default to the following versions:

* Terraform: `latest` (depends on the Terraform version of `hashicorp/terraform:light`)
* AWS Provisioner: `2.6.0`
* Ansible Provisioner: `2.2.0`

You can overwrite the versions of both the AWS Provisioner and the Ansible Provider within the `docker build` command:

```bash
docker build -t terraform-aws-ansible --build-arg AWS_PROVIDER_VERSION=2.1.1 --build-arg ANSIBLE_PROVISIONER_VERSION=2.1.1 .
```

The availailable versions can be found here:

* [AWS Provider Versions](https://github.com/terraform-providers/terraform-provider-aws/releases)
* [Ansible Provisioner Versions](https://github.com/radekg/terraform-provisioner-ansible/releases)

## Required Secrets

The image can be used to automate your Infrastructure creation with Gitlab CI pipelines. It is therefor required to provide both AWS credentials and login data of the user that the Ansible provisioner uses.

These must be provided as Gitlab CI secrets (project environment variables):

* `ANSIBLE_BECOME_PASS`: The password to become root on the hosts to be provisioned by ansible
* `ANSIBLE_VAULT_PASS`: The password the decrypt the [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) (optional)
* `AWS_ACCESS_KEY_ID`: An AWS access key id to be used by the AWS provider
* `AWS_SECRET_ACCESS_KEY`: An AWS secret key to be used by the AWS provider
* `ID_RSA_TERRAFORM`: An SSH private key to be used by Ansible

## The Gitlab CI Pipeline Configuration

Tha actual pipeline can be configures as shown in this example:

```yml
image:
  name: rflume/terraform-aws-ansible:latest

stages:
  # Dev environment stages
  - validate dev
  - plan dev
  - apply dev

variables:
  AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
  AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY

# Create files w/ required the secrets
before_script:
  - echo "$ID_RSA_TERRAFORM" > /root/.ssh/id_rsa_terraform
  - echo "$ANSIBLE_VAULT_PASS" > /etc/ansible/vault_password_file
  - echo "$ANSIBLE_BECOME_PASS" > /etc/ansible/become_pass

# Apply Terraform on DEV environment
validate:dev:
  stage: validate dev
  script:
    - cd environments/dev
    - terraform init
    - terraform validate
  only:
    changes:
      - environments/dev/**/*
      - modules/**/*

plan:dev:
  stage: plan dev
  script:
    - cd environments/dev
    - terraform init
    - terraform plan -out "planfile_dev"
  artifacts:
    paths:
      - environments/dev/planfile_dev
  only:
    changes:
      - environments/dev/**/*
      - modules/**/*

apply:dev:
  stage: apply dev
  script:
    - cd environments/dev
    - terraform init
    - terraform apply -input=false "planfile_dev"
  dependencies:
    - plan:dev
  allow_failure: false
  only:
    refs:
      - master
    changes:
      - environments/dev/**/*
      - modules/**/*
```

### Project Layout

The above pipeline works for the following project layout:

```text
├── ansible-provisioning
│ └── roles
│   └── my-global-role
│     └── ...
│
├── global
│   ├── files
│   │ └── user_data.sh
│   └── {main|outputs|terraform|vars}.tf
│
├── environments
│ ├── dev
│ │ └── {main|outputs|terraform|vars}.tf
│ ├── stage
│ │ └── {main|outputs|terraform|vars}.tf
│ └── prod
│   └── {main|outputs|terraform|vars}.tf
│
├── modules
│ └── my_module
│   ├── ansible
|   │ └── playbook
|   │   └── …
│   └── {main|outputs|terraform|vars}.tf
│
├── .gitlab-ci.yml
└── README.md
```

### Ansible Provisioning

To provision a newly created resource *directly* within Terraform, the Ansible provisioner is included in the image. For more information on available parameters, checkout the [Github project by radekg](https://github.com/radekg/terraform-provisioner-ansible).

This is a brief example on how to use Ansible provisioning with this Docker image:

```t
# ec2 instance
resource "aws_instance" "default" {
   ...
   user_data = "${data.terraform_remote_state.global.user_data}"
   ...
   associate_public_ip_address = true
   key_name = "${data.terraform_remote_state.global.ssh_pubkey}"
   # ignore user_data updates, as this will require a new resource!
   lifecycle {
     ignore_changes = [
       "user_data",
     ]
   }
}

# instance provisioner
resource "null_resource" "default_provisioner" {
  triggers {
    default_instance_id = "${aws_instance.default.id}"
  }

  connection {
    host = "${aws_instance.default.public_ip}"
    type = "ssh"
    user = "terraform"   # as created in 'user_data'
    private_key = "${file("/root/.ssh/id_rsa_terraform")}"
  }
  # wait for the instance to become available
  provisioner "remote-exec" {
    inline = [
      "echo 'ready'"
    ]
  }
  # ansible provisioner
  provisioner "ansible" {
    plays {
      playbook = {
        file_path = "${path.module}/ansible/playbook/main.yml"
        roles_path = [
          "${path.module}/../../../../../ansible-provisioning/roles",
        ]
      }
    hosts = ["${aws_instance.default.public_ip}"]
    become = true
    become_method = "sudo"
    become_user = "root"

    extra_vars = {
      ...
      ansible_become_pass = "${file("/etc/ansible/become_pass")}"
    }

    vault_password_file = "/etc/ansible/vault_password_file"
  }
}
```
