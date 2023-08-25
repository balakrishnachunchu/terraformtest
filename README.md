provider "aws" {
  region     = "us-west-1"
  access_key = "AKIAQWMWLQWD77UF2VXB"
  secret_key = "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"
}

data "aws_availability_zones" "available" {}

locals {
  name   = "ex-${basename(path.cwd)}"
  region = "eu-west-1"

  vpc_cidr = "10.0.0.0/16"
  azs      = slice(data.aws_availability_zones.available.names, 0, 3)

  user_data = <<-EOT
    #!/bin/bash
    echo "Hello Terraform!"
  EOT

  tags = {
    Name       = local.name
    Example    = local.name
    Repository = "https://github.com/terraform-aws-modules/terraform-aws-ec2-instance"
  }
}

################################################################################
# EC2 Module
################################################################################

module "ec2_complete" {
  source = "../../"

  name = local.name

  ami                         = ami-08a52ddb321b32a8c (64-bit (x86)) 
  instance_type               = "c5.xlarge" # used to set core count below
  availability_zone           = element(module.vpc.azs, 0)
  subnet_id                   = element(module.vpc.private_subnets, 0)
  vpc_security_group_ids      = [module.security_group.security_group_id]
  placement_group             = aws_placement_group.web.id
  associate_public_ip_address = true
  disable_api_stop            = false

  create_iam_instance_profile = true
  iam_role_description        = "IAM role for EC2 instance"
  iam_role_policies = {
    AdministratorAccess = "arn:aws:iam::aws:policy/AdministratorAccess"
  }

  # only one of these can be enabled at a time
  hibernation = true
  # enclave_options_enabled = true

  user_data_base64            = base64encode(local.user_data)
  user_data_replace_on_change = true

  cpu_options = {
    core_count       = 2
    threads_per_core = 1
  }
  enable_volume_tags = false
  root_block_device = [
    {
      encrypted   = true
      volume_type = "gp3"
      throughput  = 200
      volume_size = 50
      tags = {
        Name = "my-root-block"
      }
    },
  ]

  ebs_block_device = [
    {
      device_name = "/dev/sdf"
      volume_type = "gp3"
      volume_size = 5
      throughput  = 200
      encrypted   = true
      kms_key_id  = aws_kms_key.this.arn
      tags = {
        MountPoint = "/mnt/data"
      }
    }
  ]

  tags = local.tags
}

module "ec2_network_interface" {
  source = "../../"

  name = "${local.name}-network-interface"

  network_interface = [
    {
      device_index          = 0
      network_interface_id  = aws_network_interface.this.id
      delete_on_termination = false
    }
  ]

  tags = local.tags
}

module "ec2_metadata_options" {
  source = "../../"

  name = "${local.name}-metadata-options"

  subnet_id              = element(module.vpc.private_subnets, 0)
  vpc_security_group_ids = [module.security_group.security_group_id]

  metadata_options = {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 8
    instance_metadata_tags      = "enabled"
  }

  tags = local.tags
}

module "ec2_t2_unlimited" {
  source = "../../"

  name = "${local.name}-t2-unlimited"

  instance_type               = "t2.micro"
  cpu_credits                 = "unlimited"
  subnet_id                   = element(module.vpc.private_subnets, 0)
  vpc_security_group_ids      = [module.security_group.security_group_id]
  associate_public_ip_address = true

  maintenance_options = {
    auto_recovery = "default"
  }

  tags = local.tags
}

module "ec2_t3_unlimited" {
  source = "../../"

  name = "${local.name}-t3-unlimited"

  instance_type               = "t3.micro"
  cpu_credits                 = "unlimited"
  subnet_id                   = element(module.vpc.private_subnets, 0)
  vpc_security_group_ids      = [module.security_group.security_group_id]
  associate_public_ip_address = true

  tags = local.tags
}

module "ec2_disabled" {
  source = "../../"

  create = false
}
