# Java (Spring Boot) + Istio on Kubernetes/OpenShift

There are three different and super simple microservices in this system and they are chained together in the following sequence:

customer -> preferences -> recommendations

For now, they have a simple exception handling solution for dealing with
a missing dependent service, it just returns the error message to the end-user.

There are two more simple apps that illustrate how Istio handles egress routes: egressgithub and egresshttpbin

**Table of Contents**



<!-- toc -->

* [Prerequisite CLI tools](#prerequisite-cli-tools)
* [Setup minishift](#setup-minishift)
* [Setup environment](#setup-environment)
* [Istio installation script](#istio-installation-script)
* [Deploy customer](#deploy-customer)
* [Deploy preferences](#deploy-preferences)
* [Deploy recommendations](#deploy-recommendations)
* [Updating & redeploying code](#updating-redeploying-code)
* [Tracing](#tracing)
* [Monitoring](#monitoring)
* [Istio RouteRule Changes](#istio-routerule-changes)
  * [recommendations:v2](#recommendationsv2)
* [Changing Istio RouteRules](#changing-istio-routerules)
    * [All users to recommendations:v2](#all-users-to-recommendationsv2)
    * [All users to recommendations:v1](#all-users-to-recommendationsv1)
    * [All users to recommendations v1 and v2](#all-users-to-recommendations-v1-and-v2)
    * [Split traffic between v1 and v2](#split-traffic-between-v1-and-v2)
* [Fault Injection](#fault-injection)
  * [HTTP Error 503](#http-error-503)
  * [Delay](#delay)
* [Retry](#retry)
* [Timeout](#timeout)
* [Smart routing based on user-agent header (Canary Deployment)](#smart-routing-based-on-user-agent-header-canary-deployment)
    * [Set recommendations to all v1](#set-recommendations-to-all-v1)
    * [Set Safari users to v2](#set-safari-users-to-v2)
    * [Set mobile users to v2](#set-mobile-users-to-v2)
    * [Clean up](#clean-up)
* [Mirroring Traffic (Dark Launch)](#mirroring-traffic-dark-launch)
* [Access Control](#access-control)
    * [Whitelist](#whitelist)
    * [Blacklist](#blacklist)
* [Load Balancer](#load-balancer)
* [Circuit Breaker](#circuit-breaker)
    * [Fail Fast with Max Connections & Max Pending Requests](#fail-fast-with-max-connections-max-pending-requests)
    * [Pool Ejection](#pool-ejection)
* [Egress](#egress)
    * [Create HTTPBin Java App](#create-httpbin-java-app)
    * [Create the Github Java App](#create-the-github-java-app)
    * [Istio-ize Egress](#istio-ize-egress)
* [Rate Limiting](#rate-limiting)
* [Tips & Tricks](#tips-tricks)

<!-- toc stop -->



## Prerequisite CLI tools
You will need in this tutorial
* minishift (https://github.com/minishift/minishift/releases)
* docker (https://www.docker.com/docker-mac)
* kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-via-curl)
* oc (eval $(minishift oc-env))
* mvn (https://archive.apache.org/dist/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz)
* stern (brew install stern)
* istioctl (will be installed via the steps below)
* curl, gunzip, tar are built-in to MacOS or part of your bash shell
* git (everybody needs the git CLI)

## Setup minishift
Assumes minishift, tested with minshift v1.10.0+10461c6

Minishift creation script
```bash
#!/bin/bash

# add the location of minishift execuatable to PATH
# I also keep other handy tools like kubectl and kubetail.sh
# in that directory

export PATH=/Users/burr/minishift_1.10.0/:$PATH

minishift profile set tutorial
minishift config set memory 8GB
minishift config set cpus 3
minishift config set vm-driver virtualbox (optional)
minishift config set image-caching true
minishift addon enable admin-user
minishift config set openshift-version v3.7.0

minishift start

```
## Setup environment

```bash
eval $(minishift oc-env)
eval $(minishift docker-env)
oc login $(minishift ip):8443 -u admin -p admin
```
Note: In this tutorial, you will often be polling the customer endpoint with curl, while simultaneously viewing logs via stern or kubetail.sh and issuing commands via oc and istioctl.  Consider using three terminal windows.


## Istio installation script

```bash
#!/bin/bash

curl -LO https://github.com/istio/istio/releases/download/0.4.0/istio-0.4.0-osx.tar.gz

gunzip istio-0.4.0-osx.tar.gz

tar -xvzf istio-0.4.0-osx.tar

cd istio-0.4.0

# make sure we are logged in
oc login $(minishift ip):8443 -u admin -p admin

oc adm policy add-scc-to-user anyuid -z istio-ingress-service-account -n istio-system

oc adm policy add-scc-to-user anyuid -z istio-egress-service-account -n istio-system

oc adm policy add-scc-to-user anyuid -z default -n istio-system

oc create -f install/kubernetes/istio.yaml

oc project istio-system

oc expose svc istio-ingress

oc apply -f install/kubernetes/addons/prometheus.yaml

oc apply -f install/kubernetes/addons/grafana.yaml

oc apply -f install/kubernetes/addons/servicegraph.yaml

oc expose svc servicegraph

oc expose svc grafana

oc expose svc prometheus

oc process -f https://raw.githubusercontent.com/jaegertracing/jaeger-openshift/master/all-in-one/jaeger-all-in-one-template.yml | oc create -f -

```
Wait for Istio's components to be ready

```
oc get pods
NAME                             READY     STATUS    RESTARTS   AGE
grafana-3617079618-4qs2b         1/1       Running   0          4m
istio-ca-1363003450-tfnjp        1/1       Running   0          4m
istio-ingress-1005666339-vrjln   1/1       Running   0          4m
istio-mixer-465004155-zn78n      3/3       Running   0          5m
istio-pilot-1861292947-25hnm     2/2       Running   0          4m
jaeger-210917857-2w24f           1/1       Running   0          4m
prometheus-168775884-dr5dm       1/1       Running   0          4m
servicegraph-1100735962-tdh78    1/1       Running   0          4m
```

And if you need quick access to the OpenShift console

```
minishift console
```
Note: on your first launch of the OpenShift console via minishift, you will like receive a warning with
"Your connection is not private", it depends on your browser type and settings.  Simply select "Proceed to 192.168.99.100 (unsafe)" to bypass the warning.

For minishift, with the admin-user addon, the user is "admin" and the password is "admin"

## Deploy customer

Make sure you have are logged in
```
oc whoami
```
and you have setup the project/namespace

```
oc new-project tutorial
oc adm policy add-scc-to-user privileged -z default -n tutorial
```
Then clone the git repository
```
git clone https://github.com/redhat-developer-demos/istio-tutorial
cd istio-tutorial
```
 Start deploying the microservice projects, starting with customer
```
cd customer

mvn clean package

docker build -t example/customer .

docker images | grep customer
```
Note: Your very first docker build will take a bit of time as it downloads all the layers.  Subsequent rebuilds of the docker image, updating only the jar/app layer will be very fast.

Add *istioctl* to your $PATH, you downloaded it a few steps back.  An example
```
export PATH=/Users/burr/minishift_1.10.0/istio-0.4.0/bin:$PATH

istioctl version

Version: 0.4.0
GitRevision: 24089ea97c8d244493c93b499a666ddf4010b547-dirty
GitBranch: 6401744b90b43901b2aa4a8bced33c7bd54ffc13
User: root@cc5c34bbd1ee
GolangVersion: go1.8

```
Now let's deploy the customer pod with its sidecar
```
oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n tutorial

oc create -f src/main/kubernetes/Service.yml -n tutorial
```

Since customer is the forward most microservice (customer -> preferences -> recommendations), let's add an OpenShift Route that exposes that endpoint.

```
oc expose service customer

oc get route

oc get pods -w
```
Waiting for Ready 2/2, to break out of the waiting use "ctrl-c"

Then test the customer endpoint
```
curl $(minishift openshift service customer --url)
```
You should see the following error because preferences and recommendations are not yet deployed.

```
C100 * I/O error on GET request for "http://preferences:8080": preferences; nested exception is java.net.UnknownHostException: preferences *
```
Also review the logs
```
stern customer -c customer

customer-3600192384-4g8h8 customer org.springframework.web.client.ResourceAccessException: I/O error on GET request for "http://preferences:8080": preferences; nested exception is java.net.UnknownHostException: preferences

```

Back to the main istio-tutorial directory

```
cd ..
```

## Deploy preferences
```
cd preferences

mvn clean package

docker build -t example/preferences .

docker images | grep preferences

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n tutorial

oc create -f src/main/kubernetes/Service.yml

oc get pods -w
```
Wait for the Ready 2/2
```
curl $(minishift openshift service customer --url)
```
It will respond with an error since recommendations is not yet deployed. 
Note: We could make this a bit more resilent in a future iteration of this tutorial
```
C100 * 503 Service Unavailable *
```
and check out the logs 
```
stern preferences -c preferences

preferences-243057078-rwvds preferences org.springframework.web.client.ResourceAccessException: I/O error on GET request for "http://recommendations:8080": recommendations; nested exception is java.net.UnknownHostException: recommendations
```
Back to the main istio-tutorial directory

```
cd ..
```

## Deploy recommendations
Note: The tag "v1" at the end of the image name is important.  We will be creating a v2 version of recommendations later in this tutorial.   Having both a v1 and v2 version of the recommendations code will allow us to exercise some interesting aspects of Istio's capabilities.

```
cd recommendations

mvn clean package

docker build -t example/recommendations:v1 .

docker images | grep recommendations

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n tutorial

oc create -f src/main/kubernetes/Service.yml

oc get pods -w

curl $(minishift openshift service customer --url)
```

it returns

```
C100 * {"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 1 *
```
and you can monitor the recommendation logs with
```
stern recommendations -c recommendations
```

Back to the main istio-tutorial directory

```
cd ..
```

## Updating & redeploying code
When you wish to change code (e.g. editing the .java files) and wish to "redeploy", simply:
```
cd {servicename}

vi src/main/java/com/example/{servicename}/{Servicename}Controller.java
```
Make your edits and esc-w-q

```
mvn clean package
docker build -t example/{servicename} .

oc get pods -o jsonpath='{.items[*].metadata.name}' -l app={servicename}
oc get pods -o jsonpath='{.items[*].metadata.name}' -l app={servicename},version=v1

oc delete pod -l app={servicename},version=v1
```
Why the delete pod?

Based on the Deployment configuration, Kubernetes/OpenShift will recreate the pod, based on the new docker image as it attempts to keep the desired replicas available

```
oc describe deployment {servicename} | grep Replicas
```

## Monitoring
Out of the box, you get monitoring via Prometheus and Grafana.  
```
minishift openshift service grafana --in-browser
```
Make sure to select "Istio Dashboard" in the Grafana Dashboard

![alt text](readme_images/grafana1.png "Grafana Istio Dashboard")

Scroll-down to see the stats for customer, preferences and recommendations

![alt text](readme_images/grafana2.png "Customer Preferences")

## Custom Metrics
Istio also allows you to specify custom metrics which can be seen inside of the Prometheus dashboard

```
minishift openshift service prometheus --in-browser
```
Add the custom metric and rule.  First make sure you are in the "istio-tutorial" directory and then

```
oc apply -f istiofiles/recommendations_requestcount.yml -n istio-system
```
In the Prometheus dashboard, add the following
```
round(increase(istio_recommendations_request_count{destination="recommendations.tutorial.svc.cluster.local" }[60m]))
```
and select Execute

![alt text](readme_images/prometheus_custom_metric.png "Prometheus with custom metric")

Then run several requests through the system
```
curl $(minishift openshift service customer --url)
```
Note: you may have to refresh the browser for the Prometheus graph to update. And you may wish to make the interval 5m (5 minutes) as seen in the screenshot above.

## Tracing

Tracing requires a bit of work on the Java side.  Each microservice needs to pass on the headers which are used to enable the traces.

https://github.com/redhat-developer-demos/istio-tutorial/blob/master/customer/src/main/java/com/example/customer/tracing/HttpHeaderForwarderHandlerInterceptor.java

and

https://github.com/redhat-developer-demos/istio-tutorial/blob/master/customer/src/main/java/com/example/customer/CustomerApplication.java#L21-L31

To open the Jaeger console, select customer from the list of services and Find Traces
```
minishift openshift service jaeger-query --in-browser
```
![alt text](readme_images/jaegerUI.png "Jaeger with Customer")

## Istio RouteRule Changes
### recommendations:v2
We can experiment with Istio routing rules by making a change to RecommendationsController.java like the following and creating a "v2" docker image.

```java
logger.debug(String.format("Big Red Dog v2 %s %d", HOSTNAME, count));

return ResponseEntity.ok(String.format("Clifford v2 %s %d", HOSTNAME, count));
```

The "v2" tag during the docker build is significant.

There is also a 2nd deployment.yml file to label things correctly

```
cd recommendations

mvn clean package

docker build -t example/recommendations:v2 .

docker images | grep recommendations
example/recommendations                  v2                  c31e399a9628        5 seconds ago       438MB
example/recommendations                  v1              f072978d9cf6        8 minutes ago      438MB

```
*Important:* back up one directory before applying the deployment yaml.  We have a 2nd Deployment to manage the v2 version of recommendations.  
```
cd ..

oc apply -f <(istioctl kube-inject -f kubernetesfiles/recommendations_v2_deployment.yml) -n tutorial

oc get pods -w
```
Wait for those pods to show "2/2", the istio-proxy/envoy sidecar is part of that pod
```
NAME                                  READY     STATUS    RESTARTS   AGE
customer-3600192384-fpljb             2/2       Running   0          17m
preferences-243057078-8c5hz           2/2       Running   0          15m
recommendations-v1-60483540-9snd9     2/2       Running   0          12m
recommendations-v2-2815683430-vpx4p   2/2       Running   0         15s
```
and test the customer endpoint
```
curl $(minishift openshift service customer --url)
```

you likely see "Clifford v1 {hostname} 5", where the 5 is basically the number of times you hit the endpoint.

```
curl $(minishift openshift service customer --url)
```

you likely see "Clifford v2 {hostname} 1" as by default you get round-robin load-balancing when there is more than one Pod behind a Service

Send several requests to see their responses
```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .1
done
```

The default Kubernetes/OpenShift behavior is to round-robin load-balance across all available pods behind a single Service.  Add another replica of recommendations-v2 Deployment.

```
oc scale --replicas=2 deployment/recommendations-v2
```

Now, you will see two requests into the v2 and one for v1.
```
C100 * {"P1":"Red", "P2":"Big"} && Clifford v1 60483540-mlv48 302 *
C100 * {"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-hqfz7 293 *
C100 * {"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-hqfz7 294 *
```
Scale back to a single replica of the recommendations-v2 Deployment

```
oc scale --replicas=1 deployment/recommendations-v2
```
and back to the main directory
```
cd ..
```

## Changing Istio RouteRules

#### All users to recommendations:v2
From the main istio-tutorial directory,
```
oc create -f istiofiles/route-rule-recommendations-v2.yml -n tutorial

curl $(minishift openshift service customer --url)
```

you should only see v2 being returned

#### All users to recommendations:v1
Note: "replace" instead of "create" since we are overlaying the previous rule

```
oc replace -f istiofiles/route-rule-recommendations-v1.yml -n tutorial

oc get routerules -n tutorial

oc get routerules/recommendations-default -o yaml -n tutorial
```
#### All users to recommendations v1 and v2
By simply removing the rule

```
oc delete routerules/recommendations-default -n tutorial
```
and you should see the default behavior of load-balancing between v1 and v2
```
curl customer-tutorial.$(minishift ip).nip.io
```
#### Split traffic between v1 and v2
Canary Deployment scenario: push v2 into the cluster but slowly send end-user traffic to it, if you continue to see success, continue shifting more traffic over time

```
oc get pods -l app=recommendations -n tutorial
NAME                                  READY     STATUS    RESTARTS   AGE
recommendations-v1-3719512284-7mlzw   2/2       Running   6          2h
recommendations-v2-2815683430-vn77w   2/2       Running   0          1h
```
Create the routerule that will send 90% of requests to v1 and 10% to v2
```
oc create -f istiofiles/route-rule-recommendations-v1_and_v2.yml -n tutorial
```
and send in several requests

```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .1
done
```

In another terminal, change the mixture to be 75/25
```
oc replace -f istiofiles/route-rule-recommendations-v1_and_v2_75_25.yml -n tutorial
```

Clean up
```
oc delete routerule recommendations-v1-v2 -n tutorial
```


## Fault Injection
Apply some chaos engineering by throwing in some HTTP errors or network delays.  Understanding failure scenarios is a critical aspect of microservices architecture  (aka distributed computing)

### HTTP Error 503
By default, recommendations v1 and v2 are being randomly load-balanced as that is the default behavior in Kubernetes/OpenShift

```
oc get pods -l app=recommendations -n tutorial
NAME                                  READY     STATUS    RESTARTS   AGE
recommendations-v1-3719512284-7mlzw   2/2       Running   6          18h
recommendations-v2-2815683430-vn77w   2/2       Running   0          3h
```

You can inject 503's, for approximately 50% of the requests
```
oc create -f istiofiles/route-rule-recommendations-503.yml -n tutorial

curl $(minishift openshift service customer --url)
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 6*
curl $(minishift openshift service customer --url)
C100 *{"P1":"Red", "P2":"Big"} && 503 Service Unavailable *
curl $(minishift openshift service customer --url)
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 60483540-2pt4z 6*
```
Clean up
```
oc delete routerule recommendations-503 -n tutorial
```
### Delay
The most insidious of possible distributed computing faults is not a "down" service but a service that is responding slowly, potentially causing a cascading failure in your network of services.

```
oc create -f istiofiles/route-rule-recommendations-delay.yml -n tutorial
```
And hit the customer endpoint

```bash
#!/bin/bash
  
while true
do
time curl $(minishift openshift service customer --url)
sleep .1
done
```
You will notice many requets to the customer endpoint now have a delay.
If you are monitoring the logs for recommendations v1 and v2, you will also see the delay happens BEFORE the recommendations service is actually called

```
stern recommendations -n tutorial
or
./kubetail.sh recommendations -n tutorial
```
Clean up
```
oc delete routerule recommendations-delay -n tutorial
```

## Retry
Instead of failing immediately, retry the Service N more times

We will use Istio and return 503's about 50% of the time.  Send all users to v2 which will throw out some 503's

```
oc create -f istiofiles/route-rule-recommendations-v2_503.yml -n tutorial
```

Now, if you hit the customer endpoint several times, you should see some 503's

```
curl $(minishift openshift service customer --url)
C100 *{"P1":"Red", "P2":"Big"} && 503 Service Unavailable *
```

Now add the retry rule
```
oc create -f istiofiles/route-rule-recommendations-v2_retry.yml -n tutorial
```
and after a few seconds, things will settle down and you will see it work every time
```
curl $(minishift openshift service customer --url)

C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 60483540-2pt4z 123*
```
You can see the active RouteRules via
```
oc get routerules -n tutorial
```
Now, delete the retry rule and see the old behavior, some random 503s
```
oc delete routerule recommendations-v2-retry -n tutorial

curl $(minishift openshift service customer --url)
```
Now, delete the 503 rule and back to random load-balancing between v1 and v2
```
oc delete routerule recommendations-v2-503 -n tutorial

curl $(minishift openshift service customer --url)
```

## Timeout
Wait only N seconds before giving up and failing.  At this point, no other route rules should be in effect.  oc get routerules and oc delete routerule rulename if there are some.

First, introduce some wait time in recommendations v2 by uncommenting the line that call the timeout method. Update RecommendationsController.java making it a slow perfomer

```java
    @RequestMapping("/")
    public ResponseEntity<String> getRecommendations() {
        count++;
        logger.debug(String.format("Big Red Dog v1 %s %d", HOSTNAME, count));

        timeout();

        logger.debug("recommendations ready to return");
        if (misbehave) {
            return doMisbehavior();
        }
        return ResponseEntity.ok(String.format("Clifford v1 %s %d", HOSTNAME, count));
    }
```
Rebuild and redeploy
```
cd recommendations

mvn clean package

docker build -t example/recommendations:v2 .

docker images | grep recommendations

oc delete pod -l app=recommendations,version=v2 -n tutorial

cd ..
```
Hit the customer endpoint a few times, to see the load-balancing between v1 and v2 but with v2 taking a bit of time to respond

```bash
#!/bin/bash
  
while true
do
time curl $(minishift openshift service customer --url)
sleep .1
done
``` 

Then add the timeout rule

```
oc create -f istiofiles/route-rule-recommendations-timeout.yml -n tutorial

time curl $(minishift openshift service customer --url)
```
You will see it return v1 OR 504 after waiting about 1 second

```
time curl $(minishift openshift service customer --url)
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 6*
time curl $(minishift openshift service customer --url)
C100 *{"P1":"Red", "P2":"Big"} && 504 Gateway Timeout *
```

Clean up, delete the timeout rule

```
oc delete routerule recommendations-timeout -n tutorial

```
## Smart routing based on user-agent header (Canary Deployment)

What is your user-agent?

https://www.whoishostingthis.com/tools/user-agent/

Note: the "user-agent" header being forwarded in the Customer and Preferences controllers in order for route rule modications around recommendations

#### Set recommendations to all v1
```
oc create -f istiofiles/route-rule-recommendations-v1.yml -n tutorial
```
#### Set Safari users to v2
```
oc create -f istiofiles/route-rule-safari-recommendations-v2.yml -n tutorial

oc get routerules -n tutorial
```

and test with a Safari (or even Chrome on Mac since it includes Safari in the string).  Safari only sees v2 responses from recommendations

and test with a Firefox browser, it should only see v1 responses from recommendations.

There are two ways to get the URL for your browser:

```
minishift openshift service customer --in-browser

```
That will open the openshift service `customer` in browser

Or

if you need just the url alone:

```
minishift openshift service customer --url
http://customer-tutorial.192.168.99.102.nip.io
```
You can also attempt to use the curl -A command to test with different user-agent strings.  

```
curl -A Safari $(minishift openshift service customer --url)
curl -A Firefox $(minishift openshift service customer --url)
```

You can describe the routerule to see its configuration
```
oc describe routerule recommendations-safari -n tutorial
```


Remove the Safari rule

```
oc delete routerule recommendations-safari -n tutorial
```
#### Set mobile users to v2
```
oc create -f istiofiles/route-rule-mobile-recommendations-v2.yml -n tutorial

curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" $(minishift openshift service customer --url)
```

#### Clean up
```
oc delete routerule recommendations-mobile -n tutorial
```

## Mirroring Traffic (Dark Launch)
Wiretap, eavesdropping
Note: does not seem to work in 0.4.0

```
oc get pods -l app=recommendations -n tutorial
```
You should have 2 pods for recommendations based on the steps above

```
oc get routerules -n tutorial
```
You should have NO routerules
if so "oc delete routerule rulename -n tutorial"

Make sure you are in the main directory of "istio-tutorial"

```
oc create -f istiofiles/route-rule-recommendations-v1-mirror-v2.yml -n tutorial

curl $(minishift openshift service customer --url)
```

## Access Control

#### Whitelist

We'll create a whitelist on the preferences service to only allow requests from the recommendations service, which will make the preferences service invisible to the customer service. Requests from the customer service to the preferences service will return a 404 Not Found HTTP error code.

```
istioctl create -f istiofiles/acl-whitelist.yml -n tutorial
```

```
curl $(minishift openshift service customer --url)
C100 *404 Not Found *
```

##### To reset the environment:

```
istioctl delete -f istiofiles/acl-whitelist.yml -n tutorial
```

#### Blacklist

We'll create a blacklist making the customer service blacklist to the preferences service. Requests from the customer service to the preferences service will return a 403 Forbidden HTTP error code.

```
istioctl create -f istiofiles/acl-blacklist.yml -n tutorial
```

```
curl $(minishift openshift service customer --url)
C100 *403 Forbidden * 
```

##### To reset the environment:

```
istioctl delete -f istiofiles/acl-blacklist.yml -n tutorial
```


## Load Balancer

By default, you will see "round-robin" style load-balancing, but you can change it up, with the RANDOM option being fairly visible to the naked eye.

Add another v2 pod to the mix

```
oc scale deployment recommendations-v2 --replicas=2 -n tutorial

```
Wait a bit (oc get pods -w to watch)
and curl the customer endpoint many times

```
curl $(minishift openshift service customer --url)
```

Add a 3rd v2 pod to the mix

```
oc scale deployment recommendations-v2 --replicas=3 -n tutorial

oc get pods -n tutorial
NAME                                  READY     STATUS    RESTARTS   AGE
customer-1755156816-cjd2z             2/2       Running   0          1h
preferences-3336288630-2cc6f          2/2       Running   0          1h
recommendations-v1-3719512284-bn42p   2/2       Running   0          59m
recommendations-v2-2815683430-97nnf   2/2       Running   0          43m
recommendations-v2-2815683430-d49n6   2/2       Running   0          51m
recommendations-v2-2815683430-tptf2   2/2       Running   0          33m
```
Wait for those 2/2 (two containers in each pod) and then poll the customer endpoint

```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .1
done
```
The results should follow a fairly normal round-robin distribution pattern

```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 7* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-985tm 2* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 2* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-glw97 2* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 8* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-985tm 3* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 3* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-glw97 3* 
```

Now, add the Random LB DestinationPolicy

```
oc create -f istiofiles/recommendations_lb_policy_app.yml -n tutorial
```

And you should see a different pattern of which pod is being selected
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 10* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 5* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-glw97 4* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 11* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-glw97 5* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-glw97 6* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 6* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-glw97 7* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 7* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 8* 
```

Clean up
```
oc delete -f istiofiles/recommendations_lb_policy_app.yml -n tutorial

oc scale deployment recommendations-v2 --replicas=1 -n tutorial
```

## Circuit Breaker

#### Fail Fast with Max Connections & Max Pending Requests
First, you need to insure you have a routerule in place.  Let's use a 50/50 split of traffic which is more like the default behavior of Kubernetes.  

```
oc create -f istiofiles/route-rule-recommendations-v1_and_v2_50_50.yml -n tutorial
```

and if you polling the endpoint repeatedly, you will see the Istio behavior:

```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .5
done
```
Output
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 20* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 21* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 16* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 17* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 22* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 23* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 24* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 18* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 25* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 19* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-8hjsd 20* 
```
With vanilla Kubernetes/OpenShift, the distrubtion of load is more round robin, while with Istio it is 50/50 but more random.

Next, update RecommendationsController.java by uncommenting the line that call the timeout method, and changing the flag `misbehave` to `true`. These modifications will make it a slow perfomer and throw somes 503s.

```java
    /**
     * Flag for throwing a 503 when enabled
     */
    private boolean misbehave = true;
    
    // ...

    @RequestMapping("/")
    public ResponseEntity<String> getRecommendations() {
        count++;
        logger.debug(String.format("Big Red Dog v1 %s %d", HOSTNAME, count));

        timeout();

        logger.debug("recommendations ready to return");
        if (misbehave) {
            return doMisbehavior();
        }
        return ResponseEntity.ok(String.format("Clifford v1 %s %d", HOSTNAME, count));
    }
```
Rebuild, redeploy
```
cd recommendations
mvn clean package
docker build -t example/recommendations:v2 .
docker images | grep recommendations
oc delete pod -l app=recommendations,version=v2 -n tutorial
```

The deletion of the previously running pod will cause Kubernetes/OpenShift to restart it based on the new docker image.

Back to the main directory
```
cd ..
```
and test the customer endpoint

```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .5
done
```

Whenever you are hitting v2, you will notice the slowness in the response based on the Thread.sleep(3000)

Watch the logging output of recommendations

```
Terminal 1:
./kubetail.sh recommendations -n tutorial
or
brew install stern
stern recommendations -c recommendations -n tutorial

Terminal 2:
curl $(minishift openshift service customer --url)
```

Now add the circuit breaker.

```
istioctl create -f istiofiles/recommendations_cb_policy_version_v2.yml -n tutorial

istioctl get destinationpolicies -n tutorial
```
More information on the fields for the simple circuit-breaker
https://istio.io/docs/reference/config/traffic-rules/destination-policies.html#simplecircuitbreakerpolicy


Use gatling, but first modify the URL gatling is pointing at
https://github.com/redhat-developer-demos/istio-tutorial/blob/master/gatling_test/src/test/scala/RecordedSimulation.scala#L11

then 
```
cd gatling_test
mvn integration-test -Dusers=2 -Dendpoint.url=$(minishift openshift service customer --url)
```
and open the generated report.  

Note: the file name of the report is output from the "mvn integration-test" execution.

```
open /Users/burr/minishift_1.10.0/redhat-developer-demos2/istio-tutorial/gatling_test/target/gatling/recordedsimulation-1516395386155/index.html
```

When using 2 concurrent users, all requests are likely to succeed, there are in fact 2 pods of recommendations available. But build reports a failure as we had set the reponse time to be less than 3 seconds

```
mvn integration-test -Dendpoint.url=$(minishift openshift service customer --url)
```

It will still likely succeed, as by default the number of concurrent users is 5

At this point, that is enough load to have tripped the circuit-breaker and you should see some failures in the report.

If you wish to peer inside the CB

```
istioctl get destinationpolicies recommendations-circuitbreaker -o yaml -n tutorial
```

Now, delete the Destination Policy 
```
istioctl delete destinationpolicy recommendations-circuitbreaker -n tutorial
```
and re-run the load test
```
mvn integration-test
```
Now, even with a load of 5 where there are only two pods, you should see all requests succeed as there is no circuit-breaker in the middle, tripping/opening, but you will notice maven build reporting failure because of the SLA in reponse time which is set at 3 seconds


Clean up
```
oc delete routerule recommendations-v1-v2 -n tutorial
```

#### Pool Ejection
There is a 2nd circuit-breaker policy yaml file. In this case, we are attempting load-balancing pool ejection.  We want the slow and misbehaving instance of recommendations v2 to be kicked out and more requests to be handled by v1.  Envoy refers to this as "outlier detection".

Throw some requests at the customer endpoint
```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .1
done
```

By default, you will see load-balancing behind that URL, across the 2 pods that are currently in play. By default Kubernetes/OpenShift will alternative between v1 and v2
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 8* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 897* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 9* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 898* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 10* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 899* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 11* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 900* 
```
Add a 2nd pod to recommendations
```
oc scale deployment recommendations-v2 --replicas=2 -n tutorial

oc get pods

NAME                                  READY     STATUS    RESTARTS   AGE
customer-3600192384-fpljb             2/2       Running   0          2h
preferences-243057078-8c5hz           2/2       Running   0          2h
recommendations-v1-60483540-2pt4z     2/2       Running   0          40m
recommendations-v2-2815683430-t7b9q   2/2       Running   0          21s
recommendations-v2-2815683430-xw7qg   2/2       Running   0          19m
```
and your pattern will change slightly to v1, v2, v2 then repeat
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 901* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-t7b9q 2* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 13* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 902* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-t7b9q 3* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 14*
```

In another Terminal, setup the Destination Policy
```
istioctl create -f istiofiles/recommendations_cb_policy_app.yml -n tutorial
```
You should see the Random load-balancing take effect
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 905* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 18* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 906* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 19* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-t7b9q 6* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 20* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 21* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 22* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 23* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-t7b9q 7* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 907* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 908* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 909* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 24* 
```

Now, simply just delete a v2 pod as that will cause 5xx errors

```
oc delete pod recommendations-v2-2815683430-t7b9q
```
you should see a single 503 returned to the end-user
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 30* 
{"timestamp":1516466806299,"status":503,"error":"Service Unavailable","exception":"com.example.customer.CustomerController$ServiceUnavailableException","message":"503 Service Unavailable","path":"/"}
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 31* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 32* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 919* 
```

OR throw in some misbehavior by getting the pod identifiers

```
oc get pods
```
and then shelling into a v2 pod
```
oc exec -it recommendations-v2-2815683430-xw7qg -c recommendations /bin/bash
```
and then hit its misbehave endpoint to set the flag
```
curl localhost:8080/misbehave
```
At this point, you should get a 503 from the v2 pod that was flagged and 
you should see requests/traffic focusing on the "good" pods, until the sleepWindow expires.
```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-gltbm 5* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-xw7qg 33* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 985* 
{"timestamp":1516467547342,"status":503,"error":"Service Unavailable","exception":"com.example.customer.CustomerController$ServiceUnavailableException","message":"503 Service Unavailable","path":"/"}
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 986* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-gltbm 6* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-gltbm 7* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 987* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 60483540-2pt4z 988* 
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 2815683430-gltbm 8*
```
If you wait long enough, you should see the v2 pod reenter the load-balancing pool 

Clean up

```
oc scale deployment recommendations-v2 --replicas=1 -n tutorial

istioctl delete destinationpolicies recommendations-poolejector -n tutorial
```

## Egress
There are two examples of egress routing, one for httpbin.org and one for github.  Egress routes allow you to apply rules to how internal services interact with external APIs/services.

Create a namespace/project to hold these egress examples
```
oc new-project istioegress
oc adm policy add-scc-to-user privileged -z default -n istioegress
```

#### Create HTTPBin Java App
```
cd egresshttpbin/

mvn spring-boot:run

curl localhost:8080

ctrl-c

mvn clean package

docker build -t example/egresshttpbin:v1 .

docker images | grep egress

docker run -it -p 8080:8080 --rm example/egresshttpbin:v1

curl $(minishift ip):8080

ctrl-c

docker ps | grep egress

docker ps -a | grep egress

oc apply -f src/main/kubernetes/Deployment.yml -n istioegress

oc create -f src/main/kubernetes/Service.yml

oc expose service egresshttpbin

curl egresshttpbin-istioegress.$(minishift ip).nip.io

```

Note: It does not work...yet, more to come.

Back to the main istio-tutorial directory

```
cd ..
```

#### Create the Github Java App
```
cd egressgithub/

mvn clean package

docker build -t example/egressgithub:v1 .

docker images | grep egress

docker run -it -p 8080:8080 --rm example/egressgithub:v1

curl $(minishift ip):8080
```

Note: it will not work now but it will once Istio-ized

```
ctrl-c

docker ps | grep egress

oc apply -f src/main/kubernetes/Deployment.yml -n istioegress

oc create -f src/main/kubernetes/Service.yml

oc expose service egressgithub

curl egressgithub-istioegress.$(minishift ip).nip.io

cd ..
```

#### Istio-ize Egress
```
istioctl create -f istiofiles/egress_httpbin.yml

istioctl get egressrules

curl egresshttpbin-istioegress.$(minishift ip).nip.io
```
or shell into the pod by getting its name and then using that name with oc exec
```
oc exec -it $(oc get pods -o jsonpath="{.items[*].metadata.name}" -l app=egresshttpbin,version=v1) /bin/bash

curl localhost:8080

curl httpbin.org/user-agent

curl httpbin.org/headers

exit
```
add a egressrule for google
```
cat <<EOF | istioctl create -f -
apiVersion: config.istio.io/v1alpha2
kind: EgressRule
metadata:
  name: google-egress-rule
spec:
  destination:
    service: www.google.com
  ports:
    - port: 443
      protocol: https
EOF
```
and shell into it for testing

```
oc exec -it $(oc get pods -o jsonpath="{.items[*].metadata.name}" -l app=egressgithub,version=v1) /bin/bash

curl http://www.google.com:443

exit
```
Now, execute the Java code that hits api.google.com/users
```
istioctl create -f istiofiles/egress_github.yml

curl egressgithub-istioegress.$(minishift ip).nip.io
```

## Rate Limiting
Here we will limit the number of concurrent requests into recommendations v2

Current view of the v2 RecommendationsController.java
```java
package com.example.recommendations;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.http.HttpStatus;

@RestController
public class RecommendationsController {
    int cnt = 0; // helps us see the lifecycle 
    boolean misbehave = false; // a flag for throwing a 503
    final String hostname = System.getenv().getOrDefault("HOSTNAME", "unknown");

    @RequestMapping("/")
    public String getRecommendations() {

        cnt ++;        
        // hostname.substring(19) to remove "recommendations-v1-"
        System.out.println("Big Red Dog v2 " + hostname.substring(19) + " " + cnt);

        // begin timeout and/or circuit-breaker example
        try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {			
			e.printStackTrace();
		}
        System.out.println("recommendations ready to return");
        // end circuit-breaker example */
        // throw new ServiceUnavailableException();
        return "Clifford v2 " + hostname.substring(19) + " " + cnt;
    }
    @RequestMapping("/misbehave")
    public HttpStatus misbehave() {
        this.misbehave = true; // set a flag
        return HttpStatus.OK;
    }
}

@ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
class ServiceUnavailableException extends RuntimeException {
    public ServiceUnavailableException(String message) {
        super(message);
    }
}
```

Now apply the rate limit handler

```
istioctl create -f istiofiles/recommendations_rate_limit_handler.yml
```

Now setup the requestcount quota

```
istioctl create -f istiofiles/rate_limit_rule.yml
```

Throw some requests at customer

```bash
#!/bin/bash

while true
do curl $(minishift openshift service customer --url)
sleep .1
done
```

and watch the logs for preferences to see some 429 Too Many Requests errors

```
stern preferences -c preferences -n tutorial
```

Clean up

```
istioctl delete -f istiofiles/rate_limit_rule.yml

istioctl delete -f istiofiles/recommendations_rate_limit_handler.yml
```

## Tips & Tricks
Some tips and tricks that you might find handy

You have two containers in a pod
```
oc get pods -o jsonpath="{.items[*].spec.containers[*].name}" -l app=customer -n tutorial
```
From these images
```
oc get pods -o jsonpath="{.items[*].spec.containers[*].image}" -l app=customer -n tutorial
```
Get the pod ids
```
CPOD=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=customer -n tutorial)
PPOD=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=preferences -n tutorial)
RPOD1=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=recommendations,version=v1 -n tutorial)
RPOD2=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=recommendations,version=v2 -n tutorial)
```

The pods all see each other's services
```
oc exec $CPOD -c customer -n tutorial curl http://preferences:8080 
oc exec $CPOD -c customer -n tutorial curl http://recommendations:8080
oc exec $RPOD2 -c recommendations -n tutorial curl http://customer:8080
```

```
oc exec $CPOD -c customer -n tutorial curl http://localhost:15000/routes > afile.json
```
Look for "route_config_name": "8080", you should see 3 entries for customer, preferences and recommendations
https://gist.github.com/burrsutter/9117266f84efe124590e9014793c10f6

Now add a new routerule
```
oc create -f istiofiles/route-rule-recommendations-v2.yml
```
The review the routes again
```
oc exec $CPOD -c customer -n tutorial curl http://localhost:15000/routes > bfile.json
```

Here is the Before:
https://gist.github.com/burrsutter/9117266f84efe124590e9014793c10f6#file-gistfile1-txt-L41

and

https://gist.github.com/burrsutter/9117266f84efe124590e9014793c10f6#file-gistfile1-txt-L45

And the After:
https://gist.github.com/burrsutter/8b92da2ad0a8ec1b975f5dfa6ddc17f8#file-gistfile1-txt-L41

and

https://gist.github.com/burrsutter/8b92da2ad0a8ec1b975f5dfa6ddc17f8#file-gistfile1-txt-L45


If you need the Pod IP
```
oc get pods -o jsonpath='{.items[*].status.podIP}' -l app=customer -n tutorial
```

Dive into the istio-proxy container

```
oc exec -it $CPOD -c istio-proxy -n tutorial /bin/bash
cd /etc/istio/proxy
ls
cat envoy-rev3.json
```

Snowdrop Troubleshooting
https://github.com/snowdrop/spring-boot-quickstart-istio/blob/master/TROUBLESHOOT.md
