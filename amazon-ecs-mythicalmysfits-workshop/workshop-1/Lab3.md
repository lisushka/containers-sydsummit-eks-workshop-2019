# Lab 3: Incrementally build and deploy each microservice using EKS

It's time to break apart the monolithic adoption into microservices. To help with this, let's see how the monolith works in more detail.

> The monolith serves up several different API resources on different routes to fetch info about Mysfits, "like" them, or adopt them.
>
> The logic for these resources generally consists of some "processing" (like ensuring that the user is allowed to take a particular action, that a Mysfit is eligible for adoption, etc) and some interaction with the persistence layer, which in this case is DynamoDB.

> It is often a bad idea to have many different services talking directly to a single database (adding indexes and doing data migrations is hard enough with just one application), so rather than split off all of the logic of a given resource into a separate service, we'll start by moving only the "processing" business logic into a separate service and continue to use the monolith as a facade in front of the database. This is sometimes described as the [Strangler Application pattern](https://www.martinfowler.com/bliki/StranglerApplication.html), as we're "strangling" the monolith out of the picture and only continuing to use it for the parts that are toughest to move out until it can be fully replaced.

> The ALB has another feature called [path-based routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#path-conditions), which routes traffic based on URL path to particular target groups.  This means you will only need a single instance of the ALB to host your microservices.  The monolith service will receive all traffic to the default path, '/'.  Adoption and like services will be '/adopt' and '/like', respectively.

Here's what you will be implementing:

![Lab 4](images/04-arch.png)

*Note: The green tasks denote the monolith and the orange tasks denote the "like" microservice*

    
As with the monolith, you'll be using [EKS](https://aws.amazon.com/eks/) to deploy these microservices, but this time we'll walk through all the deployment steps for a fresh service.

### Instructions:

1. First, we need to add some glue code in the monolith to support moving the "like" function into a separate service. You'll use your Cloud9 environment to do this.  If you've closed the tab, go to the [Cloud9 Dashboard](https://console.aws.amazon.com/cloud9/home) and find your environment. Click "**Open IDE**". Find the `app/monolith-service/service/mythicalMysfitsService.py` source file, and uncomment the following section:

    ```
    # @app.route("/mysfits/<mysfit_id>/fulfill-like", methods=['POST'])
    # def fulfillLikeMysfit(mysfit_id):
    #     serviceResponse = mysfitsTableClient.likeMysfit(mysfit_id)
    #     flaskResponse = Response(serviceResponse)
    #     flaskResponse.headers["Content-Type"] = "application/json"
    #     return flaskResponse
    ```

    This provides an endpoint that can still manage persistence to DynamoDB, but omits the "business logic" (okay, in this case it's just a print statement, but in real life it could involve permissions checks or other nontrivial processing) handled by the `process_like_request` function.

2. With this new functionality added to the monolith, rebuild the monolith docker image with a new tag, such as `nolike`, and push it to ECR just as before (It is a best practice to avoid the `latest` tag, which can be ambiguous. Instead choose a unique, descriptive name, or even better user a Git SHA and/or build ID):

    <pre>
    $ cd app/monolith-service
    $ docker build -t monolith-service:nolike .
    $ docker tag monolith-service:nolike <b><i>ECR_REPOSITORY_URI</i></b>:nolike
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:nolike
    </pre>

3. Now, build the like service and push it to ECR.

    To find the like-service ECR repo URI, navigate to [Repositories](https://console.aws.amazon.com/ecs/home#/repositories) in the ECS dashboard, and find the repo named like <code><b><i>STACK_NAME</i></b>-like-XXX</code>.  Click on the like-service repository and copy the repository URI.

    ![Getting Like Service Repo](images/04-ecr-like.png)

    *Note: Your URI will be unique.*

    <pre>
    $ cd app/like-service
    $ docker build -t like-service .
    $ docker tag like-service:latest <b><i>ECR_REPOSITORY_URI</i></b>:latest
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:latest
    </pre>

4. Navigate to Kubernetes/micro folder. Now, just as in Lab 2, create a new revision of the kubernetes object  (this time pointing to the "nolike" version of the container image), AND update the monolith service to use this revision. Call this object nolike-app.yaml (there should be a sample file in the folder)

5. Before we deploy this microservice, we'll go into the details of setting up the [ALB Ingress Controller](https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/). 

 Kubernetes Ingress is an api object that allows you manage external (or) internal HTTP[s] access to Kubernetes services running in a cluster. Amazon Elastic Load Balancing Application Load Balancer (ALB) is a popular AWS service that load balances incoming traffic at the application layer (layer 7) across multiple targets, such as Amazon EC2 instances, in a region. ALB supports multiple features including host or path based routing, TLS (Transport layer security) termination, WebSockets, HTTP/2, AWS WAF (web application firewall) integration, integrated access logs, and health checks.

The AWS ALB Ingress controller is a controller that triggers the creation of an ALB and the necessary supporting AWS resources whenever a Kubernetes user declares an Ingress resource on the cluster. The Ingress resource uses the ALB to route HTTP[s] traffic to different endpoints within the cluster. The AWS ALB Ingress controller works on any Kubernetes cluster including Amazon Elastic Container Service for Kubernetes (EKS). For details on how the Kubernetes Ingress works with aws-alb-ingress-controller, please [read](https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/) this link. 

[alb-ingress](images/ALBIngress.png)



7. Once the new like service is deployed, test liking a Mysfit again by visiting the website. Check the CloudWatch logs again and make sure that the like service now shows a "Like processed." message. If you see this, you have succesfully factored out like functionality into the new microservice!

8. If you have time, you can now remove the old like endpoint from the monolith now that it is no longer seeing production use.

    Go back to your Cloud9 environment where you built the monolith and like service container images.

    In the monolith folder, open mythicalMysfitsService.py in the Cloud9 editor and find the code that reads:

    ```
    # increment the number of likes for the provided mysfit.
    @app.route("/mysfits/<mysfit_id>/like", methods=['POST'])
    def likeMysfit(mysfit_id):
        serviceResponse = mysfitsTableClient.likeMysfit(mysfit_id)
        process_like_request()
        flaskResponse = Response(serviceResponse)
        flaskResponse.headers["Content-Type"] = "application/json"
        return flaskResponse
    ```
    Once you find that line, you can delete it or comment it out.

    *Tip: if you're not familiar with Python, you can comment out a line by adding a hash character, "#", at the beginning of the line.*

9. Build, tag and push the monolith image to the monolith ECR repository.

    Use the tag `nolike2` now instead of `nolike`.

    <pre>
    $ docker build -t monolith-service:nolike2 .
    $ docker tag monolith-service:nolike <b><i>ECR_REPOSITORY_URI</i></b>:nolike2
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:nolike2
    </pre>

    If you look at the monolith repository in ECR, you'll see the pushed image tagged as `nolike2`:

    ![ECR nolike image](images/04-ecr-nolike2.png)

10. Now make one last deployment for the monolith to refer to this new container image URI (this process should be familiar now, and you can probably see that it makes sense to leave this drudgery to a CI/CD service in production), update the monolith service to use the new deployment, and make sure the app still functions as before.

### Checkpoint:
Congratulations, you've successfully rolled out the like microservice from the monolith.  If you have time, try repeating this lab to break out the adoption microservice.  Otherwise, please remember to follow the steps below in the **Workshop Cleanup** to make sure all assets created during the workshop are removed so you do not see unexpected charges after today.

Go [back](README.md)