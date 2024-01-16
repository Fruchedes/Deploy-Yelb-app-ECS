
#### Why Yelb 

When experimenting with new technologies you often follow tutorials that let you deploy purpose-built simple demo application. With Yelb I wanted to make this experience the other way around: start with an existing application and adapt it to the various technologies, services and platforms you are learning. I found this to be way more fun and useful. 


1. User's Journey:A user accesses the Yelb application through a web browser.
2. Request Forwarded to Load Balancer (In the Case of Load Balancer Deployment): If the application is deployed with a load balancer (e.g., in a containerized environment or using AWS Elastic Load Balancing), the user's request is initially directed to the load balancer.
3. Load Balancer Routing: The load balancer distributes incoming requests across multiple instances of the yelb-ui component.
The load balancer ensures even distribution of traffic for scalability and fault tolerance.
4. Front-end (yelb-ui) Handling: The load balancer forwards the user's request to one of the instances running the yelb-ui component.
The yelb-ui component serves JS code to the user's browser.
The code is compiled from an Angular 2 application, providing a dynamic user interface.
5. Front-end Interaction with Sinatra (yelb-appserver): When a user interacts with the UI (e.g., voting for a restaurant), the yelb-ui component communicates with the back-end logic layer (yelb-appserver).
6. Communication with Sinatra (yelb-appserver): The yelb-ui sends HTTP requests to the yelb-appserver API.
The API is implemented using Sinatra, a Ruby web framework.
The yelb-appserver processes requests, updates pie charts based on votes, and interacts with the Redis cache and Postgres database.
7. Interaction with Cache (Redis): For page views, the yelb-appserver interacts with a Redis cache to store and retrieve the number of page views.
Redis is used for its fast in-memory storage capabilities.
8. Interaction with Database (Postgres or DynamoDB): For persisting votes, the yelb-appserver interacts with a Postgres database in the traditional deployment model.
In the serverless deployment model, DynamoDB is used as a replacement for both Redis and Postgres.
9. Response to User: The yelb-appserver processes the user's request, updates data in the cache and database, and sends a response back to the yelb-ui.
The yelb-ui dynamically updates based on the response, reflecting changes in the UI.
10. User Experience: The user experiences a dynamic application where votes are processed, pie charts are updated, and the page reflects real-time interactions.
Yelb can be deployed (potentially) on each of these abstractions from bare metal (EC2) all the way to Lambda (serverless). 


#### Yelb architecture

The current architecture layout is pretty straightforward. 

There is a front-end component called `yelb-ui` that is responsable for vending the JS code to the browser. This code is compiled from an Angular 2 application. Depending on the deployment model this code can be served from an instance (EC2), from a container (Docker, Kubernetes, ECS) or from an S3 bucket (serverless). The `yelb-ui` component may also include, depending on the deployment model, an `nginx` proxy. 

At the time of this writing the only application component that exists is `yelb-appserver`. This is a Sinatra application that basically read and write to a cache server (`redis-server`) as well as a Postgres backend database (`yelb-db`). Redis is used to store the number of page views whereas Postgres is used to persist the votes. Note that the `yelb-db` container image is nothing more than Postgres (10) customized to create the database schema the application relies on. In the `serverless` deployment, new system variables has been introduced to support a DynamoDB backend (as a replacement for both redis and postgres).  

The picture below shows the architecture of Yelb:

![yelb-architecture](images/yelb-architecture.png)

Note: the picture above is representative of the instance (EC2) based deployment model and the container based deployment model. The serverless deployment model is architecturally different.


#### What does Yelb do?

Yelb allows users to vote on a set of alternatives (restaurants) and dynamically updates pie charts based on number of votes received. In addition to that Yelb keeps track of number of page views as well as it prints the hostname of the `yelb-appserver` instance serving the API request upon a vote or a page refresh. This allows an individual to demo the application solo, or involving people (e.g. an audience during a presentation) asking them to interact by pointing their browser to the application (which will increase the page count) and voting their favorite restaurant.    

This is how the UI looks like at this point:

![yelb-ui](images/yelb-ui.png)

The nature of the app allows people to experiment with the statless `yelb-ui` and `yelb-appserver` components (think scaling out, blue/green depoyments, etc.) as well as experiment with the stateful `redis-server` and `yelb-db` backend components. 

#### How can I use it?

If you intend to use Yelb as a generic application with the ultimate goal of learning and playing with multiple platforms (from cloud instances, to containers through potentially serverless), then you should probably go straight into the `deployments` folder and specifically in the `platformdeployment` folder. This is where all the various configuration files for the various platforms are available. This folder will ideally be a continuous work in progress. 

If you intend to contribute, fork or understand anyway how the single components work I suggest you dig into each of them looking at the code (and the various dockerfiles) to learn how they work and how they are packaged. In addition to that you should look into the `deployments` folder how to start the application in development mode and test mode.  

#### Known issues and limitations

- There is a subtle issue when you browse the app for the first time where the browser console shows errors. This is likely due to problems with variables not initialized properly. This needs further investigation. The app works just fine and there is no evidence of the problem unless you inspect the page and go to the console outputs.   
- There is a ton of optimization that could be achieved both in the application code as well as in the packaging constructs (e.g. Dockerfiles). The package in its current form works but it is far from being considered optmized. All in all, readability is prioritized over optmization. 
- Idempotency may be a problem. While I am trying to explicitly declare container tags and modules versions moving forward, there is a chance that if you try to build your own containers from the Dockerfile provided and/or install the project on a cloud instance, the output may be diverging from one setup to another. Generic commands such as `apt-get update`, `npm install` and `gem install sinatra` may lead to different versions of code and modules being pulled at build and setup times.
- The cache instance is still named `redis-server` (this will be renamed `yelb-cache` in the future for consistency)
- While the intent was to build an application whose layers could scale out (and scale independently) in a classic microservices scenario, very little testing has gone into scale out deployment scenarios so far. Most of the tests have been done with one (1) instance of service per layer.


##### Userdata script
#!/bin/bash
sudo yum update -y
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo yum install git -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
curl -L https://raw.githubusercontent.com/docker/compose-cli/main/scripts/install/install_linux.sh | sh
sudo usermod -a -G docker ec2-user
sudo service docker start
sudo chkconfig docker on
