# IP-geolocation-containerized-project-with-Application-Load-Balancer

Here, I have created a program which provides us with the geolocation of an IP address using docker containers.

![docker](https://github.com/anandu79/IP-geolocation-containerized-project-with-Application-Load-Balancer/blob/main/images/docker.jpg?raw=true)

For that, we have to launch 3 Ammazon Linux EC2 instances.
I have changed their names to api-service-instance1, api-service-instance2, and api-caching-instance1 for better understanding.

We can change the hostname using:

```
hostnamectl set-hostname api-service-instance1
hostnamectl set-hostname api-service-instance2
hostnamectl set-hostname api-caching-instance1
```

Install docker in all 3 servers using the below command:

```
yum install docker -y
```

Enable docker service:

```
systemctl enable docker.service
```

## Docker container creation in api-caching-instance1

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

Now that we have created redis container in api-caching-instance, we have to create 2 docker containers api-service-1 and api-service-2 in api-service-instance1. For that, SSH into the instance and use the below command:

```
docker container run \ 
-d \ 
-p 8081:8080 \ 
--name api-service1 \ 
--restart always \ 
-e REDIS_PORT="6379" \ 
-e REDIS_HOST="private IP address of ap-caching-instance1" \ 
-e APP_PORT="8080" \ 
-e API_KEY="your API key from https://app.ipgeolocation.io/" \ 
fujikomalan/ipgeolocation-api-service:latest
```

***In `REDIS_HOST` we will have to add the private IP address of the api-caching-instance1. If the server reboots, the IP address will change and we will have to edit the code again. In order to resolve this, we will create a private hosted zone instead and create a zone that points to the private IP address of api-caching-intsnace1. Or else we can create a Redis cluster in Elasticache and point it to the private hosted zone.***

Here I am creating a Redis cluster in Elasticache, for that do the following.

Go to ElastiCache dashboard in your AWS > Create cluster > and select "Create Redis cluster"

1. In **Cluster mode**, select Disabled.

2. In **Cluster info**, provide name and a discription(optional)

3. In **Location** we can choose whether to host the cluster in the AWS Cloud or on premises. I am selecting **AWS Cloud** here. Multi-AZ should be enabled.

4. In **Cluster settings**, change the Node type to "cache.t2.micro". Number of replicas should be 2. 

5. In **Subnet group settings**, select "Choose existing subnet group".

6. Select "Specify Availability Zones" in **Availability Zone placements**, and select the regions and replicas as per your requirement.

7. Click on Next.

8. In Advanced settings, Enable "Encryption at rest" under the **Security** section. Encryption key should be set as default. Select a security group too.

9. Backup, Maintenance, and Logs sections are optional, configure if required.

10. Give an appropriate Tag and click on Next. Then review the settings and click on **Create**.

> *Please note that it will take some time to complete the creation of the redis cluster.*

After the creation of Redis cluster, go to Route53 and create a private hosted zone. I'm naming my private zone "ipgeolocation.local". Select the region according to your requirement, provide an appropriate tag and click on Create hosted zone. 

Then, create a CNAME Record under the private zone "ipgeolocation.local" which points to the Primary endpoint of the redis cluster. Give an appropriate Record name as per your requirement.
> Primary endpoint will be under the cluster details section in Elasticache.

After that we can edit the `REDIS_HOST` section in the above command and it will look like:

```
docker container run \ 
-d \ 
-p 8081:8080 \ 
--name api-service1 \ 
--restart always \ 
-e REDIS_PORT="6379" \ 
-e REDIS_HOST="redis.ipgeolocation.local" \ 
-e APP_PORT="8080" \ 
-e API_KEY="your API key from https://app.ipgeolocation.io/" \ 
fujikomalan/ipgeolocation-api-service:latest
```

In `API_KEY` section, we have to add our personal API key from the IP geolocation app directly and it is not secure. So in order to add the API key, we have to use a service called "Secrets Manager" in AWS.
> AWS Secrets Manager helps you protect access to your applications, services, and IT resources. You can easily rotate, manage, and retrieve database credentials, API keys, and other secrets throughout their lifecycle.

For that, go to AWS Secrets Manager > Click on "Store a new secret".

1. In **Secret type**, select "Other type of secret". In **Key/value pairs**, select "Plaintext" and add your API key in the format shown below:

```
{
    "ipgeolocation-api-key":"your API key from https://app.ipgeolocation.io/"
}
```

Then, in **Encryptio key** select "aws/secretsmanager" and click on Next.

2. Next, **Configure secret**. Provide an appropriate Secret name, Description, and add an appropriate Name tag.
Then click on Next.

3. Next, we have to configure rotation. It's an op[tional setting and do it if required.

4. Review the secret and click on "store" to complete the creation.

Now, we can change the command to create docker container in api-service-instance1 accrodingly. The command will look like:

```
docker container run \ 
-d \ 
-p 8081:8080 \ 
--name ipgeolocation-api-service \ 
--restart always \ 
-e REDIS_PORT="6379" \ 
-e REDIS_HOST="redis.ipgeolocation.local" \ 
-e APP_PORT="8080" \ 
-e API_KEY_FROM_SECRETSMANAGER="True" \ 
-e SECRET_NAME="ipgeolocation-secret" \ 
-e SECRET_KEY="ipgeolocation-api-key" \ 
-e REGION="ap-south-1" \ 
fujikomalan/ipgeolocation-api-service:latest
```

> `SECRET_NAME` is the name provided to the secret that I have created in AWS secrets Manager. The `SECRET_KEY` can be obtained from **Secret value** section in the secret that we have created.

***Even if we create the container, it won't work properly, because in order to fetch the key value, the container should have access to the Secrets manager, to access the secrets manager. In order to solve the issue, we will create an IAM role and attach it to api-service-instance1 and api-service-instance2.***

So, create an IAM role and attach **SecretsManagerReadWrite** policy to it. After that, attach the IAM role to the instances api-service-instance1 and api-service-instance2. 

We can attach an existing IAM role to an instance by folowing the below steps:


>1. Select the instance, choose **Actions**, **Security**, **Modify IAM role**.
>2. Select the IAM role to attach to your instance, and choose **Save**.

Now we can proceed to create the docker containers 
 












