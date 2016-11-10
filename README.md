# apigee-cloudfoundry-edgemicro

The purpose of this repository is to:
1. spin up a Microgateway instance with Bosh
2. spin up a sample Spring Boot target service (hello world)
3. restrict access to the target service via Microgateway (firewall rules)
4. spin up an Nginx instance to sit in front of Micrograteway to terminate SSL and load balance multiple Microgateway instances

A subsequent goal is to start both Microgateway and Spring Boot on the same VM.

## What is completed?
Items 1 and 2 are complete.  

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

6.  Pull the git submodule located in `src/hello_world_spring`.

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

## Test Microgateway
Setup a route to your Bosh instance.
```
sudo route add -net 10.244.0.0/19 192.168.50.4
```

Linux users
```
sudo ip route add 10.244.0.0/19 via 192.168.50.4
```

### Authorization Token
First you have to obtain an access token.
```
curl -i -X POST --user client_id:client_secret "https://org-env.apigee.net/edgemicro-auth/token" -d '{"grant_type": "client_credentials"}' -H "Content-Type: application/json"
```

The response is show below.
```
{"token":"jwt"}
```

### Hello Greeting
The above response will have a JWT. Include that in the `Authorization` header as a `Bearer` token.

```
curl -H "Authorization: Bearer jwt" http://10.244.0.2:8000/edgemicro_bosh_hello/greeting
```

Response
```
{"id":2,"content":"Hello, World!"}
```

## Test Spring Boot Hello World!
You can access the target server directly with the following curl command.

```
curl http://10.244.0.6:8080/greeting
```

## Logging
There are several log files that are created and you can ssh into the VM to see the logs. Or use Bosh cli to download the log file to your local machine.

### Logging Directory
```
/var/vcap/sys/log/edgemicro
```

### Log Files Created
Log files created are:
```
edgemicro_init.log
edgemicro_configure.log
edgemicro.stderr.log
key.log
secret.log
startup.log
```

## Submodules
This product uses the following submodule, which is stored in `src/hello_world_spring`.
https://github.com/spring-guides/gs-rest-service
