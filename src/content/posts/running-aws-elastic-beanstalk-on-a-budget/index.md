---
id: 'b0516270-9a71-4e06-9597-f46fd4bdd3d1'
title: 'Running AWS Elastic Beanstalk on a budget'
description: "A simple guide to running an AWS Elastic Beanstalk server on the lowest possible budget possible, with none of the features you don't need."
date: '2019-10-27 00:16:14'
tags:
  - AWS
  - DevOps
---

When it comes to getting apps up and running fast in AWS, Elastic Beanstalk is an attractive option
for teams that don't want to get bogged down with dealing with lower-level infrastructure.
Unfortunately, as Elastic Beanstalk is also aimed at scaling up to enterprise applications, you can
also incur non-obvious costs if you don't know what you're doing.

For small, low traffic projects, where you might only want to run a single application server and
database, Elastic Beanstalk can quickly become uneconomical. We'll be discussing what these costs
are and how we can avoid them.

## The costs of scale

Elastic Beanstalk (EB) provides out-of-the-box functionality for things like auto-scaling, load
balancing, networking and connecting to a database. These features are provided by other AWS
services and their usage is not necessarily free or cheap. Enabling some of these features without
due-diligence can lead us to having a disproportionately expensive application.

### Instance types

EB runs on top of EC2 and you get to choose what instance type you want, so if you're on a budget,
you will probably want to go for something on a lower spec like a `t3.small` or `t3.micro`.
Unfortunately, EB does not automatically provide options for the substantially cheaper spot instance
equivalents.

