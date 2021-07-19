# docker-traefik-intro
Traefik Proxy in a Docker container with TLS using a Let’s Encrypt certificate 
and TLS DNS-01 challenge all inside your private network

Leon Steenkamp  
2021-07-18


## Introduction
This project uses docker-compose to run a Traefik reverse proxy container and 
a _whoami_ container to test against.

Traefik Proxy serves as a TLS endpoint for the hosts behind the proxy. Let’s 
Encrypt's DNS-01 challenge is used for certificate generation because the proxy 
is not accessable from the Internet. This example only shows HTTP/S web entry 
points but other entry points (UDP/TCP) are possible.

The different challenge types that can be used with Let's Encrypt are covered 
at here - https://letsencrypt.org/docs/challenge-types/

_Requirements for this project_: You will need a host with docker and 
docker-compose installed, an AWS account and domain that you control in AWS 
Route53.

### Sections
The README is divided into the following sections:

1. Create AWS IAM user
2. Docker-compose file
3. Traefik static configuration
4. Traefik dynamic configuration

This README assumes you control the _example.co.za_ domain. Traefik is 
configured using YAML configuration files.

### Traefik documentation links
* Traefik documentation - https://doc.traefik.io/traefik/  
* Docker-compose with let's encrypt: DNS Challenge example - https://doc.traefik.io/traefik/user-guides/docker-compose/acme-dns/  

### Useful addresses
Some useful addresses after you have the container up and running:

* http://_traefik-host-address_:8080/ping
* http://_traefik-host-address_:8080/dashboard
* https://whoami.example.co.za/


## Configure IAM user
For the Let’s Encrypt _DNS-01_ challenge to work an AWS IAM user and policy have 
to be created for Traefik to use. The process is explained in the Traefik 
documentation but there are a few things that tripped me up the first time. 
Please note that I am not an experienced Route53 user, be sure you are 
comfortable with the changes you are making to your setup.

Log into AWS with a user that can create IAM users and policies. Under IAM 
Policies create a policy using the JSON tab and use the policy in JSON format 
from the second link below. The policy editor might complain about the empty 
_Sid_ values. Add your own _Sid_ if you want. The policy is also shown below.

"The Sid (statement ID) is an optional identifier that you provide for the 
policy statement." - https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_sid.html

```JSON
{
   "Version": "2012-10-17",
   "Statement": [
       {
           "Sid": "",
           "Effect": "Allow",
           "Action": [
               "route53:GetChange",
               "route53:ChangeResourceRecordSets",
               "route53:ListResourceRecordSets"
           ],
           "Resource": [
               "arn:aws:route53:::hostedzone/*",
               "arn:aws:route53:::change/*"
           ]
       },
       {
           "Sid": "",
           "Effect": "Allow",
           "Action": "route53:ListHostedZonesByName",
           "Resource": "*"
       }
   ]
}
```

Create an IAM user with programmatic access and use "attach existing policies 
directly" to add the policy created above (you might need use the filter to 
find your policy). Make a note of the user's access key and access key ID.

While in the AWS management console make a note of the Route53 _Hosted zone ID_ 
the domain is under. The key and two IDs are used in the Traefik configuration.

This process is also explained in the link below:
1. https://doc.traefik.io/traefik/https/acme/
2. https://go-acme.github.io/lego/dns/route53/


## Docker-compose file
The docker-compose file is based on the examples in the Traefik documentation. 

The container forwards three ports for HTTP and HTTPS traffic. The Traefik 
dashboard is also exposed.

For my example reverse proxy deployment mostly pointed to services running on 
external hosts and not services running in other docker containers on a docker 
network. The _whoami_ container gives you an idea how labels can be used to 
configure Traefik.

It looks like Traefik needs to be restarted when new router rules with new 
hostnames are added. Otherwise, certificates are not created for the new hosts.

