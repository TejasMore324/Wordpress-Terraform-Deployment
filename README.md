# Deploy WordPress Application on AWS Using Terraform

 

- WordPress is a free and open-source Content Management System (CMS) software that is used to create websites, blogs, and online stores. It is one of the most popular CMS tools in the world, with over 40% of all websites on the internet built using WordPress.

- WordPress is written in PHP and uses a MySQL or MariaDB database to store content and settings. 

- It is designed to be user-friendly, with a wide range of themes and plugins available to customize the appearance and functionality of a website
  
- It sets up the necessary AWS infrastructure, including VPC, subnets, route tables, internet and NAT gateways, security groups, RDS for the database, EC2 instances for application servers, and an Application Load Balancer (ALB).

 - It includes the creation of the necessary AWS infrastructure and the deployment of the WordPress application.

### Deployment Architecture 

![Untitled Diagram drawio (26) (1)](https://github.com/Hamid-R1/project/assets/112654341/914bd53d-274c-4b17-b6a3-5854c8c7d2d6)

### AWS Infrastructure Overview:
- VPC with 6 subnets (2 public, 2 private, 2 database)
- 3 route tables (public, private, database)
- Internet Gateway and NAT Gateway
- 4 Security Groups (for bastion server, ALB, app servers, RDS)
- RDS instance with MySQL engine
- EC2 instances (1 bastion server, 2 app servers)
- Application Load Balancer (ALB) with target group

### Step-by-Step guide for deployment

### Step 1: Create VPC and Subnets
Create a VPC with 6 subnets (2 public, 2 private, 2 database), and set up route tables, Internet Gateway, and NAT Gateway.

**`vpc.tf`**

```hcl
resource "aws_vpc" "skyage_vpc" {
  cidr_block           = "172.16.0.0/16"
  enable_dns_hostnames = true
  tags = {
    Name = "skyage-vpc"
  }
}

# Public Subnets
resource "aws_subnet" "public_subnet1" {
  vpc_id            = aws_vpc.skyage_vpc.id
  cidr_block        = "172.16.1.0/24"
  availability_zone = "ap-south-1a"
  tags = {
    Name = "PublicSubnet1"
  }
}

resource "aws_subnet" "public_subnet2" {
  vpc_id            = aws_vpc.skyage_vpc.id
  cidr_block        = "172.16.2.0/24"
  availability_zone = "ap-south-1b"
  tags = {
    Name = "PublicSubnet2"
  }
}

# Private Subnets
resource "aws_subnet" "private_subnet1" {
  vpc_id            = aws_vpc.skyage_vpc.id
  cidr_block        = "172.16.3.0/24"
  availability_zone = "ap-south-1a"
  tags = {
    Name = "PrivateSubnet1"
  }
}

resource "aws_subnet" "private_subnet2" {
  vpc_id            = aws_vpc.skyage_vpc.id
  cidr_block        = "172.16.4.0/24"
  availability_zone = "ap-south-1b"
  tags = {
    Name = "PrivateSubnet2"
  }
}

# Database Subnets
resource "aws_subnet" "database_subnet1" {
  vpc_id            = aws_vpc.skyage_vpc.id
  cidr_block        = "172.16.5.0/24"
  availability_zone = "ap-south-1a"
  tags = {
    Name = "DatabaseSubnet1"
  }
}

resource "aws_subnet" "database_subnet2" {
  vpc_id            = aws_vpc.skyage_vpc.id
  cidr_block        = "172.16.6.0/24"
  availability_zone = "ap-south-1b"
  tags = {
    Name = "DatabaseSubnet2"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "skyage_igw" {
  vpc_id = aws_vpc.skyage_vpc.id
  tags = {
    Name = "skyage-igw"
  }
}

# NAT Gateway
resource "aws_eip" "nat_eip" {
  vpc = true
}

resource "aws_nat_gateway" "skyage_nat_gw" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet1.id
  tags = {
    Name = "skyage-nat-gw"
  }
}

# Route Tables and Associations
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.skyage_vpc.id
  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public_subnet_association1" {
  route_table_id = aws_route_table.public_rt.id
  subnet_id      = aws_subnet.public_subnet1.id
}

resource "aws_route_table_association" "public_subnet_association2" {
  route_table_id = aws_route_table.public_rt.id
  subnet_id      = aws_subnet.public_subnet2.id
}

resource "aws_route" "route_igw" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.skyage_igw.id
}

resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.skyage_vpc.id
  tags = {
    Name = "private-rt"
  }
}

resource "aws_route_table_association" "private_subnet_association1" {
  route_table_id = aws_route_table.private_rt.id
  subnet_id      = aws_subnet.private_subnet1.id
}

resource "aws_route_table_association" "private_subnet_association2" {
  route_table_id = aws_route_table.private_rt.id
  subnet_id      = aws_subnet.private_subnet2.id
}

resource "aws_route" "route_nat_gw" {
  route_table_id         = aws_route_table.private_rt.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.skyage_nat_gw.id
}

resource "aws_route_table" "database_rt" {
  vpc_id = aws_vpc.skyage_vpc.id
  tags = {
    Name = "database-rt"
  }
}

resource "aws_route_table_association" "database_subnet_association1" {
  route_table_id = aws_route_table.database_rt.id
  subnet_id      = aws_subnet.database_subnet1.id
}

resource "aws_route_table_association" "database_subnet_association2" {
  route_table_id = aws_route_table.database_rt.id
  subnet_id      = aws_subnet.database_subnet2.id
}
```

### Step 2: Create Security Groups
Define security groups for the bastion server, ALB, app servers, and RDS.


![Untitled Diagram drawio (23)](https://user-images.githubusercontent.com/112654341/228327387-8057dbb8-060a-462a-978f-237e3d047426.png)




**`security-group.tf`**

```hcl
# Security Group for Bastion Server
resource "aws_security_group" "bastion_sg" {
  name        = "bastion-sg"
  description = "Allow port 22 from anywhere"
  vpc_id      = aws_vpc.skyage_vpc.id
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
}

# Security Group for ALB
resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow port 80 and 443 from anywhere"
  vpc_id      = aws_vpc.skyage_vpc.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group for App Servers
resource "aws_security_group" "app_sg" {
  name        = "app-sg"
  description = "Allow port 22 from bastion-sg and 80 from alb-sg"
  vpc_id      = aws_vpc.skyage_vpc.id
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion_sg.id]
  }
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group for RDS
resource "aws_security_group" "rds_sg" {
  name        = "rds-sg"
  description = "Allow port 3306 from app-sg only"
  vpc_id      = aws_vpc.skyage_vpc.id
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Step 3:  Create RDS Instance

 
Provision an RDS instance with MySQL as the database engine.

**`rds.tf`**

```hcl
resource "aws_db_subnet_group" "skyage_db_subnet_group" {
  name       = "skyage-db-subnet-group"
  subnet_ids = [aws_subnet.database_subnet1.id, aws_subnet.database_subnet2.id]
  tags = {
    Name = "skyage-db-subnet-group"
  }
}

resource "aws_db_instance" "skyage_rds" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t2.micro"
  name                 = "wordpressdb"
  username             = "admin"
  password             = "password"
  parameter_group_name = "default.mysql8.0"
  db_subnet_group_name = aws_db_subnet_group.skyage_db_subnet_group.name
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  skip_final_snapshot  = true
  tags = {
    Name = "skyage-rds"
  }
}
```

### Step 4: Create EC2 Instances
Set up EC2 instances for the bastion server and app servers.

**`ec2.tf`**

```hcl
# Bastion Server
resource "aws_instance" "bastion" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet1.id
  security_groups = [aws_security_group.bastion_sg.name]

  tags = {
    Name = "bastion-server"
  }
}