[Spot instances](https://aws.amazon.com/ec2/spot/pricing/) are much better value for money than the
on-demand (default) variants, and for our lower spec `t3` types, spot instances are roughly 33% of
the price! The only negative is that spot instances do incur the risk of being less reliable and can
be unexpectedly terminated if the spot price goes above what you're willing to pay for them.

Whilst EB doesn't offer spot instances by default, you can actually enable them via an extension
that can be deployed with your application bundle. Simply add the following into a `.config` file
in your extensions directory like `.ebextensions/spot-price.config`.

```yaml
Resources:
  AWSEBAutoScalingLaunchConfiguration:
    Type: 'AWS::AutoScaling::LaunchConfiguration'
    Properties:
      SpotPrice:
        'Fn::GetOptionSetting':
          Namespace: 'aws:elasticbeanstalk:application:environment'
          OptionName: 'EC2_SPOT_PRICE'
          DefaultValue: { 'Ref': 'AWS::NoValue' }
```

Then you just need to set the `EC2_SPOT_INSTANCE` environment variable through your application
configuration in the EB UI (under 'Software'). This should be the price you're willing to pay, so I
would recommend setting it to at least 70% of the on-demand price to ensure that your application
will be _nearly_ guaranteed to be running. There are no actual guarantees your instance will be
running 99.99+% of the time, but this should be good enough for staging environments or applications
where that level of up-time is not critical.

See this [Stack Overflow answer](https://stackoverflow.com/questions/51336418/how-to-use-spot-instance-with-amazon-elastic-beanstalk?answertab=votes#tab-top)
for a more comprehensive breakdown of how to use spot instances.

### Auto-scaling

EB is marketed as 'impossible to outgrow', which means that by default an EC2 auto-scaling group is
setup for you. This will try to auto-scale your number of instances up if there are too many
requests coming through. Generally, you should restrict it to only scale to the number of instances
that you are willing to pay for. This can be done through the EB UI in your application
configuration (under 'Capacity').

For the types of application we're interested in, we probably just want to set the environment type
to 'single instance', preventing any auto-scaling. This configuration particularly works well in
combination with my next suggestion on load balancing. If creating the environment through the EB
UI, this will be done as part of the default configuration, however you may need to specify it if
you intend to use something like Terraform to setup your infrastructure.

### Load balancing

To make EB's auto-scaling functionality work, a load balancer is required and by default this is
provided by the Elastic Load Balancing (ELB) service. It's a great way to provide this kind of
functionality to an application that needs to scale, but for our simple requirements it is totally
overkill.

Running an ELB load balancer will cost you at least ~$19.3 a month (and even more depending on your
bandwidth requirements). This is a substantial amount that essentially costs as much as the EC2
instances we intend to run our application on! It particularly doesn't make much sense to if we
decide to only run a single instance (as described in [Auto-scaling](#auto-scaling)).

We can completely disable load balancing from being used in EBS, either through the UI or upon
environment creation. Thankfully EB doesn't activate this by default when using the UI, but if you
use something like Terraform, you may need to specify this explicitly.

### Networking

Whilst our application might be simple, we should still try and have a good security posture to
avoid being targeted by bad actors. One of the simplest things we can do to create a security layer
is to setup a Amazon Virtual Private Cloud (VPC) with private and public subnets. This essentially
creates our own internal network where we can control which services are exposed to the internet.
VPCs are free so there really isn't any reason not to use them!

In a full-featured EB stack, we would typically want to place our application and database servers
inside of the private subnet. An ELB application load balancer could then be placed in the public
subnet and proxy any HTTP traffic through to the application servers. This means our application and
database servers are never fully exposed to the internet and significantly reduces the potential
attack surface area.

For our application stack, we only really need to place the database in the private subnet but won't
be able to do the same for our application server. This will need to be placed in the public subnet
to receive HTTP traffic, and you will also need to setup a security group to only allow the
application server to communicate with the database server.

When setting up your VPC, you will probably end up wondering if you need a NAT gateway. These are
typically used to enable communication between servers in the private subnet to servers in the
public subnet. For our use-case, they are unnecessary as we will be placing the application server
in the public subnet. NAT gateways are really expensive and come in at a minimum of ~$36.5 a month
(and much more depending on bandwidth usage). You should generally be avoiding these unless your
application depends on this kind of networking.

Setting up the full VPC is out of scope for this post, but I would recommend using something like
Terraform to simplify setting up this infrastructure as it's quite fiddly to be managing manually.
There are a number of open source modules that abstract away a lot of the complexity, but I would
recommend the following:

- [Cloud Posse's terraform-aws-vpc](https://github.com/cloudposse/terraform-aws-vpc)
- [AWS VPC Terraform module](https://github.com/terraform-aws-modules/terraform-aws-vpc)

Once your VPC is setup, you can just configure your EB environment to place any EC2 instances in the
VPC's public subnet(s) in the configuration UI (under 'Network').

### Database

EB can automatically provision an RDS database that our application can connect to. Unfortunately,
it doesn't give us much control over its configuration and tightly couples the two (which can be a
problem later). This can also make it harder to use it in our VPC's private subnet. Consequently, I
would recommend setting up the RDS database separately from EB.

As long as your VPC is setup correctly with the required security group to allow application to
database traffic, you should have no problem setting up your EB application server to connect to the
database as if they were in the same subnet.

In terms of the instance type, you will really only need a single `t3` instance type that can be
slightly less spec'd than the application server. Unfortunately, you can't make RDS instances any
cheaper unless you decide to go with a reserved instance, so on shoestring budget or if you're just
trying to create a staging environment, you might want to consider rolling your own database server
on EC2. Personally, I feel the peace of mind afforded by RDS is worth the higher price though (at
least in production).

## Conclusion

Elastic Beanstalk offers a range of features that make it a scalable solution for larger projects
that can expect high volume traffic, but for smaller projects these are often unnecessary features
that we should disable.

Looking at prices, the carte blanche Elastic Beanstalk experience will cost you (at least) the
following:

- On-demand EC2 instance (starting from `t3.small`) - **$17.2+** per month
- ELB load balancer - **$19.3+** per month
- VPC NAT gateway - **$36.5+** per month
- RDS database (starting from a PostgreSQL `t3.micro`) - **$15.3+** per month

This totals up to a staggering **$88.3** per month and is clearly excessive for most small
applications, testing environments, and similar workloads.

Comparatively, by not running most of these features and using spot instances, we can change our
costs to:

- Spot EC2 instance (starting from `t3.small`) - **$5.2+** per month
- RDS database (starting from a PostgreSQL `t3.micro`) - **$15.3+** per month

Our new total ends up being around **$20.5** per month, providing a massive 4x cost saving. It pays
to be careful with what features you enable in Elastic Beanstalk!

Thankfully, the default Elastic Beanstalk setup using the UI recommends a fairly reasonable
configuration for a basic application and shouldn't incur most of these costs. Just be aware that
costs can quickly balloon in AWS, so it's always good to do some research before buying into any of
their other services.
