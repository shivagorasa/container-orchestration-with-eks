# Container orchestration using kubernetes and route application using ingress controller
## Description
Deploy two different web applications in kubernetes and create a single route
using K8s Ingress controller.
## Infrastructure setup
Setup Infrastructure in AWS using terraform to create EC2 using Windows
command prompt.
## Kubernetes setup
Create K8s cluster using EKS command line interface (1 Master and 3 Worker
nodes).
## Deploy Application
Deploy 2 different applications using deployment.yml files (Create 2 different yml
files and apply using K8s command line interface).
## Setup Ingress Route
Setup Ingress controller in K8s cluster and map local ip into some domain name
and use that domain name to route 2 application.
Ex: 1st application >> www.example.com
2nd application >> www.example.com/2nd
