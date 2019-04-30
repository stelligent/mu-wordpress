# mu-wordpress

Run WordPress on Amazon ECS and RDS with mu

This is the samlpe code for Stelligent's blog post, [Mu, WordPress, and You](https://stelligent.com/2017/06/08/wordpress-mu-and-you/).


## Overview

We can use [mu](https://getmu.io) to:

+ customize the official [WordPress Docker image](https://hub.docker.com/r/_/wordpress/),
+ storing a copy in [Amazon's ECR Docker registry](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_Console_Repositories.html),
+ running it in [Amazon's EC2 Container Service](https://aws.amazon.com/ecs/),
+ fronted by an [Application Load Balancer](https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/),
+ backed by an [RDS Aurora database](https://aws.amazon.com/rds/aurora/),
+ deployed with [CodeBuild](https://aws.amazon.com/codebuild/)
and [CodePipeline](https://aws.amazon.com/codepipeline/),

Mu does it all for you through [AWS CloudFormation](https://aws.amazon.com/cloudformation/) using a pair of simple YAML files.


## Architectural Summary

### Components

GitHub stores your infrastructural code.

mu acts as your front-end to AWS by generating and applying CloudFront
templates, orchestrated by CodePipeline.

The official WordPress Docker container is deployed to Amazon ECS,
and your custom copy is stored in Amazon ECR.

An ECS cluster is run for each environment we define, "test" and "prod".

An AWS ALB sits in front of each cluster.

Your WordPress database will be provided by an Amazon RDS cluster, one for
each environment. Each runs Aurora, Amazon's highly optimized clone of MySQL.
   
### Flow

AWS CodePipeline orchestrates all the other services in AWS. It's 
used to give you a continuous delivery pipeline:

1. It watches your GitHub repo for changes and automatically applies
   them shortly after you push.
1. AWS CodeBuild uses `buildspec.yml` to run any custom steps you add
   there.
1. AWS CodeBuild generates your own Docker image by combining the results
   of the last step with the official WordPress image and storing it in
   Amazon ECR.
1. Your container is deployed to your "test" environment.
1. You manually inspect your container and approve or reject it.
1. If you approve it, your container is deployed to your "prod" environment.


## Getting Started

### Setup

[Fork](https://help.github.com/articles/fork-a-repo/)
https://github.com/stelligent/mu-wordpress into your own GitHub account,
and then clone it to your workstation:

    git clone _your_fork_of_mu-wordpress_
    cd mu-wordpress

(Why do all that instead of just cloning? You don't technically need to
fork the Stelligent mu-wordpress repo unless you want to follow its
changes, but it's a convenient way to get a copy in your GitHub account.)

CodePipeline is going to watch _your_ repo for changes, which will give
you power-user convenience: just push your code to trigger updates in
your WordPress deployment. [Infrastructure as Code](https://stelligent.com/2015/11/11/infrastructure-as-code-part-deux-a-hit-at-reinvent-2015/), amiright?

### Config

Next, edit `mu.yml` and change `pipeline.source.repo` to point to your
own GitHub account instead of "stelligent":

    pipeline:
      source:
        provider: GitHub
        repo: _your_github_username_/mu-wordpress

Set your AWS region if you want to use something other than the default,
`us-east-1`:

    export AWS_DEFAULT_REGION=us-west-2

Let's create a keypair you can use to debug any issues that might come
up on your containerized EC2 instances:

    ssh-keygen -C mu-wordpress -f ~/.ssh/mu-wordpress
    aws ec2 import-key-pair --key-name mu-wordpress \
     --public-key-material file:///$HOME/.ssh/mu-wordpress.pub

The key named "mu-wordpress" is referenced in `mu.yml`. If you use 
a different name, or don't want to define a key at all, be sure to
change the `keyName` definition there.

Commit your changes and push them back up to your GitHub account:

    git add mu.yml
    git commit -m'first config' && git push

### Spin it up

Start up your pipeline, which will deploy to 2 environments, "test" and
"prod":

    mu pipeline up

Mu will ask you for a GitHub token. CodePipeline uses it to watch your
repo for changes so that it can automatically deploy them.
[Create a new token](https://github.com/settings/tokens) in your own
GitHub account and grant it the "admin:repo_hook" and "repo" permissions.
Save it somewhere, like [a nice password manager](https://1password.com).
Enter it when mu asks for it. (But don't give it to anything else! ;^)

Watch your pipeline get deployed:

    mu pipeline logs -f

### Inspect

When that's done, you can verify that you have 2 environments, "test" and
"prod":
    
    mu env list

You'll see a table like this:

    +-------------+-----------------------+---------------------+---------------------+------------+
    | ENVIRONMENT |         STACK         |       STATUS        |     LAST UPDATE | MU VERSION |
    +-------------+-----------------------+---------------------+---------------------+------------+
    | test        | mu-cluster-test       | CREATE_COMPLETE     | 2017-05-23 14:48:04 | 0.1.13     |
    | prod        | mu-cluster-prod       | CREATE_COMPLETE     | 2017-05-23 16:23:28 | 0.1.13     |
    +-------------+-----------------------+---------------------+---------------------+------------+

You can view the details on any of the environments:

    mu env show test

If you want to watch the "test" environment's services get deployed, or view
logs from the "test" environment, try these:

    mu service logs -f test
    mu env logs -f test

### Initialize WordPress

When your test environment is initialized, you can load the WordPress
admin installer and get your site started. Find the base URL with:

    mu env show test

You'll see a block at the top that includes "Base URL":

    Environment:    test
    Cluster Stack:  mu-cluster-test (UPDATE_IN_PROGRESS)
    VPC Stack:      mu-vpc-test (UPDATE_COMPLETE)
    Bastion Host:   1.2.3.4
    Base URL:       http://mu-cl-some-long-uuid.us-west-2.elb.amazonaws.com

Append "/wp-admin" to that and load the URL in your browser:

    http://mu-cl-some-long-uuid.us-west-2.elb.amazonaws.com/wp-admin

Follow the instructions there to set up a WordPress admin user,
initialize the database, etc.


### Update your content

Everything in your repo's `html` directory will be installed in your
containers. Add files there and they'll end up in `/var/www/html`,
right alongside WordPress. Want to install persistent plugins?
Put them in `html/wp-content/plugins`. Want to install a theme?
Add it to `html/wp-content/themes`.

### Caveat

This is a really simple proof-of-concept for deploying and managing a
WordPress installation through code. Making it robust is more complex,
and not within the scope of the basic presentation here.


## FAQ

> How can I get my database passwords if mu manages them for me?

You can read them from Amazon's SSM ParameterStore:

    aws ssm get-parameters --names mu-database-mu-wordpress-test-DatabaseMasterPassword --with-decryption
    aws ssm get-parameters --names mu-database-mu-wordpress-prod-DatabaseMasterPassword --with-decryption

If you use different environment names, you can list all the available
parameters to find the right names:

    aws ssm describe-parameters


## References:

* https://getmu.io
* https://stelligent.com/category/mu/
* https://hub.docker.com/r/_/wordpress/
