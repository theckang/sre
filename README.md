# OpenShift Site Reliability Engineering

The goal of this repo is to demonstrate aspects of Site Reliability Engineering using OpenShift and OpenShift Service Mesh.

## Prequisites

* Admin access to OpenShift cluster

## Setup

Follow the [instructions](https://docs.openshift.com/container-platform/4.5/service_mesh/service_mesh_install/installing-ossm.html#ossm-operatorhub-install_installing-ossm) to deploy service mesh.  Use the default CRDs provided.

After you install the service mesh control plane, create a new project:

```bash
oc create myproject
```

Add this project to the service mesh:

```bash
oc create -f - <<EOF
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
    - myproject
EOF
```

Download the SRE workshop repo to install the sample microservices application:

```bash
git clone https://github.com/RedHatGov/sre-workshop-code
```

Note: The original source for this application is [here](https://github.com/dudash/openshift-microservices).

Deploy the microservices and gateway:

```bash
oc new-app -f ./setup/microservices-app-ui.yaml -e FAKE_USER=true
oc new-app -f ./setup/microservices-boards.yaml
oc create -f ./setup/gateway.yaml
```

Set the gateway URL:

```bash
GATEWAY_URL=$(oc get route istio-ingressgateway -n istio-system --template='http://{{.spec.host}}')
echo $GATEWAY_URL
```

## SLO Dashboards

Start by sending traffic to the app:

```bash
while true; do curl -s -o /dev/null $GATEWAY_URL; done
```

Open Grafana dashboards in the browser:

```bash
echo $(oc get route grafana -n istio-system --template='https://{{.spec.host}}/dashboards')
```

Download the `dashboard/sample.json` file and import it to Grafana.

Navigate to the imported dashboard and you should see various SLO charts.  In the top right, switch the time range to `Last 5 minutes`. 

![Dashboard](/dashboard/images/dashboard.png?raw=true)

The SLOs use two Service Level Indicators: availability (% of successful requests) and latency (# of seconds to process request).

SLO #1: 95% of requests are successful and return within 1 second (measured in 1 min interval)

SLO #2: 90% of requests are successful and return within 500 milliseconds (measured in 1 min interval)

The time interval is set to 1 minute for the purposes of demonstration.  In reality, this interval would be longer (e.g. 30 days).

The corresponding Error Budget charts are generated for each SLO.

## Failure Scenarios

### Autoscaling

In this scenario, we are going to add autoscaling to the application.

Make sure you are sending traffic to the app if you aren't already:
```bash
while true; do curl -s -o /dev/null $GATEWAY_URL; done
```

Add autoscaling:

```bash
oc apply -f scenarios/autoscaling/app-ui-autoscale.yaml
```

Navigate to Grafana.  Wait a minute and click the refresh icon in the top right.

![Refresh Icon](/dashboard/images/refresh.png?raw=true)

The SLO will be breached, and the error budget will be depleted.

![Failure](/dashboard/images/failure.png?raw=true)

Open the application UI in the browser
```bash
echo $GATEWAY_URL
```

It will return `no healthy upstream`.  Not good.  Our application is inaccessible, and our users will be very unhappy.

What went wrong?  This is an exercise for you to find out :)

Identify:
* How to roll back this change to a previous state
* What factors contributed to the failure?
* How to fix the issue and add autoscaling successfully

Bonus:
* Would this behavior change with a `Deployment` instead of `DeploymentConfig`?  How?

Note: When you fix the issue and deploy autosclaing, it can take awhile for the horizontal pod autoscaler to pick up CPU metrics.  (I've seen up to eight failures before the metrics are successfully retrieved).

### Cron Job

In this scenario, we are going to add a CPU intensive `CronJob`.  We only want one job to run at any time to avoid overtaking the cluster's resources.

Make sure you are sending traffic to the app if you aren't already:
```bash
while true; do curl -s -o /dev/null $GATEWAY_URL; done
```

Take a look at the resources available in your worker nodes:

```bash
oc adm top node -l node-role.kubernetes.io/worker
```

The CPU usage should be relatively low across your nodes.  If your usage is high, remove any other applications or projects you are running that aren't relevant to this exercise.

Deploy the CPU intensive `CronJob`:

```bash
oc apply -f scenarios/cronjob/cronjob.yaml
```

Wait 5 minutes.  The `CronJob` will overtake the worker nodes.

```bash
oc adm top node -l node-role.kubernetes.io/worker
```

Stress the application:

```bash
siege -t 1H -c 6 "$GATEWAY_URL/stress"
```

The SLO will be breached, and the error budget will be depleted.

![SLO Failure](/dashboard/images/slo_failure.png?raw=true)

What went wrong?

Identify:
* How to roll back this change to a previous state
* What factors contributed to the failure?
* How to fix the issue and run the `CronJob` successfully

Bonus:
* How do we prevent the `CronJob` from running indefinitely?  Why should we avoid this?

### Priority Class

In this scenario, we are going to add a `DaemonSet` with a priority class.  We are going to make sure there is plenty of CPU before deploying this `DaemonSet`.  Since the `DaemonSet` has medium priority and there is plenty of CPU resources available, we don't expect any impact to the application.

Make sure you are sending traffic to the app if you aren't already:
```bash
while true; do curl -s -o /dev/null $GATEWAY_URL; done
```

Delete any limit ranges:
```bash
oc delete limitrange --all
```

Modify the daemon set CPU requests.  Use 75% of your node's capacity.  For example, the current YAML requests `12` cores on a `16` vCPU machine.
```bash
oc edit scenarios/priorityclass/medium-daemonset.yaml
```

Observe CPU usage.  There should be plenty of room to run the daemonset:
```bash
oc adm top node -l node-role.kubernetes.io/worker
```

Create medium priority class:
```bash
oc apply -f scenarios/priorityclass/medium-priority.yaml
```

Create daemon set using medium priority:
```bash
oc apply -f scenarios/priorityclass/medium-daemonset.yaml
```

Navigate to Grafana.  The SLO will be breached, and the error budget will be depleted.

What went wrong?

Identify:
* How to roll back this change to a previous state
* What factors contributed to the failure?
* How to fix the issue and add the medium priority `DaemonSet` successfully

### Health Checks

In this scenario, we are going to add [health checks](https://docs.openshift.com/container-platform/4.5/applications/application-health.html) to our application.  The `ReadinessProbe` will ensure the application is ready before it receives traffic, and the `LivenessProbe` will restart the application pod if it determines the application is unhealthy.

For the purpose of this exercise, use my forked repo to deploy the application.

```bash
oc patch bc app-ui -p '{"spec":{"source":{"git":{"uri": "https://github.com/theckang/service-mesh-workshop-code.git"}}}}'
oc start-build app-ui
```

Add health checks to the application:
```bash
oc apply -f scenarios/healthchecks/probes.yaml
```

Stress the application:

```bash
siege -t 1H -c 6 "GATEWAY_URL/stress"
```

The SLO will be breached, and the error budget will be depleted.

![SLO Failure](/dashboard/images/slo_failure.png?raw=true)

If you run:
```bash
oc get pods -l app=app-ui
```

The newest version of the application fails to deploy with probes added.

What went wrong?

Identify:
* How to roll back this change to a previous state
* What factors contributed to the failure?
* How to fix the issue and add health checks successfully
