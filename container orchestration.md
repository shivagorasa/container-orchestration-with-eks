Container orchestration using kubernetes and route application using
ingress controller
Description
Deploy two different web applications in kubernetes and create a single route
using K8s Ingress controller.

# Prerequisites 

* A Kubernetes cluster above version 1.2.
* The kubectl command-line tool installed in your local environment and configured to connect to your cluster


# Infrastructure setup

Setup Infrastructure in AWS using terraform to create EC2 using Windows
command prompt.

Create a terraform script as follows with your aws credentials and create a file called main.tf with following code:

```
# Define provider
provider "aws" {
  region = "your_region"
}

# Create a security group
resource "aws_security_group" "instance_sg" {
  name        = "instance-sg"
  description = "Security group for the instance"
  
  # Inbound rule for SSH access
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Inbound rule for HTTP access
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Outbound rule allowing all traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Launch EC2 instance
resource "aws_instance" "example" {
  ami           = "ami-xxxxxxxxxxxxxxx"  # Specify your AMI ID
  instance_type = "t2.medium"
  key_name      = "your_keypair_name"
  security_groups = [aws_security_group.instance_sg.name]
  
  tags = {
    Name = "ec2-created from terrafrom"
  }
}

```

Configure your aws using aws configure and authenticate with your aws access key and secret key with necessary permissions

This will create following instance 

![!\[alt text\](image.png)](ec2_terraform.png)


# Kubernetes setup

Create K8s cluster using EKS command line interface (1 Master and 3 Worker
nodes).

To create a Kubernetes (K8s) cluster using the Amazon Elastic Kubernetes Service (EKS) command-line interface (CLI) with one master node and three worker nodes, you'll need to follow these steps:

* Install and Configure AWS CLI: Make sure you have the AWS CLI installed and configured with appropriate credentials.

* Install eksctl: eksctl is a CLI tool provided by AWS to create and manage EKS clusters. Install it using a package manager like Homebrew or by downloading the binary from the GitHub releases page.

* Create Cluster Configuration File: Create a YAML file defining your cluster configuration. For example, you can name it cluster.yaml:

* create a key in /ssh location 

```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/eks_rsa
```

* Use following to create cluster using eksctl


```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: your-cluster-name
  region: your-region

nodeGroups:
  - name: ng-1
    instanceType: your-instance-type
    desiredCapacity: 3
    minSize: 1
    maxSize: 4
    ssh:
      allow: true
      publicKeyPath: ~/.ssh/eks_rsa.pub

```
use ``` eksctl create cluster -f cluster.yaml``` to create cluster

![!\[alt text\](image.png)](<cluster creation.png>)

Following cluster is created in aws 

![!\[alt text\](image.png)](cluster.png)

then verify for instance 
![!\[alt text\](image.png)](instance-stakc.png)

then use ``` kubectl get nodes``` to verify cluster creation

![!\[alt text\](image.png)](nodes.png)

# Now to Deploy Application

Deploy 2 different applications using deployment.yml files (Create 2 different yml
files and apply using K8s command line interface).

Before you deploy the Nginx Ingress, you will deploy a “Hello World” app called hello-kubernetes to have some Services to which you’ll route the traffic. 

```nano hello-kubernetes-first.yaml```

Add the following lines:
```
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes-first
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-kubernetes-first
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes-first
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes-first
  template:
    metadata:
      labels:
        app: hello-kubernetes-first
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.10
        ports:
        - containerPort: 8080
        env:
        - name: MESSAGE
          value: Hello from the first deployment!
```

This configuration defines a Deployment and a Service. The Deployment consists of three replicas of the paulbouwer/hello-kubernetes:1.7 image and an environment variable named MESSAGE (you will see its value when you access the app). The Service here is defined to expose the Deployment in-cluster at port 80.

create this first variant of the hello-kubernetes app in Kubernetes by running the following command: ```kubectl create -f hello-kubernetes-first.yaml```

![!\[alt text\](image.png)](dep1.png)

To verify the Service’s creation, run the following command: ```kubectl get service hello-kubernetes-first```

You’ll find that the newly created Service has a ClusterIP assigned, which means that it is working properly.

![!\[alt text\](image.png)](svc.png)

Create a new file called hello-kubernetes-second.yaml:
```nano hello-kubernetes-second.yaml``

with following lines
```
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes-second
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-kubernetes-second
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes-second
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes-second
  template:
    metadata:
      labels:
        app: hello-kubernetes-second
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.10
        ports:
        - containerPort: 8080
        env:
        - name: MESSAGE
          value: Hello from the second deployment!
```
similar to above (first deployment) use : ```kubectl create -f hello-kubernetes-second.yaml``` and ```kubectl get service``` to verify 

![!\[alt text\](image.png)](service2.png)

Both hello-kubernetes-first and hello-kubernetes-second are listed, which means that Kubernetes has created them successfully.

# Setup Ingress Route

Setup Ingress controller in K8s cluster and map local ip into some domain name
and use that domain name to route 2 application.

Lets use nginx controller , install helm and then proceed to furthur 

To install the Nginx Ingress Controller to your cluster, you’ll first need to add its repository to Helm by running:
```helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx```

Update your system to let Helm know what it contains:
```helm repo update```

Finally, run the following command to install the Nginx ingress:
```helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true```

This command installs the Nginx Ingress Controller from the stable charts repository, names the Helm release nginx-ingress, and sets the publishService parameter to true.

Run this command to watch the Load Balancer become available:
```kubectl --namespace default get services -o wide -w nginx-ingress-ingress-nginx-controller``` 
This command fetches the Nginx Ingress service in the default namespace and outputs its information, but the command does not exit immediately. With the -w argument, it watches and refreshes the output when changes occur.

![!\[alt text\](image.png)](loadbalancer1.png)

After some time has passed, the IP address of your newly created Load Balancer will appear:

![!\[alt text\](image.png)](lboutput.png)

Next, you’ll need to ensure that your two domains are pointed to the Load Balancer via A records. This is done through your DNS provider. 

Map Local IP to Domain Name: You can achieve this by adding an entry in your local hosts file. On Unix-based systems, this file is typically located at /etc/hosts. Add a line like this:
```<local-ip> your-domain.com```

# Exposing the App Using an Ingress

We'll store the Ingress in a file named hello-kubernetes-ingress.yaml. Create it using your editor:
```nano hello-kubernetes-ingress.yaml```

Add the following lines to your file:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-kubernetes-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: "hw1.your_domain_name"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: hello-kubernetes-first
            port:
              number: 80
  - host: "hw2.your_domain_name"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: hello-kubernetes-second
            port:
              number: 80
```
We define an Ingress Resource with the name hello-kubernetes-ingress. Then, you specify two host rules so that ```hw1.your_domain``` is routed to the hello-kubernetes-first Service, and ```hw2.your_domain``` is routed to the Service from the second deployment (hello-kubernetes-second).

Create it in Kubernetes by running the following command:
```kubectl apply -f hello-kubernetes-ingress.yaml```

We can now navigate to hw1.your_domain in your browser. The first deployment will load:

![alt text](step3a.png)

The second variant (hw2.your_domain) will display a different message:

![alt text](step3b.png)

We have verified that the Ingress Controller correctly routes requests, in this case from your two domains to two different Services.

Submitted by ___ShivaKumar Gorasa___
