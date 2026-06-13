# Cloud Infrastructure High Availability Design

![High Availability](https://img.shields.io/badge/Domain-High%20Availability-blue)
![AWS](https://img.shields.io/badge/AWS-Multi--AZ-orange)
![Azure](https://img.shields.io/badge/Azure-Availability%20Zones-blue)
![RTO](https://img.shields.io/badge/RTO-Under%2015%20min-green)
![RPO](https://img.shields.io/badge/RPO-Under%205%20min-green)
![Terraform](https://img.shields.io/badge/IaC-Terraform-purple)

---

## Project Overview

This project documents comprehensive high availability cloud infrastructure design across AWS and Azure — covering multi-AZ deployments, auto scaling groups, load balancing, database replication, disaster recovery planning, and business continuity procedures.

Built to demonstrate operational readiness for:
-  Cloud Infrastructure Engineer roles
-  Site Reliability Engineer roles
-  Solutions Architect roles
-  Platform Engineer roles

---

##  Availability Targets

| Tier | SLA Target | RTO | RPO | Architecture |
|---|---|---|---|---|
| Critical | 99.99% | 5 minutes | 1 minute | Multi-region active-active |
| High | 99.9% | 15 minutes | 5 minutes | Multi-AZ active-passive |
| Standard | 99.5% | 1 hour | 30 minutes | Single region Multi-AZ |
| Development | 99.0% | 4 hours | 1 hour | Single AZ |

---

##  High Availability Architecture

---

## Auto Scaling Configuration

```hcl
# Launch Template
resource "aws_launch_template" "app" {
  name_prefix   = "app-launch-template"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  monitoring {
    enabled = true
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    environment = var.environment
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "app-server"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = var.private_subnet_ids
  min_size            = 2
  max_size            = 10
  desired_capacity    = 2

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  target_group_arns         = [aws_lb_target_group.app.arn]
  health_check_type         = "ELB"
  health_check_grace_period = 300

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
    }
  }

  tag {
    key                 = "Name"
    value               = "app-server"
    propagate_at_launch = true
  }
}

# Scaling Policies
resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out"
  scaling_adjustment     = 2
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_autoscaling_policy" "scale_in" {
  name                   = "scale-in"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 600
  autoscaling_group_name = aws_autoscaling_group.app.name
}
```

---

## Application Load Balancer

```hcl
# Application Load Balancer
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = true
  enable_http2               = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb-logs"
    enabled = true
  }

  tags = {
    Name        = "app-alb"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Target Group with health checks
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 10
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = false
  }

  tags = {
    Name = "app-target-group"
  }
}

# HTTPS Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# HTTP to HTTPS redirect
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.app.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

## Disaster Recovery Runbook

```markdown
### DR Activation Triggers
- Primary region availability below 99%
- RTO threshold approaching
- AWS service health dashboard shows regional outage
- Business decision to failover

### Phase 1 — Assessment (0–5 minutes)
1. Confirm primary region failure is real not false positive
2. Check AWS Service Health Dashboard
3. Attempt to reach primary region resources
4. Notify DR team and management
5. Make go/no-go failover decision

### Phase 2 — Failover Execution (5–15 minutes)
1. Promote RDS read replica to primary in DR region
2. Update application configuration for new DB endpoint
3. Scale up DR region Auto Scaling Group
4. Update Route 53 health check to point to DR region
5. Verify traffic flowing to DR region
6. Test application functionality in DR region

### Phase 3 — Validation (15–30 minutes)
1. Verify application accessible in DR region
2. Check database connections working
3. Monitor CloudWatch dashboards
4. Confirm all integrations functional
5. Notify stakeholders of successful failover

### Phase 4 — Failback (When primary region recovers)
1. Verify primary region fully recovered
2. Synchronize data from DR back to primary
3. Test primary region before switching back
4. Update Route 53 to route back to primary
5. Scale down DR region after traffic confirmed
6. Document complete incident timeline
```

---

##  Chaos Engineering Tests

| Test | Target | Expected Behavior |
|---|---|---|
| AZ failure simulation | Terminate all instances in one AZ | ASG launches replacements in other AZ |
| Database failover | Reboot RDS primary | Automatic failover to standby within 60s |
| Load balancer test | Remove healthy targets | Traffic routes to remaining healthy targets |
| Network partition | Block traffic between subnets | Application degrades gracefully |
| CPU stress test | Spike CPU on all instances | ASG scales out within 5 minutes |

---

## Standards & Frameworks Referenced

- **AWS Well-Architected Framework** — Reliability Pillar
- **Azure Architecture Framework** — Reliability
- **Uptime Institute** — Tier standards for availability
- **ITIL v4** — Service continuity management
- **ISO 22301** — Business continuity management
- **NIST SP 800-34** — Contingency planning

---

## Tools & Technologies

![AWS](https://img.shields.io/badge/AWS-Multi--AZ-orange)
![Azure](https://img.shields.io/badge/Azure-Availability%20Zones-blue)
![Terraform](https://img.shields.io/badge/Terraform-IaC-purple)
![Route53](https://img.shields.io/badge/Route%2053-Failover-orange)
![CloudWatch](https://img.shields.io/badge/CloudWatch-Monitoring-yellow)
![Chaos Engineering](https://img.shields.io/badge/Chaos-Engineering-red)

---

## Author

**George Amankwaa Sarpong**
Cloud Infrastructure Engineer | High Availability & Disaster Recovery
📍 Accra, Ghana 
🔗 [LinkedIn](https://linkedin.com/in/georgesarpong)
🌐 [GitHub Portfolio](https://github.com/GeorgeSarpong)

---

*This project is part of a broader portfolio demonstrating readiness for Cloud Infrastructure Engineer and Site Reliability Engineer roles in the US market.*
