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

After the creation of Redis cluster, go to [Route53](https://console.aws.amazon.com/route53/) and create a private hosted zone. I'm naming my private zone "ipgeolocation.local". Select the region according to your requirement, provide an appropriate tag and click on Create hosted zone. 

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

Then, in **Encryption key** select "aws/secretsmanager" and click on Next.

2. Next, **Configure secret**. Provide an appropriate Secret name, Description, and add an appropriate Name tag.
Then click on Next.

3. Next, we have to configure rotation. It's an optional setting and do it if required.

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

Now we can proceed to create the docker containers in the servers api-service-instance1 and api-service-instance2.
 
```
docker container run \ 
-d \ 
-p 8081:8080 \ 
--name ipgeolocation-api-service1 \ 
--restart always \ 
-e REDIS_PORT="6379" \ 
-e REDIS_HOST="redis.ipgeolocation.local" \ 
-e APP_PORT="8080" \ 
-e API_KEY_FROM_SECRETSMANAGER="True" \ 
-e SECRET_NAME="ipgeolocation-secret" \ 
-e SECRET_KEY="ipgeolocation-api-key" \ 
-e REGION_NAME="ap-south-1" \ 
fujikomalan/ipgeolocation-api-service:latest
```
and

```
docker container run \ 
-d \ 
-p 8082:8080 \ 
--name ipgeolocation-api-service2 \ 
--restart always \ 
-e REDIS_PORT="6379" \ 
-e REDIS_HOST="redis.ipgeolocation.local" \ 
-e APP_PORT="8080" \ 
-e API_KEY_FROM_SECRETSMANAGER="True" \ 
-e SECRET_NAME="ipgeolocation-secret" \ 
-e SECRET_KEY="ipgeolocation-api-key" \ 
-e REGION_NAME="ap-south-1" \ 
fujikomalan/ipgeolocation-api-service:latest
```

Now we have created 2 containers in api-service-instance1.

```
docker container ls -a
```

Output:

```
CONTAINER ID   IMAGE                                          COMMAND            CREATED         STATUS         PORTS                                       NAMES
7a8a6a155461   fujikomalan/ipgeolocation-api-service:latest   "python3 app.py"   8 seconds ago   Up 7 seconds   0.0.0.0:8082->8080/tcp, :::8082->8080/tcp   ipgeolocation-api-service2
dfd6c1972aeb   fujikomalan/ipgeolocation-api-service:latest   "python3 app.py"   2 minutes ago   Up 2 minutes   0.0.0.0:8081->8080/tcp, :::8081->8080/tcp   ipgeolocation-api-service1
```

Now load `Public IPv4 DNS:8081/ip/8.8.8.8` and `Public IPv4 DNS:8082/ip/8.8.8.8`. We will receive an output as shown below.

![output1](https://github.com/anandu79/IP-geolocation-containerized-project-with-Application-Load-Balancer/blob/main/images/output1.jpg)

## Docker container creation in api-caching-instance2

SSH into api-caching-instance2 and use the exact same commands as api-caching-instance1 to create containers.

## Target group creation

Now we have to create a target group. 

1. On the navigation panel, under LOAD BALANCING, choose **Target Groups**.

2. Choose **Create target group**.

3. For **Choose a target type**, select Instances to register targets by instance ID.

4. For **Target group name**, type a name for the target group. Select HTTP protocol and set protocol as 80. Protocol version should be set as HTTP1.

5. In **health checks section**, go to "advanced healt checks" and set the success codes as 200-499.

6. Provide an appropriate name tag if needed.

7. Choose **Next**.

8. Register targets

When registering the targets, select api-service-instance1, change Ports for the selected instances to 8081, click on **Include as pending below**.
Again select api-service-instance1, change Ports for the selected instances to 8082, click on **Include as pending below**.

Follow the same procedure when registering targets for api-service-instance2.

9. Click on **Create target group**.

## Appliocation load balancer creation

Next, we havew to create an application load balancer using the target group that we have created.

To configure your load balancer and listener:

1. In the navigation pane, under Load Balancing, choose **Load Balancers**.

2. Choose **Create Load Balancer**.

3. Under Application Load Balancer, choose **Create**.

4. In **Basic configuration**, give a name for the load balancer

5. In **Network mapping** section, select the VPC that you used for your EC2 instances. For **Mappings**, select two or more Availability Zones and corresponding subnets.

6. For **Security groups**, select a security group. 

7. In **Listeners and routing**, select HTTPS protocol and the port number should be 443. In Default action, select the target group that we have created above.

8. In **Secure listener settings**, select an existing ACM certificate or request a new one. 

9. Then, Add a tag to categorize your load balancer.

10. Click on **create load balancer**.

Now the Load Balancer has been created.

###### Setting HTTP to HTTPS redirection

1. Go to the **load balancers section**, select the load balancer that we have created, and click on **listeners**. Select **Add listener**.

2. In **Listener details**, select HTTP protocol and the port number will be 80. 

3. In **Default actions**, click on add actions dropdown and select **Redirect**. select HTTPS protocol and the port number should be 443. 

4. Provide a **Tag** if needed.

5. Click on **Add**.

## Pointing public Domain name to the Application Load Balancer

1. Go to [Route53](https://console.aws.amazon.com/route53/) console in your AWS account.

2. In the navigation pane, choose Hosted zones.

3. Choose the name of the hosted zone that has the domain name that you want to use to route traffic to your load balancer.

4. Choose Create record.

5. Provide an appropriate **Record name**.

6. Turn on **Alias**.

7. In **Route traffic to**, **Choose Alias to Application and Classic Load Balancer** as endpointr, then choose the Region that the endpoint is from, Choose the load balancer that we have created.











