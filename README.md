# IP-geolocation-containerized-project-with-Application-Load-Balancer

Here, I have created a program which provides us with the geolocation of an IP address using docker containers.

![docker](https://github.com/anandu79/IP-geolocation-containerized-project-with-Application-Load-Balancer/blob/main/images/docker.jpg?raw=true)

For that, we have to launch 3 Ammazon Linux EC2 instances.
I have changed their names to api-service-instance1, api-service-instance2, and api-caching-instance for better understanding.

We can change the hostname using:

```
hostnamectl set-hostname api-service-instance1
hostnamectl set-hostname api-service-instance2
hostnamectl set-hostname api-caching-instance
```

Install docker in all 3 servers using the below command:

```
yum install docker -y
```

Enable docker service:

```
systemctl enable docker.service
```

## Docker container creation in api-caching-instance

SSH into api-caching-instance and create a docker container using [redis](https://hub.docker.com/_/redis).

> Redis is an open source key-value store that functions as a data structure server.

You can use the below command to create the container:

```
docker container run --name redis -d --restart always -p 6379:6379 redis:latest
```

Use the below command to list the containers created in the server:

```
docker container ls -a
```

Output:

```
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                       NAMES
931794dc2c3a   redis:latest   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   redis
```

## Container creation in api-service-instance1

Now that we have created redis container in api-caching-instance, we have to create 2 docker containers ap-service-1 and ap-service-2 in api-service-instance1. For that, SSH into the instance and use the below command:

```
docker container run \ 
-d \ 
-p 8081:8080 \ 
--name api-service1 \ 
--restart always \ 
-e REDIS_PORT="6379" \ 
-e REDIS_HOST="private IP address of ap-caching-instance" \ 
-e APP_PORT="8080" \ 
-e API_KEY="your API key from https://app.ipgeolocation.io/" \ 
fujikomalan/ipgeolocation-api-service:latest
```

























