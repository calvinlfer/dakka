# Java and Node.JS Image
_Helping you deploy an Akka Cluster_

A compact docker image based on Alpine Linux with:

- JRE 8 (8u111)
- NodeJS 7 (7.7.1)
- kms-env 
- bootstrapping behavior for retrieving host ip /port within container

In addition to having Node.JS and Java 8 installed, this image
comes with [kms-env](https://github.com/ukayani/kms-env) installed.
If you pass in environment variables encrypted using `kms-env`. 
The image will automatically decrypt them. 

In order for decryption to work, your docker container must be running on 
an ec2-instance with a role that has access to AWS KMS (and the master 
key used for encryption).

## Running on ECS

In order to run an Akka Cluster on ECS in a way that supports [bin-pack](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-strategies.html)ing, you assign random ports on the ECS host for [clustering](http://doc.akka.io/docs/akka/current/scala/cluster-usage.html#a-simple-cluster-example). In your Docker container, you fix your clustering port to `2551` and bind to `0.0.0.0` meaning all interfaces. Let's see how to do this. Since you are dealing with Docker and ECS, you have to deal with Network Address Translation (NAT) you have to translate between the container IP and port to the ECS host IP and port so other members of the cluster can communicate with you. The idea is that you want your application to bind to `0.0.0.0` inside the Docker container and use port `2551` but you want to _advertise_ to all external parties (i.e. outside your Docker container) that you are addressable using the ECS host's IP and the ECS host's port which is mapped to the Docker container.

<img src=https://user-images.githubusercontent.com/14280155/29138263-221e6cd6-7d11-11e7-97b4-31a0282f5895.png width=300>

Akka supports this [kind of behavior](http://doc.akka.io/docs/akka/current/scala/remoting.html#akka-behind-nat-or-in-a-docker-container) via `bind-hostname` and `bind-port`. The idea is that you use the ECS machine's hostname (i.e. the IP of the machine hosting your Docker container) for `akka.remote.netty.tcp.hostname` and use the ECS machine's port mapped to your internal Docker container port for `akka.remote.netty.tcp.port`. For `akka.remote.netty.tcp.bind-hostname`, set this to `0.0.0.0` and set the `akka.remote.netty.tcp.bind-port` to `2551`. 

We mention that we need to set the `hostname` and the `port` to the ECS host that is hosting the Docker container. This image installs Docker and expects that you use volumes and map the `docker.sock` so that you can run `docker` commands within the running container. The purpose of doing this is to obtain how the ECS host's ports maps to the Container's ports using `docker inspect`. This is how we can find out the external port that mapped to the Docker container port. The last concern remains of figuring out the ECS host. This can be done via CloudFormation where the ECS host ip is placed in a file like `/etc/hostip`, you would also be expected to mount that file into this container in order to obtain the ECS host IP. 

You need to volume mount the Docker socket to obtain the external port and `/etc/hostip` to obtain the external IP of the host. You must also feed in an environment variable named `APP_PORT` which refers to the clustering port (`2551`). This will result into two environment variables being populated: `HOST_IP` which is the IP of the ECS host that is hosting the Docker container and `HOST_PORT` which is the port of the ECS host that is mapped to your Docker container's port (`2551`). This is accessible to your Akka application and you can use the following configuration to take advantage of the environment variables:

```hocon
akka {
  remote {
    enabled-transports = ["akka.remote.netty.tcp"]
    netty.tcp {
      bind-hostname = 0.0.0.0
      bind-port = 2551

      hostname = ${?HOST_IP}
      port = ${?HOST_PORT}
    }
  }
}
```

This will correctly set up your cluster so nodes can communicate with each other and form a cluster. 