### Volumes
Getting the Traefik dynamic configuration to load dynamically when using docker 
took some trial, error and searching. The trick seems to be mounting a directory 
and not the file directly and telling Traefik to look at the directory rather 
than just the file. This also covered in the Traefik configuration section. 

The created certificates are stored in the _letsencrypt\\_ directory.

### Docker secrets
As shown in the Traefik documentation, docker secrets are used to store the 
AWS IAM user credentials.

It was not immediatly clear what environment variables were needed to pass the 
Route53 credentials to Traefik, but after reading the _LEGO_ page carefully 
you'll notice that AWS_SHARED_CREDENTIALS_FILE should be used instead of 
AWS_ACCESS_KEY_ID_FILE and AWS_SECRET_ACCESS_KEY_FILE when using files.

The _secrets\\_ directory is used to store the details used by the Let’s 
Encrypt DNS-01 challenge. I used AWS Route53 for the DNS-01 challenge, the 
values needed here will probably differ if you use another provider.

In the _secrets\\_ directory you need to create two files:
1. aws_hosted_zone_id.secret
2. aws_shared_credentials.secret

The only contents in the _aws_hosted_zone_id.secret_ file is the zone ID 
 where the domain you are using lives.

The contents of the _aws_shared_credentials.secret_ is shown below:

```ini
[default]
aws_access_key_id = add-your-id-here
aws_secret_access_key = add-your-key-here
```

Add the details from the AWS IAM user you created in these two files.


## Traefik static configuration
The **traefik.yml** static configuration file is based on the examples in the 
Traefik documentation. 

Traefik uses the concepts of entry points, routers, middlewares and services. 
These are explained in the documentation and it might take a few reads to fully 
understand. 

Two entry points are defined for _web_ and _websecure_ that handle HTTP traffic 
on port 80 and 443. I added redirection from the _web_ to _websecure_ entry points. 

Logging is set to debug, so you might want to change this. Logging can be seen 
when _docker-compose_ is run without the _-d_ (detached) option. 

The other important section to notice is _certificatesResolvers_. This is where 
Let's Encrypt comes in. Add a valid email address to this section.

The docker provider allows for label based dynamic configuration when using 
docker contrainers. The file provider allows for file-based dynamic 
configuration. 

**Note**: When using docker is looks like specifiying a directory to watch 
for a dynamic configuration file works better than specifying a file directly.


## Traefik dynamic configuration
The **dynamic_conf.yml** dynamic configuration file is based on the examples in 
the Traefik documentation. 

The dynamic configuration is reloaded while Traefik is running without having 
to restart Traefik (altough it might be needed to restart Traefik if a new host 
is added to force certificate generation if a certificate does not exist).

### Services
Two example services are specified, one that forwards traffic to a specific port 
and the other that defaults to port 80. 

A _serversTransport_ is specified that skips TLS verification to these hosts, 
otherwise, a browser warning is generated. This is a bit of a chicken and egg 
problem if not skipping the check. One of the reasons for running the reverse 
proxy is to handle TLS certificates, if these hosts could provide valid 
certificates then this exercise would not be needed. The assumption is that the 
network and hosts behind the reverse proxy is trusted.

Services can specify more than one host which allows for load balancing by 
Traefik. If there is only one host, then you only have to specifiy one host. 
Replace the example IP addresses with addresses to your services.

### Routers
_Routers_ join entry points to services while _middlewares_ do any 
modifications that might be needed in between. 

In the example two routers listen on the _websecure_ entry point with rules 
for two different subdomains. Traffic for each subdomain is forwarded to a 
different service and both routers make use of the _certResolver_ specified in 
the Traefik static configuration.

An example of a _middleware_ is shown where a path is added to the request sent 
to the service (host). In this case _/admin_ is added - the Pi-hole admin 
panel is located at this path. The Pi-hole admin panel can be accessed from 
_hole.example.co.za_ without needing the _/admin_ path at the end.
