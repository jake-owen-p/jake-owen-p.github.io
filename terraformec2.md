# Terraform - EC2

Creating the ec2 instance is the simple part

## EC2
```terraform
resource "aws_instance" "web" {
    ami           = data.aws_ami.al2.id
    instance_type = "t2.micro"
    tags = {
        Name = "processor-ec2"
    }
}
```

## AWS Linux2 AMI for EC2
```terraform
data "aws_ami" "al2" {
    most_recent = true
    owners      = ["amazon"]
    filter {
        name   = "owner-alias"
        values = ["amazon"]
    }
    filter {
        name   = "name"
        values = ["amzn2-ami-hvm*"]
    }
}
```

If you want to ssh in to the instance, you must add a security group to allow for tcp connections, along with a key/pair to access it securely.

The below security group allows for an inbound TCP connection on port 22, and allows all traffic outbound. It can then be used in the `aws_instance` setup by referencing the name `security_groups = [aws_security_group.allow_tcp.name]`

## AWS Security Group
```terraform
resource "aws_security_group" "allow_tcp" {
  name        = "allow_tcp"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_tcp"
  }
}
```

## PEM Key
Finally, you'll want to access this securely. You must create a key/pair within amazon first as Terraform does not support this. This is because you must download your private key as a one off.

Once you've done this, you can pass the name to your `aws_instance` setup using `key_name = "video-messaging-processing-kp"`

