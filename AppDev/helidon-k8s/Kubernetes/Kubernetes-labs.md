[Go to Overview Page](../README.md)

![](../../../common/images/customer.logo2.png)

# Migration of Monolith to Cloud Native

## C. Deploying to Kubernetes 

<details><summary><b>Self guided student - video introduction</b></summary>
<p>

This video is an introduction to the Kubernetes labs. Once you've watched it please press the "Back" button on your browser to return to the labs.

[![Kubernetes labs Introduction Video](https://img.youtube.com/vi/6Kg-zH6h3Is/0.jpg)](https://youtu.be/6Kg-zH6h3Is "Kubernetes labs introduction video")

</p>
</details>

---

## Introduction

In this series of labs we will focus on the specific features of Kubernetes to run Microservices.  These labs assume you have previously executed the **A. Helidon** and **B. Docker** part of this lab series.  In case you would want to execute *only this part C. Kubernetes of the labs,* you need to perform some [initial steps](../ManualSetup/KubernetesSetup.md) to prepare your environment.

## The Labs

### 1. Basic Kubernetes
This section covers how to run the docker images in kubenetes, how to use kubernetes secrets to hold configuration and access information, how to use an ingress to expose your application on a web port. Basically this covers how to make your docker based services run in in a kubernetes cluster.

We also look at using Helm to install kubernetes "infractructure" such as the ingress server

[The basic kubernetes labs](base-kubernetes/KubernetesBaseLabs.md)



### 2. Monitoring services -  Prometheus for data gathering
Once a service is running in Kubernetes we want to start seeing how well it's working in terms of the load on the service. At a basic level this is CPU / IO's but more interesting are things like the number of requests being serviced.

Monitoring metrics may also help us determining things like how changes when releasing a new version of the service may effect it's operation, for example does adding a database index increase the services efficiency by reducing lookup times, or increase it by adding extra work when updating the data. With this information you can determine if a change is worthwhile keeping.

[Prometheus lab](monitoring-kubernetes/MonitoringWithPrometheusLab.md)

### 3. Monitoring services - Grafana for data display
As you've seen Prometheus is great at capturing the data, but it's not the worlds best tool for displaying the data. Fortunately for us there is an open source tool called **Grafana** which is way better than Prometheus at this.

The process for installing and using Grafana is detailed in the next lab :  
[Visualising with Grafana lab document.](monitoring-kubernetes/VisualizingWithGrafanaLab.md)






## Cloud Native with Kubernetes


### 4. Is it running, and what to do if it isn't

Kubernetes doesn't just provide a platform to run containers in, it also provides a base for many other things including a comprehensive service availability framework which handles monitoring containers and services to see if they are still running, are still alive and are capable of responding to requests.

To understand how this works see the [Health Readiness Liveness labs.](cloud-native-labs/Health-readiness-liveness/Health-liveness-readiness.md)



### 5. Horizontal and Auto Scaling

Kubernetes also supports horizontal scaling of services, enabling multiple instances of a service to run with the load being shared amongst all of them. 

This first scaling lab shows how you can manually control the number of instances.

[The horizontal scaling labs](cloud-native-labs/Horizontal-scaling/Horizontal-scaling.md) 

Horizontal scaling provides you with a manual process to control how many instances of a microservice you have running, but Kubernetes also offers a mechanism to automatically change the number of instances.

This second scaling labs shows how you can have Kubernetes automatically scale the number of instances for you.

[The auto scaling labs](cloud-native-labs/Horizontal-scaling/Auto-scaling.md)


### 6. Rolling out deployment updates

Commonly when a service is deployed it will be updated, Kubernetes provides support for performing rolling upgrades, ensuring that the service continues running during the upgrade. Built into this are easy ways to reverse a deployment roll out to one of it's previous states.

[Rolling updates labs](cloud-native-labs/Rolling-updates/Rolling-updates.md)




### 7. Sections still to be written in Kubernetes cloud native

The following sections have not yet been written, they will be added in time.

#### Automatic CI/CD and Kubenetes

Integration of DevCS pipelines into the lab using DevCS as the build engine.

#### Automatic A/B testing

Automated testing of different versions of a microservice to see which is most effective.

#### Services Meshes.

The service mesh is one of the latest ideas to come out of the cloud native forum. It's basically a layer that sits on top of the various services in your Kubenetes environment. It can provide multiple capabilities, including ensuring that those communications re secure (service meshes can manage cross service encryption) but also implementing policies within your service communications, for example rate limiting to prevent a wayward services using all of the resources of another service, thus starving other clients. One very useful feature is the ability to do things like identify test traffic based on the user, and to route the test traffic to a different version of a service. Thus enabling the testing of a new version of a microservice within the full operational environment, but with the normal user traffic still going to the production versions of a service.







**Further Information**
For links to useful web pages and other information that I found while writing these labs [see this link](further-information/further-information.md)



## End of this tutorial

Congratulations, you have reached the end of the tutorial !  You are now ready to start refactoring your own applications with the techniques you learned during this session !



------

[Go to Overview Page](../README.md)