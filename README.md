provider "aws" {
  region     = "us-west-1"
  access_key = "AKIAQWMWLQWD77UF2VXB"
  secret_key = "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"
}

resource "aws_instance" "terraformtest2" {
  ami           = "ami-0"
  instance_type = "t2.micro" 

  tags = {
    Name = "terraformtest2"
  }
}

resource "aws_security_group" "example_security_group" {
  name_prefix = "example-security-group"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  
    }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"  # Allow all outbound traffic
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group_rule" "egress_rule" {
  type        = "egress"
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.example_security_group.id
}
