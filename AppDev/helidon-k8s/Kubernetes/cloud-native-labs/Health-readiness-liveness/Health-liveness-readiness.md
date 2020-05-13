[Go to Overview Page](../Kubernetes-labs.md)

![](../../../../../common/images/customer.logo2.png)

# Migration of Monolith to Cloud Native

## C. Deploying to Kubernetes
## 4. Health, Readiness and Liveness


<details><summary><b>Self guided student - video introduction</b></summary>
<p>

This video is an introduction to the Kubernetes health, readiness and liveness lab. Once you've watched it please press the "Back" button on your browser to return to the labs.

[![Kubernetes health, readiness and liveness lab Introduction Video](https://img.youtube.com/vi/z1dKR94TQOE/0.jpg)](https://youtu.be/z1dKR94TQOE "Kubernetes health, readiness and liveness lab introduction video")

</p>
</details>

---

### **Introduction**

Kubernetes provides a service that monitors the pods to see if they meet the requirements in terms of running, being responsive, and being able to process requests. 

A core feature of Kubernetes is the assumption that eventually for some reason or another something will happen that means a service cannot be provided, and the designers of Kubernetes made this a core understanding of how it operates. Kubernetes doesn't just set things up they way you request, but it also continuously monitors the state of the entire deployments so that if the system does not meet what was specified Kubernetes steps in and automatically tries to adjust things so it does !

These labs look at how that is achieved.

### Is the container running ?

As we've seen a service in Kubernetes is delivered by programs running in containers. The way a container operates is that it runs a single program, once that program exists then the container exits, and the pod is no longer providing the service. 


<details><summary><b>Getting the service IP address if you don't have it</b></summary>
<p>
If you haven't written it down, or have forgotten how to get the IP address of the ingress controller service you can do the following

- In the OCI Cloud Shell type the following
  - `kubectl get services -n ingress-nginx`
  
```
NAME                                          TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-nginx-ingress-controller        LoadBalancer   10.96.210.131   132.145.253.186   80:31021/TCP,443:32009/TCP   19m
ingress-nginx-nginx-ingress-default-backend   ClusterIP      10.96.67.181    <none>            80/TCP                       19m
```

The Column EXTERNAL-IP gives you the IP address, in this case the IP address for the ingress-controller load balancer is `132.145.253.186` ** but this of course will be different in your environment !**
</p></details>

First let's make sure that the service is running, (replace <ip address> with the external ip address of the ingress)

- In the OCI Cloud Shell
  - `curl -i -X GET -u jack:password http://<ip address>:80/store/stocklevel`

```
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 14:01:18 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

Lets look at the pods to check all is running fine:

-  `kubectl get pods` 

```
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-rl8wg   1/1     Running   0          92m
storefront-65bd698bb4-cq26l     1/1     Running   0          92m
zipkin-88c48d8b9-jkcvq          1/1     Running   0          92m
```

We can see the state of our pods, look at the RESTARTS column, all of the values are 0 (if there was a problem, for example Kubernetes could not download the container images you would see a message in the Status column, and possibly additional re-starts, but here everything is A-OK.)

We're going to simulate a crash in our program, this will cause the container to exit, and Kubernetes will identify this and start a replacement container in the pod for us.

Using the name of the storefront pod above let's connect to the container in the pod using kubectl:

-  `kubectl exec storefront-65bd698bb4-cq26l -ti -- /bin/bash` 

Simulate a major fault that causes a service failure by killing the process running our service :

  - `kill -1 1`

```
root@storefront-65bd698bb4-cq26l:/# command terminated with exit code 137
```

<details><summary><b>How do you know it's process 1 ?</b></summary>
<p>
To be honest this is a bit of inside knowledge, docker images run the command they are given as process 1. The Graalvm image is pretty restricted in the commands it contains and sadly does not include the `ps` command, so sadly we can't check this.
</p></details>

Within a second or two of the process being killed the connection to the container in the pod is terminated as the container exits.

If we now try getting the data again it still responds  (replace the IP address with the one for your service)

- `curl -i -k -X GET -u jack:password https://987.123.456.789/store/stocklevel`

```
HTTP/2 200 
server: nginx/1.17.8
date: Fri, 27 Mar 2020 10:08:15 GMT
content-type: application/json
content-length: 220
strict-transport-security: max-age=15724800; includeSubDomains

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

- Let's look at the pod details again:
  -  `kubectl get pods`

```
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-rl8wg   1/1     Running   0          107m
storefront-65bd698bb4-cq26l     1/1     Running   1          107m
zipkin-88c48d8b9-jkcvq          1/1     Running   0          107m
```

Now we can see that the pod names are still the same, but the storefront pod has had a restart.

Kubernetes has identified that the container exited and within the pod restarted a new container. Another indication of this is if we look at the logs we can see that previous activity is no longer displaying:

-  `kubectl logs storefront-65bd698bb4-cq26l `

```
2020.01.02 14:06:30 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Starting server
2020.01.02 14:06:32 INFO org.jboss.weld.Version Thread[main,5,main]: WELD-000900: 3.1.1 (Final)
...
2020.01.02 14:10:02 INFO com.oracle.labs.helidon.storefront.resources.StorefrontResource Thread[hystrix-io.helidon.microprofile.faulttolerance-1,5,server]: Requesting listing of all stock
2020.01.02 14:10:04 INFO com.oracle.labs.helidon.storefront.resources.StorefrontResource Thread[hystrix-io.helidon.microprofile.faulttolerance-1,5,server]: Found 5 items
```

The reason it took a bit longer than usual when accessing the service is that the code was doing the on-demand setup of the web services.

### Liveness
We now have mechanisms in place to restart a container if it fails, but it may be that the container does not actually fail, just that the program running in it ceases to behave properly, for example there is some kind of non fatal resource starvation such as a deadlock. In this case the pod cannot recognize the problem as the container is still running.

Fortunately Kubernetes provides a mechanism to handle this as well. This mechanism is called **Liveness probes**, if a pod fails a liveness probe then it will be automatically restarted.

You may recall in the Helidon labs (if you did them) we created a liveness probe, this is an example of Helidon is designed to work in cloud native environments.

- Navigate to the **helidon-kubernetes** folder
- Open the file **storefront-deployment.yaml**
- Search for the Liveness probes section. This is under the spec.template.spec.containers section

```
        resources:
          limits:
            # Set this to me a whole CPU for now
            cpu: "1000m"
#        # Use this to check if the pod is alive
#        livenessProbe:
#          #Simple check to see if the liveness call works
#          # If must return a 200 - 399 http status code
#          httpGet:
#             path: /health/live
#             port: health-port
#          # Give it a few seconds to make sure it's had a chance to start up
#          initialDelaySeconds: 60
#          # Let it have a 5 second timeout to wait for a response
#          timeoutSeconds: 5
#          # Check every 5 seconds (default is 1)
#          periodSeconds: 5
#          # Need to have 3 failures before we decide the pod is dead, not just slow
#          failureThreshold: 3
        # This checks if the pod is ready to process requests
#        readinessProbe:
```

As you can see this section has been commented out. 

- On each line remove the # (only the first one, and only the # character, be sure not to remove any whitespace.) 
- Save the file

The resulting section should look like this:

```
        resources:
          limits:
            # Set this to me a whole CPU for now
            cpu: "1000m"
        # Use this to check if the pod is alive
        livenessProbe:
          #Simple check to see if the liveness call works
          # If must return a 200 - 399 http status code
          httpGet:
             path: /health/live
             port: health-port
          # Give it a few seconds to make sure it's had a chance to start up
          initialDelaySeconds: 60
          # Let it have a 5 second timeout to wait for a response
          timeoutSeconds: 5
          # Check every 5 seconds (default is 1)
          periodSeconds: 5
          # Need to have 3 failures before we decide the pod is dead, not just slow
          failureThreshold: 3
        # This checks if the pod is ready to process requests
#        readinessProbe:
```

This is a pretty simple test to see if there is a running service, in this case we use the service to make an http get request (this is made by the framework and is done from outside the pod) on the 9080:/health/live url (we know it's on port 9080 as the port definition names it health-port.) There are other types of liveness probe than just get requests, you can run a command in the container itself, or just see if it's possible to open a tcp/ip connection to a port in the container. Of course this is a simple definition, it doesn't look at the many options that are available.

The first thing to say is that whatever steps your actual liveness test does it needs to be sufficient to detect service problems like deadlocks, but also to use as few resources as possible so the check itself doesn't become a major load factor.

Let's look at some of these values.

As it may take a while to start up the container, we specify and initialDelaySeconds of 60, Kubernetes won't start checking if the pod is live until that period is elapsed. If we made that to short then we may never start the container as Kubernetes would always determine it was not alive before the container had a chance to start up properly. 

The parameter **timeoutSeconds** specifies that for the http request  to have failed it could not have responded in 5 seconds. As many http service implementations are initialized on first access we need to chose a value that is long enough for the framework to do it's lazy initialization.

The parameter **periodSeconds** defines how often Kubernetes will check the container to see if it's alive and responding. This is a balance, especially if the liveness check involved significant resources (e.g. making a RDBMS call) You need to check often enough that a non responding container will be detected quickly, but not check so often that the checking process itself uses to many resources.

Finally **failureThreshold** specifies how many consecutive failures are needed before it's deemed to have failed, in this case we need 3 failures to respond

Whatever your actual implementation you need to carefully consider the values above. Get them wrong and your service may never be allowed to start, or problems may not be detected.

Let's apply the changes we made in the deployment :

- Make sure you are in the folder **helidon-kubernetes**

-  `bash undeploy.sh`

```
Deleting storefront deployment
deployment.extensions "storefront" deleted
Deleting stockmanager deployment
deployment.extensions "stockmanager" deleted
Deleting zipkin deployment
deployment.extensions "zipkin" deleted
Kubenetes config is
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-j6vf9   1/1     Running   0          66m
pod/storefront-dcc76cccb-6ztsf      1/1     Running   1          66m
pod/zipkin-88c48d8b9-7rx6q          1/1     Running   0          66m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   3h52m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   3h52m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            3h52m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       66m
replicaset.apps/storefront-dcc76cccb      1         1         1       66m
replicaset.apps/zipkin-88c48d8b9          1         1         1       66m

```

- Deploy the updated version : `bash deploy.sh`

```
Creating zipkin deployment
deployment.extensions/zipkin created
Creating stockmanager deployment
deployment.extensions/stockmanager created
Creating storefront deployment
deployment.extensions/storefront created
Kubenetes config is
NAME                                READY   STATUS              RESTARTS   AGE
pod/stockmanager-6456cfd8b6-29lmk   0/1     ContainerCreating   0          0s
pod/storefront-b44457b4d-29jr7      0/1     Pending             0          0s
pod/zipkin-88c48d8b9-bftvx          0/1     ContainerCreating   0          0s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   3h55m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   3h55m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            3h55m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   0/1     1            0           0s
deployment.apps/storefront     0/1     1            0           0s
deployment.apps/zipkin         0/1     1            0           0s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         0       0s
replicaset.apps/storefront-b44457b4d      1         1         0       0s
replicaset.apps/zipkin-88c48d8b9          1         1         0       0s

```

- Let's see how our pod is doing.
  -  `kubectl get pods`

```
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-29lmk   1/1     Running   0          24s
storefront-b44457b4d-29jr7      1/1     Running   0          24s
zipkin-88c48d8b9-bftvx          1/1     Running   0          24s
```

Note that as we have undeployed and then deployed again there are new pods and so the RESTART count is back to zero.

If we look at the logs for the storefront **before** the liveness probe has started (so before the 60 seconds from container creation) we see that it starts as we expect it to. 

- Visualize the logs :  `kubectl logs storefront-b44457b4d-29jr7`

```
2020.01.02 16:18:58 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Starting server
...
2020.01.02 16:19:07 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Running on http://localhost:8080/store
```

If however the 60 seconds has passed and the liveness call has started we will see calls being made to the status resource,

- Run the kubectl command again: `kubectl logs storefront-b44457b4d-29jr7 `

You will see multiple entries like the one below:

```
2020.01.02 16:21:11 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
```

- Look at the pods detailed info to check the state is fine :
  -  `kubectl describe pod storefront-b44457b4d-29jr7 `

```
...
Events:
  Type    Reason     Age    From                     Message
  ----    ------     ----   ----                     -------
  Normal  Scheduled  4m30s  default-scheduler        Successfully assigned tg-helidon/storefront-b44457b4d-29jr7 to docker-desktop
  Normal  Pulling    4m29s  kubelet, docker-desktop  Pulling image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal  Pulled     4m28s  kubelet, docker-desktop  Successfully pulled image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal  Created    4m28s  kubelet, docker-desktop  Created container storefront
  Normal  Started    4m28s  kubelet, docker-desktop  Started container storefront
```

It's started and no unexpected events !

Now is the time to explain that "Not frozen ..." text in the status. To enable us to actually simulate the service having a deadlock or resource starvation problem there's a bit of a cheat in the storefront LivenessChecker code :

```
	@Override
	public HealthCheckResponse call() {
		// don't test anything here, we're just reporting that we are running, not that
		// any of the underlying connections to thinks like the database are active

		// if there is a file /frozen then just lock up for 60 seconds
		// this let's us emulate a lockup which will trigger a pod restart
		// if we have enabled liveliness testing against this API
		if (new File("/frozen").exists()) {
			log.info("/frozen exists, locking for " + FROZEN_TIME + " seconds");
			try {
				Thread.sleep(FROZEN_TIME * 1000);
			} catch (InterruptedException e) {
				// ignore for now
			}
			return HealthCheckResponse.named("storefront-live").up()
					.withData("uptime", System.currentTimeMillis() - startTime).withData("storename", storeName)
					.withData("frozen", true).build();
		} else {
			log.info("Not frozen, Returning alive status true, storename " + storeName);
			return HealthCheckResponse.named("storefront-live").up()
					.withData("uptime", System.currentTimeMillis() - startTime).withData("storename", storeName)
					.withData("frozen", false).build();
		}
```

Every time it's called it checks to see it a file names /frozen exists in the root directory of the container. If it does then it will do a delay (about 60 seconds) before returning the response. Basically this means that by connecting to the container and creating the /frozen file we can simulate the container having a problem. The `Not Frozen...` is just text in the log data so we can see what's happening. Of course you wouldn't do this in a production system !

Let's see what happens in this case.

First let's start following the logs of your pod

- Run the following command (replace the pod Id with yours)
  -  `kubectl logs -f --tail=10 storefront-b44457b4d-29jr7 `


```
...
2020.01.02 16:24:36 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:24:41 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
```

- Open new browser window or tab
- Go to your cloud account
- Once in the cloud account open an OCI Cloud Shell in the new window

- Log in to the your container and create the /frozen file  (replace the pod Id with yours)
  -  `kubectl exec -ti storefront-b44457b4d-29jr7 -- /bin/bash`
  -  `touch /frozen`
- Go back to the window running the logs

Kubernetes detected that the liveness probes were not responding in time, and after 3 failures it restarted the pod.

In the logs we see the following 

```
2020.01.02 16:25:41 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:25:46 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:25:51 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:25:56 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: /frozen exists, locking for 60 seconds
Weld SE container 53fe34a2-0291-4b72-a00e-966bab7ab2ad shut down by shutdown hook
```

Kubectl tells us there's been a problem and a pod has done a restart for us

- Check the pod status: `kubectl get pods`

```
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-29lmk   1/1     Running   0          7m50s
storefront-b44457b4d-29jr7      1/1     Running   1          7m50s
zipkin-88c48d8b9-bftvx          1/1     Running   0          7m50s
```

- Look at the deployment events for the pod
  -  `kubectl describe pod storefront-b44457b4d-29jr7`

```
...
Events:
  Type     Reason     Age                  From                     Message
  ----     ------     ----                 ----                     -------
  Normal   Scheduled  8m13s                default-scheduler        Successfully assigned tg-helidon/storefront-b44457b4d-29jr7 to docker-desktop
  Warning  Unhealthy  57s (x3 over 67s)    kubelet, docker-desktop  Liveness probe failed: Get http://10.1.0.170:9080/health/live: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
  Normal   Killing    57s                  kubelet, docker-desktop  Container storefront failed liveness probe, will be restarted
  Normal   Pulling    56s (x2 over 8m12s)  kubelet, docker-desktop  Pulling image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal   Pulled     56s (x2 over 8m11s)  kubelet, docker-desktop  Successfully pulled image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal   Created    56s (x2 over 8m11s)  kubelet, docker-desktop  Created container storefront
  Normal   Started    55s (x2 over 8m11s)  kubelet, docker-desktop  Started container storefront
```

The pod became unhealthy, then the container was killed and a fresh new container restarted.

 (Leave the extra window open as you'll be using it again later)
### Readiness
The first two probes determine if a pod is alive and running, but it doesn't actually report if it's able to process events. That can be a problem if for example a pod has a problem connecting to a backend service, perhaps there is a network configuration issue and the pods path to a back end service like a database is not available.

In this situation restarting the pod / container won't do anything useful, it's not a problem with the container itself, but something outside the container, and hopefully once that problem is resolved the front end service will recover (it's it's been properly coded and doesn't crash, but in that case one of the other health mechanisms will kick in and restart it) **BUT** there is also no point in sending requests to that container as it can't process them.

Kubernetes supports a readiness probe that we can call to see is the container is ready. If the container is not ready then it's removed from the set of available containers that can provide the service, and any requests are routed to other containers that can provide the service. 

Unlike a liveness probe, if a container fails it's not killed off, and calls to the readiness probe continue to be made, if the probe starts reporting the service in the container is ready then it's added back to the list of containers that can deliver the servcie and requests will be routed to it once more.

- Make sure you are in the folder **helidon-kubernetes**
- Edit the file **storefront-deployment.yaml**

- Look for the section (just after the Liveness probe) where we define the **readiness probe**. 

```
#        readinessProbe:
#          exec:
#            command:
#            - /bin/bash
#            - -c
#            - 'curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"UP\""'
#          # No point in checking until it's been running for a while 
#          initialDelaySeconds: 15
#          # Allow a short delay for the response
#          timeoutSeconds: 5
#          # Check every 10 seconds
#          periodSeconds: 10
#          # Need at least only one fail for this to be a problem
#          failureThreshold: 1
```
- Remove the # (and only the #, not spaces or anything else) and save the file. 

The ReadinessProbe section should now look like this :

```
       # This checks if the pod is ready to process requests
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - 'curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"UP\""'
          # No point in checking until it's been running for a while 
          initialDelaySeconds: 15
          # Allow a short delay for the response
          timeoutSeconds: 5
          # Check every 10 seconds
          periodSeconds: 10
          # Need at least only one fail for this to be a problem
          failureThreshold: 1
```

The various options for readiness are similar to those for Liveliness, except you see here we've got an exec instead of httpGet.

The exec means that we are going to run code **inside** the pod to determine if the pod is ready to serve requests. The command section defined the command that will be run and the arguments. In this case we run the /bin/bash shell, -c means to use the arguments as a command (so it won't try and be interactive) and the 'curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"UP\""' is the command.

Some points about 'curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"UP\""'

Firstly this is a command string that actually runs several commands connecting the output of one to the input of the other. If you exec to the pod you can actually run these by hand if you like

The first (the curl) gets the readiness data from the service (you may remember this in the Helidon labs) In this case as the code is running within the pod itself, so it's a localhost connection and as it'd direct to the micro-service it's http

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready 
{"outcome":"UP","status":"UP","checks":[{"name":"storefront-ready","state":"UP","status":"UP","data":{"storename":"My Shop"}}]}
```

We can just use the final command the grep to look for a line containing `"outcome":"UP"` We do however have to be careful because " is actually part of what we want to look for (and if you ran the text thoguht a json formatter you may have space characters) So to define these as a constant we need to ensure it's a single string, we do this by enclosing the entire thing in quotes `"` and to prevent the `"` within the string being interpreted as end of string (and thus new argument) characters we need to escape them, hence we end up with `"\"outcome\":\"UP\""`

Now try it out:

- Connect to the pod : `kubectl exec -ti storefront-b44457b4d-29jr7  -- /bin/bash`
- Run the command in the pod:
  -  `curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"UP\""`

```
{"outcome":"UP","status":"UP","checks":[{"name":"storefront-ready","state":"UP","status":"UP","data":{"storename":"My Shop"}}]}
```

In this case the pod is ready, so the grep command returns what it's found. We are not actually concerned with what the pod returns in terms of string output, we are looking for the exit code, interactivly we can find that by looking in the $? variable:

-  `echo $?`

```
0
```

And can see that the variable value (which is what's returned back to the Kubernetes readiness infrastructure) is 0. In Unix / Linux terms this means success.

If you want to see what it would do if the outcome was not UP try running the command changing the UP to DOWN (or actually anything other than UP) **Important** While you can run this command in the pods shell •DO NOT• modify the yaml like this.

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"DOWN\""
root@storefront-b44457b4d-29jr7:/# echo $?
1
```

In this case the return value in $? is 1, not 0, and in Unix / Linux terms that means something's gone wrong.

The whole thing is held in single quotes `'curl -s http://localhost:9080/health/ready | grep "\"outcome\":\"UP\""'` to that it's treated as a single string by the yaml file parser and when being handed to the bash shell.

Remember, in this case the command is executed inside the container, so you have to make sure that the commands you want to run will be available. This is especially important if you change the base image, you might find you are relying on a command that's no longer there, or works in a different way from your expectations.

That's all we're going to do with bash shell programming for now !

Having made the changes let's undeploy the existing configuration and then deploy the new one

In the OCI Cloud Shell
- Navigate to the **helidon-kubernetes** folder
- Run the undeploy.sh script
  -  `bash undeploy.sh `

```
Deleting storefront deployment
deployment.extensions "storefront" deleted
Deleting stockmanager deployment
deployment.extensions "stockmanager" deleted
Deleting zipkin deployment
deployment.extensions "zipkin" deleted
Kubenetes config is
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-29lmk   1/1     Running   0          46m
pod/storefront-b44457b4d-29jr7      1/1     Running   1          46m
pod/zipkin-88c48d8b9-bftvx          1/1     Running   0          46m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h41m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h41m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h41m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       46m
replicaset.apps/storefront-b44457b4d      1         1         1       46m
replicaset.apps/zipkin-88c48d8b9          1         1         1       46m
```

As usual it takes a few seconds for the deployments to stop, this was about 30 seonds later

- Check only the services remain running : 
  -  `kubectl get all`

```
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h42m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h42m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h42m
```

Now let's deploy them again, run the deploy.sh script, be prepared to run kubectl get all within a few seconds of the deploy finishing.

- Run the deploy script :  `bash deploy.sh`

```
Creating zipkin deployment
deployment.extensions/zipkin created
Creating stockmanager deployment
deployment.extensions/stockmanager created
Creating storefront deployment
deployment.extensions/storefront created
Kubenetes config is
NAME                                READY   STATUS              RESTARTS   AGE
pod/stockmanager-6456cfd8b6-vqq7c   0/1     ContainerCreating   0          0s
pod/storefront-74cd999d8-dzl2n      0/1     Pending             0          0s
pod/zipkin-88c48d8b9-vdn47          0/1     ContainerCreating   0          0s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h42m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h42m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h42m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   0/1     1            0           0s
deployment.apps/storefront     0/1     1            0           0s
deployment.apps/zipkin         0/1     1            0           0s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         0       0s
replicaset.apps/storefront-74cd999d8      1         1         0       0s
replicaset.apps/zipkin-88c48d8b9          1         1         0       0s
```

- Immediately run the command `kubectl get all`

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-vqq7c   1/1     Running   0          12s
pod/storefront-74cd999d8-dzl2n      0/1     Running   0          12s
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          12s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h43m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h43m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h43m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           12s
deployment.apps/storefront     0/1     1            0           12s
deployment.apps/zipkin         1/1     1            1           12s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       12s
replicaset.apps/storefront-74cd999d8      1         1         0       12s
replicaset.apps/zipkin-88c48d8b9          1         1         1       12s
```

We see something different, usually within a few seconds of starting a pod it's in the Running state (as are these) **BUT** if we look at the pods we see that though it's running the storefront is not actually ready, the replica set doesn't have any storefront ready (even though we've asked for pod 1) and neither does the deployment. 

If after a minute or so we re-do the kubectl command it's as we expected.

```
$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-vqq7c   1/1     Running   0          95s
pod/storefront-74cd999d8-dzl2n      1/1     Running   0          95s
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          95s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h44m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h44m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h44m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           95s
deployment.apps/storefront     1/1     1            1           95s
deployment.apps/zipkin         1/1     1            1           95s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       95s
replicaset.apps/storefront-74cd999d8      1         1         1       95s
replicaset.apps/zipkin-88c48d8b9          1         1         1       95s
```

Now everything is ready, but why the delay ? What's caused it ? And why didn't it happen before ?

Well the answer is simple. If there is a readiness probe enabled a pod is not considered ready *until* the readiness probe reports success. Prior to that the pod is not in the set of pods that can deliver a service.

As to why it didn't happen before if a pod does not have a readiness probe specified then it is automatically assumed to be in a ready state as soon as it's running

What happens if a request is made to the service while before the pod is ready ? Well if there are other pods in the service (the selector for the service matches the labels and the pods are ready to respond) then the service requests are sent to those pods. If there are **no** pods ab.e to service the requests than a 503 "Service Temporarily Unavailable" is generated back to the caller

To see what happens if the readiness probe does not work we can simply undeploy the stock manager service.

- First let's check it's running fine  (replace the IP address with the one for your service)
  -  `curl -i -k -X GET -u jack:password https://987.123.456.789/store/stocklevel

```
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 17:30:32 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

- Now let's use kubectl to undeploy just the stockmanager service
  -  `kubectl delete -f stockmanager-deployment.yaml`

```
deployment.extensions "stockmanager" deleted
```
- Let's check the pods status
  -  `kubectl get pods`

```
NAME                            READY   STATUS        RESTARTS   AGE
stockmanager-6456cfd8b6-vqq7c   0/1     Terminating   0          26m
storefront-74cd999d8-dzl2n      1/1     Running       0          26m
zipkin-88c48d8b9-vdn47          1/1     Running       0          26m
```
The stock manager service is being stopped, after 60 seconds or so if we run kubectl again to get everything we see it's gone

-  `kubectl get all`

```
NAME                             READY   STATUS    RESTARTS   AGE
pod/storefront-74cd999d8-dzl2n   0/1     Running   0          28m
pod/zipkin-88c48d8b9-vdn47       1/1     Running   0          28m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   5h11m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   5h11m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            5h11m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/storefront   0/1     1            0           28m
deployment.apps/zipkin       1/1     1            1           28m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/storefront-74cd999d8   1         1         0       28m
replicaset.apps/zipkin-88c48d8b9       1         1         1       28m

```

Something else has also happened though, the storefront service has no pods in the ready state, neither does the storefront deployment and replica set. The readiness probe has run against the storefront pod and when the probe checked the results it found that the storefront pod was not in a position to operate, because the service it depended on (the stock manager) was no longer available. 

- Let's try accessing the service (replace the IP address with the one for your service)
  -  `curl -i -k -X GET -u jack:password https://987.123.456.789/store/stocklevel`

```
HTTP/1.1 503 Service Temporarily Unavailable
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 17:37:29 GMT
Content-Type: text/html
Content-Length: 203
Connection: keep-alive

<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>
```
The service is giving us a 503 Service Temporarily Unavailable message. Well to be precise this is coming from the Kubernetes as it can't find a storefront service that is in the ready state.

- Let's start the stockmager service using kubectl again
  -  `kubectl apply -f stockmanager-deployment.yaml`

```
deployment.extensions/stockmanager created
```

- Now let's see what's happening with our deployments 
  -  `kubectl get all`

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-4mpl2   1/1     Running   0          7s
pod/storefront-74cd999d8-dzl2n      0/1     Running   0          33m
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          33m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   5h16m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   5h16m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            5h16m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           7s
deployment.apps/storefront     0/1     1            0           33m
deployment.apps/zipkin         1/1     1            1           33m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       7s
replicaset.apps/storefront-74cd999d8      1         1         0       33m
replicaset.apps/zipkin-88c48d8b9          1         1         1       33m
```

The stockmanager is running, but the storefront is still not ready, and it won't be until the readiness check is called again and determines that it's ready to work.

- Looking at the kubectl output abut 90 seconds later:
  -  `kubectl get all`

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-4mpl2   1/1     Running   0          96s
pod/storefront-74cd999d8-dzl2n      1/1     Running   0          35m
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          35m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   5h18m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   5h18m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            5h18m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           96s
deployment.apps/storefront     1/1     1            1           35m
deployment.apps/zipkin         1/1     1            1           35m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       96s
replicaset.apps/storefront-74cd999d8      1         1         1       35m
replicaset.apps/zipkin-88c48d8b9          1         1         1       35m
```

The storefront readiness probe has kicked in and the services are all back in the ready state once again  (replace the IP address with the one for your service)

- Check the service : `curl -i -k -X GET -u jack:password https://987.123.456.789/store/stocklevel`

```
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 17:42:40 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

### Startup probes
You may have noticed above that we had to wait for the readiness probe to complete on a pod before it became ready, and worse we had to wait the intialDelaySeconds before we could sensibly even start testing to see if the pod was ready. This means that if we wanted to add extra pods then there is a delay before the new capacity come online and support the service, In the case of the storefront this is not to much of a problem as the service starts up fast, but for a more complex service, especially a legacy service that may have a startup time that varies a lot depending on other factors, this could be a problem, after all we want to respond to request as soon as we can.

To solve this in Kubernetes 1.16 the concept of startup probes was introduced. A startup probe is a very simple probe that tests to see if the service has started running, usually at a basic level, and then starts up the liveness and readiness probes. Effectively the startupProbe means there is no longer any need for the initialDelaySeconds.

Unfortunately however the startup probes are not supported in versions of Kubernetes prior to 1.16, and as our Kubernetes environment (and most other cloud providers) are not yet on that version we can't demo that. But there is example configuration in the storefront-deployment.yaml file to show how this would would be defined (the initialDeploymentSeconds would need to be removed from the liveness and readiness configurations.

Once we have a 1.16 or later production deployment this section of the lab will be updated to cover the startup probes in more detail.





---

You have reached the end of this lab !!

Use your **back** button to return to the **C. Deploying to Kubernetes** section