# App Servers
resource "aws_instance" "app_server1" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.private_subnet1.id
  security_groups = [aws_security_group.app_sg.name]

  tags = {
    Name = "app-server1"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd php php-mysqlnd
              systemctl start httpd
              systemctl enable httpd
              wget https://wordpress.org/latest.tar.gz
              tar -xzf latest.tar.gz
              cp -r wordpress/* /var/www/html/
              chown -R apache:apache /var/www/html/*
              EOF
}

resource "aws_instance" "app_server2" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.private_subnet2.id
  security_groups = [aws_security_group.app_sg.name]

  tags = {
    Name = "app-server2"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd php php-mysqlnd
              systemctl start httpd
              systemctl enable httpd
              wget https://wordpress.org/latest.tar.gz
              tar -xzf latest.tar.gz
              cp -r wordpress/* /var/www/html/
              chown -R apache:apache /var/www/html/*
              EOF
}
```

### Step 5: Create ALB and Target Group
Set up an Application Load Balancer and target group for the app servers.

**`alb.tf`**

```hcl
resource "aws_lb" "skyage_alb" {
  name               = "skyage-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.public_subnet1.id, aws_subnet.public_subnet2.id]
  enable_deletion_protection = false
  tags = {
    Name = "skyage-alb"
  }
}

resource "aws_lb_target_group" "skyage_tg" {
  name        = "skyage-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.skyage_vpc.id
  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
    matcher             = "200-399"
  }
  tags = {
    Name = "skyage-tg"
  }
}

resource "aws_lb_listener" "skyage_listener" {
  load_balancer_arn = aws_lb.skyage_alb.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.skyage_tg.arn
  }
}

resource "aws_lb_target_group_attachment" "app_server1_attachment" {
  target_group_arn = aws_lb_target_group.skyage_tg.arn
  target_id        = aws_instance.app_server1.id
  port             = 80
}

resource "aws_lb_target_group_attachment" "app_server2_attachment" {
  target_group_arn = aws_lb_target_group.skyage_tg.arn
  target_id        = aws_instance.app_server2.id
  port             = 80
}
```

### Step 6: Apply Terraform Configuration
1. Initialize Terraform:
   ```bash
   terraform init
   ```
2. Validate the configuration:
   ```bash
   terraform validate
   ```
3. Plan the deployment:
   ```bash
   terraform plan
   ```
4. Apply the configuration:
   ```bash
   terraform apply
   ```

### Step 7: Configure your application

- copy DNS name of pr8-alb and paste to new tab

- setup username & password for application & install

- login application dashboard & go through the dashboard

- open website which will be publicly available for end-users

By following these steps, you will have deployed a WordPress application on AWS using Terraform. 
The infrastructure includes a VPC, subnets, security groups, RDS instance, EC2 instances, and an Application Load Balancer. 
This setup ensures a scalable and secure environment for your WordPress application.
