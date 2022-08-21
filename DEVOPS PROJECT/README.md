using separate file for creating all the resources & a separate file for variables also. 

**Step 1:- Create Provider block**
  ```
  provider "aws" {
    region = "us-east-1"
    access_key = "{}"
    secret_key = "{}"
    version = "v2.70.0"
  }
  ```
**Step 2:- Create AWS VPC**
  ```
  resource "aws_vpc" "demovpc" {
     cidr_block       = "${var.vpc_cidr}"
     instance_tenancy = "default"
  tags = {
     Name = "Demo VPC"
   }
  }
  ``` 
**Step 3:- Create AWS Internet Gateway**
  ```
  resource "aws_internet_gateway" "demogateway" {
    vpc_id = "${aws_vpc.demovpc.id}"
  }
  ``` 
**Step 4:- Create AWS Subnets**
  ```
  # Creating 1st subnet 
  resource "aws_subnet" "demosubnet" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet_cidr}"
    map_public_ip_on_launch = true
    availability_zone = "us-east-1a"
    tags = {
      Name = "Demo subnet"
    }
  }
  # Creating 2nd subnet 
  resource "aws_subnet" "demosubnet1" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet1_cidr}"
    map_public_ip_on_launch = true
    availability_zone = "us-east-1b"
    tags = {
      Name = "Demo subnet 1"
    }
  }
  ```
**Step 5:- Create AWS Route Table**
  ```
  #Creating Route Table
  resource "aws_route_table" "route" {
    vpc_id = "${aws_vpc.demovpc.id}"
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.demogateway.id}"
      }
    tags = {
        Name = "Route to internet"
      } 
  }
  resource "aws_route_table_association" "rt1" {
    subnet_id = "${aws_subnet.demosubnet.id}"
    route_table_id = "${aws_route_table.route.id}"
  }
  resource "aws_route_table_association" "rt2" {
    subnet_id = "${aws_subnet.demosubnet1.id}"
    route_table_id = "${aws_route_table.route.id}"
  }
  ```

**Step 6:- Create AWS Security Group for Load Balancer**
  ```
  # Creating Security Group for ELB
  resource "aws_security_group" "demosg1" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"
  
  # Inbound Rules
  # HTTP access from anywhere
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  # HTTPS access from anywhere
    ingress {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  # SSH access from anywhere
    ingress {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  # Outbound Rules
  # Internet access to anywhere
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```
**Step 7:- Create AWS Load Balancer**

  ```
  resource "aws_elb" "web_elb" {
  name = "web-elb"
  security_groups = [
    "${aws_security_group.demosg1.id}"
  ]
  subnets = [
    "${aws_subnet.demosubnet.id}",
    "${aws_subnet.demosubnet1.id}"
  ]
  cross_zone_load_balancing   = true
  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    interval = 30
    target = "HTTP:80/"
  }
  listener {
    lb_port = 80
    lb_protocol = "http"
    instance_port = "80"
    instance_protocol = "http"
  }
  }
  ```
**Step 8:- Create AWS Launch configuration**


  ```
  resource "aws_launch_configuration" "web" {
    name_prefix = "web-"
    image_id = "ami-087c17d1fe0178315" 
    instance_type = "t2.micro"
    key_name = "tests"
    security_groups = [ "${aws_security_group.demosg.id}" ]
    associate_public_ip_address = true
    user_data = "${file("data.sh")}"
  lifecycle {
    create_before_destroy = true
  }
  }
  ```

**Step 9:- Create AWS Security group for EC2 instances**

  ```
  # Creating Security Group for ELB
  resource "aws_security_group" "demosg1" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"
  # Inbound Rules
  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # HTTPS access from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # Outbound Rules
  # Internet access to anywhere
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```


**Step 10:- Create AWS Auto Scaling Group**

  ```
  resource "aws_autoscaling_group" "web" {
    name = "${aws_launch_configuration.web.name}-asg"
    min_size             = 1
    desired_capacity     = 1
    max_size             = 2
  
    health_check_type    = "ELB"
    load_balancers = [
      "${aws_elb.web_elb.id}"
    ]
  launch_configuration = "${aws_launch_configuration.web.name}"
  enabled_metrics = [
    "GroupMinSize",
    "GroupMaxSize",
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupTotalInstances"
  ]
  metrics_granularity = "1Minute"
  vpc_zone_identifier  = [
    "${aws_subnet.demosubnet.id}",
    "${aws_subnet.demosubnet1.id}"
  ]
  # Required to redeploy without an outage.
  lifecycle {
    create_before_destroy = true
  }
  tag {
    key                 = "Name"
    value               = "web"
    propagate_at_launch = true
  }
  }
  ```
**Step 11:- Create AWS Auto Scaling Policy**

  ```
  resource "aws_autoscaling_policy" "web_policy_up" {
    name = "web_policy_up"
    scaling_adjustment = 1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_autoscaling_group.web.name}"
  }
  resource "aws_cloudwatch_metric_alarm" "web_cpu_alarm_up" {
    alarm_name = "web_cpu_alarm_up"
    comparison_operator = "GreaterThanOrEqualToThreshold"
    evaluation_periods = "2"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "120"
    statistic = "Average"
    threshold = "70"
  dimensions = {
    AutoScalingGroupName = "${aws_autoscaling_group.web.name}"
  }
  alarm_description = "This metric monitor EC2 instance CPU utilization"
  alarm_actions = [ "${aws_autoscaling_policy.web_policy_up.arn}" ]
  }
  resource "aws_autoscaling_policy" "web_policy_down" {
    name = "web_policy_down"
    scaling_adjustment = -1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_autoscaling_group.web.name}"
  }
  resource "aws_cloudwatch_metric_alarm" "web_cpu_alarm_down" {
    alarm_name = "web_cpu_alarm_down"
    comparison_operator = "LessThanOrEqualToThreshold"
    evaluation_periods = "2"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "120"
    statistic = "Average"
    threshold = "30"
  dimensions = {
    AutoScalingGroupName = "${aws_autoscaling_group.web.name}"
  }
  alarm_description = "This metric monitor EC2 instance CPU utilization"
  alarm_actions = [ "${aws_autoscaling_policy.web_policy_down.arn}" ]
  }
  ```
 
**Step 12:- Create terraform variable file**
  ```
  # Defining Public Key
  variable "public_key" {
    default = "tests.pub"
  }
  # Defining Private Key
  variable "private_key" {
    default = "tests.pem"
  }
  # Definign Key Name for connection
  variable "key_name" {
   default = "tests"
  }
  # Defining CIDR Block for VPC
  variable "vpc_cidr" {
    default = "10.0.0.0/16"
  }
  # Defining CIDR Block for Subnet
  variable "subnet_cidr" {
    default = "10.0.1.0/24"
  }
  # Defining CIDR Block for 2d Subnet
  variable "subnet1_cidr" {
    default = "10.0.2.0/24"
  }
  ```

**Step 13:- Create a user data file**

  ```
  sudo yum update -y
  sudo amazon-linux-extras install docker -y
  sudo service docker start
  sudo usermod -a -G docker ec2-user
  sudo chkconfig docker on
  sudo chmod 666 /var/run/docker.sock

  ```
* steps to create the infrastructure.

* `terraform init` initialize the working directory and downloading plugins of the AWS provider
* `terraform plan` create the execution plan for our code
* `terraform apply` create the actual infrastructure. 


After the infrastructure is ready we can verify the output by navigating `http://DNS-of-Load-Balancer` you should see the below output.

