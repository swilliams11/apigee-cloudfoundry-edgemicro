# apigee-cloudfoundry-edgemicro

The purpose of this repository is to:

1. spin up a Microgateway instance with Bosh
2. spin up a sample Spring Boot target service (hello world)
3. restrict access to the target service via Microgateway (firewall rules)
4. spin up an Nginx instance to sit in front of Micrograteway to terminate SSL and load balance multiple Microgateway instances

A subsequent goal is to start both Microgateway and Spring Boot on the same VM.  This task was completed with the [edgemicro-decorator](https://github.com/swilliams11/edgemicro-decorator).

## What is completed?
Items 1, 2, 3 and 4 are complete.  
I used the existing [Bosh Nginx release](https://github.com/cloudfoundry-community/nginx-release) to handle this.  

TLS is configured on Nginx with a self-signed certificate.  

## Bosh Lite
I used [Bosh lite](https://github.com/cloudfoundry/bosh-lite) to test it.

## Install Bosh CLI
Install the [Bosh cli](https://bosh.io/docs/bosh-cli.html).

## Summary
This script will install the latest version of Microgateway via NPM. This will start one instance (VM) and start Microgateway. Then it will spin up another instance of the Spring Boot Hello World application.  Please read through the prerequisites before you attempt to use this Bosh script.


## Prerequisites

1. You must have an Apigee Edge account, either in the cloud or on-premises.

2. You must setup [Microgateway aware proxies in Apigee Edge](http://docs.apigee.com/microgateway/latest/edge-microgateway-tutorial#Part2).   

3. Install [Bosh CLI](http://bosh.io/docs/bosh-cli.html).

4. Before you run the following commands. Make sure to update the following
properties in the `apigee-cloudfoundry-edgemicro/jobs/microgateway/spec` file.

```
properties:
  port:
    description: "Port on which edgemicro is listening"
    default: 8000
  org:
    default: org
  env:
    default: env
  username:
    default: orgadmin
  password:
    default: password
  cluster_processes:
    default: 2
```

5. Because Github has a file size limit, I could not include the JDK 8 in the blob directory.  Therefore, you must download JDK 8 update 111 for linux x-64 and place it in the following directory. It must appear exactly as shown below.

```
blobs/jdk_8u111/jdk-8u111-linux-x64.tar.gz
```

6.  Pull the git submodule located in `src/hello_world_spring`. See the [Git Submodules section](#git-submodules).

7. Pull the forked copy of [Nginx-release](https://github.com/swilliams11/nginx-release) and follow the instructions in that README to start Nginx.  

## Deploy to Bosh
The following commands assume that you are using Bosh lite. It will setup your bosh lite environment
and it will try to start up a new VM with microgateway.

```
cd apigee-cloudfoundry-edgemicro

bosh target 192.168.50.4 lite

bosh deployment manifest.yml

wget --content-disposition https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent

bosh upload stemcell bosh-stemcell-3262.2-warden-boshlite-ubuntu-trusty-go_agent.tgz

bosh create release --force

bosh upload release
bosh releases

bosh deploy
```

### Other commands to redeploy and stop VMs if you make any changes
```
bosh deploy --recreate
bosh vms

bosh stop --hard --force

bosh delete deployment microgateway
```

## SSH Into VM
```
bosh ssh
```

## Setup a Route to Bosh
Setup a route to your Bosh instance.
The `10.244.5.0/24` is for Nginx.

```
sudo route add -net 10.244.1.0/24 192.168.50.4
sudo route add -net 10.244.5.0/24 192.168.50.4

```

Linux users
```
sudo ip route add 10.244.1.0/24 via 192.168.50.4
sudo ip route add 10.244.5.0/24 via 192.168.50.4
```

## Nginx Configuration
I forked the [Bosh Nginx release](https://github.com/cloudfoundry-community/nginx-release), therefore the Nginx configuration is completed [here](https://github.com/swilliams11/nginx-release).

### Send a Request to Nginx
Request flows as shown below:
```
Nginx -> Microgateway -> Spring Boot Target Service
```

#### 1. Obtain an Authorization Token
First you have to obtain an access token.
```
curl -i -X POST --user client_id:client_secret "https://org-env.apigee.net/edgemicro-auth/token" -d '{"grant_type": "client_credentials"}' -H "Content-Type: application/json"
```

The response is show below.
```
{"token":"jwt"}
```

#### 2a. Send the Request with the Authorization Token and Use Nginx IP Address
Take the JWT from the response above and include it in the `Authorization` header as a `Bearer` token.  In this case I need to pass `--insecure` because I used a self-signed certificate to configure TLS on Nginx.
```
curl -X GET -H "Authorization: Bearer jwt" https://10.244.5.2/edgemicro_bosh_hello/greeting --insecure
```
If you don't include the Authorization header then you will receive a 401 unauthorized error.

#### 2b. Send the Request Using Domain Name
To use this approach you have to edit your `/etc/hosts` file to include the following line:
```
10.244.5.2      springhello.io
```

Then execute the following command.
```
curl -X GET -H "Authorization: Bearer jwt" https://springhello.io/edgemicro_bosh_hello/greeting --insecure
```

##### Response
```
{"id":2,"content":"Hello, World!"}
```

### Direct Access to Microgateway was Disabled
Firewall rules will be configured such that access to the Microgateway is only allowed via Nginx.  
However, at this time you can send requests directly to the microgateway.
```
curl -H "Authorization: Bearer jwt" http://10.244.1.2:8000/edgemicro_bosh_hello/greeting
```

## Target Server - Test Spring Boot Hello World!
Note that you cannot access the target server directly with the following curl command.

```
curl http://10.244.1.6:8080/greeting
```

You should receive the error below. The reason is that the firewall only allows access via the Microgateway.
```
curl: (7) Failed to connect to 10.244.1.6 port 8080: Operation timed out
```

## Logging
There are several log files that are created and you can ssh into the VM to see the logs. Or use Bosh cli to download the log file to your local machine.

To download the logs:
```
bosh vms
```
Which will display something like:

```
+------------------------------------------------+---------+----+---------+------------+
| VM                                             | State   | AZ | VM Type | IPs        |
+------------------------------------------------+---------+----+---------+------------+
| nginx/0 (2c5cf105-1ddc-41ab-a115-0171bdbe7d83) | running | z1 | small   | 11.24.51.2 |
+------------------------------------------------+---------+----+---------+------------+
```

Then enter the following command. This will download the log to your local machine.
```
bosh logs job 2c5cf105-1ddc-41ab-a115-0171bdbe7d83
```

### Microgateway Logging Directory
```
/var/vcap/sys/log/edgemicro
```

### Microgateway Log Files Created
Log files created are:
```
edgemicro_init.log
edgemicro_configure.log
edgemicro.stderr.log
key.log
secret.log
startup.log
```

## Git Submodules
This product uses the following submodule, which is stored in `src/hello_world_spring`.
https://github.com/spring-guides/gs-rest-service


```
cd src/hello_world_spring

git submodule init

git submodule update
```
