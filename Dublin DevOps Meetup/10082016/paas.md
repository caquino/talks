theme:Sketchnote
footer: _© Zendesk, 2016_ - Cassiano Aquino - caquino@zendesk.com
slidenumbers: true
![](https://media.licdn.com/mpr/mpr/AAEAAQAAAAAAAAeuAAAAJDcyMzRkZWY1LTZmNmUtNDA2Zi1iNTQwLTM3MzI3NDhlOTY1Yg.jpg)
# [FIT]The tales of a
# [FIT]*Lazy DevOps*
# [FIT]Building a *PaaS*

---
![](http://d16cvnquvjw7pr.cloudfront.net/www/img/p-brand/mentor-bg-2X.png)
#Lazy DevOps
Cassiano Aquino
Tech Lead DevOps Engineer @ Zendesk
EI5HPB
@syshero
http://syshero.org/

---
![filtered](https://www.geekinlibrariansclothing.com/wp-content/uploads/2015/06/11392783794_a9fba0350f_o.jpg)

---
![](http://static1.squarespace.com/static/5096baa8e4b09e89382757ce/50adfde2e4b03180f3e32ed6/5255131be4b0a343130d98f6/1381409876349/Bime_square_logo_rounded.png)
# [FIT]The legend says that
# [FIT]company was acquired and...

---
# Bussines requirements

- SOC2 Compliance
- Secure
- High Availability

###*BORING*

---
### Availability and Security

  - Multiple AZs
  - Private and Public subnets
  - Individual SGs
  - CIS Baseline

## Just follow AWS guidelines and CIS baseline
###*BORING*


---
### SOC2 Compliance

  - Audit everything, All changes requires a PR on GitHub
  - Deployments using Samson
  - Logging, Cloudwatch, and Elasticsearch streaming

###even more *BORING*

---
![](/Users/caquino/Desktop/presentations/lazy-devops/slack-imgs-1.gif)

---
![right filtered](https://imgflip.com/s/meme/Impossibru-Guy-Original.jpg)
# Developers requirements

- Minimum impact to the current workflow
- Avoid decreasing developers efficiency
- We don't care about this boring stuff

PS: I will try my best, I swear!


---
![filtered](http://shellaeversey.com/wp-content/uploads/2014/04/greedy-simpson.jpg)
#[FIT]*PERSONAL*
#[FIT]REQUIREMENTS

---
![right filtered](http://www.m2now.co.nz/wp-content/uploads/2015/09/lazy-fat-guy.jpg)
# Personal requirements

- Ephemeral
- Self-healing
- Minimum work
- Simple

---
![inline](/Users/caquino/Desktop/presentations/lazy-devops/document.png)

---
![inline](/Users/caquino/Desktop/presentations/lazy-devops/document1.png)

---
![inline](/Users/caquino/Desktop/presentations/lazy-devops/document2.png)

---
![filtered](http://static1.squarespace.com/static/531b83f4e4b045034d6a8cfa/t/53750ab4e4b02899e91ff39a/1400179396242/ephemeralcookie.jpg)

---
### Requirements

- No customer data on instances
  - S3
  - RDS
  - Amazon ElastiCache
  - Amazon ElasticSearch

---
![filtered](http://blocksandbricks.co.za/wp-content/uploads/2014/09/14-Self-Healing-300x225.jpg)
# [FIT]Self
# [FIT]*Healing*

---
### Requirements

- Automatic configuration
- Automatic application deployment
- Autoscaling Groups

---
### Bootstrap

1. Automation (chef/puppet/ansible/...) :x:
  1.1 Increase complexity
  1.2 Hard to get it right with ASG
2. Custom AMI per hostgroup :x:
  2.1 Hard to maintain and update
  2.2 No individual configuration

^ Automation was the easy answer to me, as it's something I'm used to
But what if I want something simpler?

---
### Bootstrap

- Base AMI + cloud-config :heavy_check_mark:
  - Defaults backed on the AMI
  - Customizations trough cloud-config

### Packer
  - [Example](https://github.com/bimeio/stack/blob/master/packer/base/packer.yml)

---
![right filtered](https://www.drupal.org/files/x-all-the-things-template.png)
### ASG

- Autoscale all the things!
  - Harder to maintain?
  - More Difficult to automate?
  - How to handle new nodes?

---
### Options

  - Automation :x:
  - Individual Images :x:
  - Base image, custom packages, and PMA :heavy_check_mark:

^ Not really harder to maintain, it's harder to understand as it's a different way of working.

---
![](https://media.licdn.com/mpr/mpr/shrinknp_400_400/AAEAAQAAAAAAAAJdAAAAJDU3M2M3ZjdhLWJjYzItNGVlMC1iZDQ2LTZkYmZmNDhhMzA1YQ.jpg)
#That is entirely illogical.
#[FIT]Packages?
^ Packaging also is not simple, right? wrong!


---
### Packaging

- fpm-cookery
  - [Consul recipe](https://github.com/bimeio/fpm-packages/blob/master/terraform/0.7.0/main.rb)
  - [Base config](https://github.com/bimeio/fpm-packages/blob/master/consul/files/etc/consul.d/server/consul.json)
  - [Scripts](https://github.com/bimeio/fpm-packages/blob/master/consul/files/usr/bin/consul-server-config)
- travis-ci
  - [Build](https://travis-ci.com/bimeio/fpm-packages)

---
### [FIT]OK, BUT WHAT ABOUT
### [FIT]REALLY REALLY REALLY
### [FIT]*CUSTOM*
### [FIT]CONFIGURATION?

---
## Option 1
### cloud-config

```
#cloud-config
bootcmd:
  - echo 'SERVER_ENVIRONMENT=${environment}' >> /etc/environment
  - echo 'SERVER_GROUP=${name}' >> /etc/environment
  - echo 'SERVER_REGION=${region}' >> /etc/environment
  - echo 'SERVER_HOSTGROUP=${hostgroup}' >> /etc/environment
  - systemctl enable consul-server-config
  - systemctl enable consul-server
runcmd:
  - systemctl start --no-block consul-server
```

---
### cloud-config

- Works well to individual node configuration
- It's not built to handle clusters and groups
- Does not support individual customization on ASGs

---
![](http://img.odometer.com/quill/b/f/1/e/4/1/bf1e41d1/73aa07868de202d9e7c401b82d60e66fd62bf5f6.jpg)
#[FIT]*P*oor
#[FIT]*M*an's
#[FIT]*A*utomation

---
![filtered](http://www.websitemarketing.co.nz/wordpress/wp-content/uploads/2014/07/silver-bullet.png)
#[FIT] *PMA* is *NOT*
#[FIT] automation!
#[FIT] *It's a hack that works for me!*

---
###Poor man's automation

- Abusing Systemd

  - [Configures Consul](https://github.com/bimeio/fpm-packages/blob/master/consul/files/usr/lib/systemd/system/consul-server-config.service) / [Starts Consul](https://github.com/bimeio/fpm-packages/blob/master/consul/files/usr/lib/systemd/system/consul-server.service)
  - [Format EBS](https://github.com/bimeio/stack/blob/master/packer/ecs/root/etc/systemd/system/format-var-lib-docker.service) / [Mount EBS](https://github.com/bimeio/stack/blob/master/packer/ecs/root/etc/systemd/system/var-lib-docker.mount)

---
### Infrastructure Configuration

  - Terraform
  - Packer
  - Consul
  - Vault

---
### Application Configuration

  - Consul - Service Discovery
    - registrator
    - envconsul
    - consul-template
  - Vault - Secret Management

---
### ECS

  - Service provisioning
  - Application deployment

---
![](https://www.ctl.io/knowledge-base/images/ecosystem-hashicorp-terraform.png)
#[FIT]Terraform

---
### Friendly advice

  - take good care of your state file
  - split your environments in different states
  - modules == functions
  - terraform fmt
  - terraform plan

---
### Conditionals

```javascript
variable "resource_enabled" {
  default = 0
}

resource "aws_instance" "main" {
  count = "${var.resource_enabled}"
  ...
}
```

---
### Terraform

- travis-ci + terraform validate

---
### Terraform 0.7

```javascript
data "aws_ami" "base_ami" {
  owners      = ["self"]
  most_recent = true
  filter {
    name   = "name"
    values = ["base/*"]
  }
  filter {
    name   = "tag:QAStatus"
    values = ["pass"]
  }
}
```

---
### Terraform 0.7

```javascript
data "aws_iam_policy_document" "consul_ro_policy" {
  statement {
    actions = [
      "ec2:Describe*",
      "autoscaling:Describe*",
    ]
    resources = [
      "*"
    ]
    effect = "Allow"
  }
}
```


---
### Infrastructure and Services

- Both managed by Terraform, different repositories
- Freedom to developers
- Easier to audit
- Incident blast management

---
### Services

```javascript
module "my_awesome_app" {
  source          = "github.com/bimeio/stack//service"
  name            = "my-awesome-app"
  image           = "my_awesome_app"
  port            = 80
  environment     = "${module.stack.environment}"
  cluster         = "${module.stack.cluster}"
  iam_role        = "${module.stack.iam_role}"
  security_groups = "${module.stack.internal_elb}"
  subnet_ids      = "${module.stack.internal_subnets}"
  log_bucket      = "${module.stack.log_bucket_id}"
  zone_id         = "${module.stack.zone_id}"
}
```

---
### What you get?

- an ECS task definition
- an ECS service definition
- a Consul service/health check

### External Service

- registrator register service with Consul
- NGINX upstream configuration from Consul

---
### NGINX+

```nginx
resolver localhost:53 valid=10s;

upstream backend {
    zone upstream_backend 64k;
    server service.consul service=my_awesome_app resolve;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

---
### What about new backends?

- consul-template

```go
{{ range $service := services }}
  {{ if ne .Name "consul" }}
upstream {{ $service.Name }} {
  zone $service.Name 64k;
  server service.consul service=$service.Name resolve;
}
  {{ end }}
{{ end }}
```

---
###[FIT]I *hope* we will live
###[FIT]happily ever after
###[FIT]*THE END*
