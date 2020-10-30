![MANUela Logo](./images/logo.png)

# Machine Learning <!-- omit in toc -->
This document describes how to prepare & execute the machine learning demo

- [Prerequisites](#prerequisites)
- [Demo Preparation](#demo-preparation)
- [Demo Execution](#demo-execution)
- [Demo Execution](#demo-execution-1)
  - [Demo ML modeling](#demo-ml-modeling)
  - [Optionally, demo model serving](#optionally-demo-model-serving)

## Prerequisites

The management hub and factory edge clusters are up and running. The [Open Data Hub](https://opendatahub.io/) is deployed during the initial hub blueprint deployment.


To open the jupyterhub , either login into the OpenShift console of your hub clusters and navigate to the ```jupyterhub``` route in the ```manuela-ml-workspace``` namespace or logon via ```oc``` cli into the hub OpenShift clusters and call

```
echo https://$(oc get route jupyterhub -o jsonpath='{.spec.host}' -n manuela-ml-workspace )
```


## Demo Preparation

The demo is focusing on the Data Scientist workbench which is a OpenShift powered [Open Data Hub](https://opendatahub.io/).

On the hub cluster, check that the jupyterhub pods are running:

```
oc get pods -n manuela-ml-workspace
NAME                                     READY   STATUS      RESTARTS   AGE
jupyterhub-1-deploy                      0/1     Completed   0          23h
jupyterhub-1-t8bht                       1/1     Running     0          23h
jupyterhub-db-1-deploy                   0/1     Completed   0          23h
jupyterhub-db-1-zmzcz                    1/1     Running     0          23h
```


## Demo Execution

**Launch a Jupyterhub**  

Get the Jupyterhub URl either from the route in the OpenShift Console or using oc::

```bash

echo https://$(oc get route jupyterhub -o jsonpath='{.spec.host}' -n manuela-ml-workspace )
...
https://jupyterhub-manuela-ml-workspace.apps.<cluster_name>.<base_domain>
```

1. Login with OpenShift credentials
2. Spwan a notebook ```s2i-minimal-notebook:3.6``` using the defaults
3. Upload ```Data-Analyses.ipynb``` and ```raw-data.csv``` from ```~/manuela-dev/ml-models/anomaly-detection/```
   

![launch-jupyter](./images/launch-jupyter.png)


## Demo Execution

### Demo ML modeling

**Demo the notebook**

Open the notebook ```Data-Analyses.ipynb```

Option 1: Lightweigt demo
- All output cells are populated. Don't run any cells. 
- Walk through the content and explain the high level flow.

Option 2: Full demo
- Clear current outputs: ```Cell``` -> ```All Output``` -> ```Clear```
- Run each cell \[Shift]\[Enter] and explain each step.


### Optionally, demo model serving

For keeping the demo setup simple, lets use manuela-stormshift-messaging on the edge cluster for show the model serving.

Show the running seldon pods in manuela-stormshift-messaging.

```bash
oc get pods -n  manuela-stormshift-messaging| grep 'seldon\|anomaly'
```

```
anomaly-detection-predictor-0-anomaly-detection-796887f9899c2jj   2/2     Running   0          22h
seldon-controller-manager-76d49f78b9-k7xc7                        1/1     Running   0          25h
```


**Show logs to see anomaly-detection-predictor in action**

Either on the OpenShift console or using oc
```
oc logs $(oc get pod -l  seldon-app=anomaly-detection-predictor -o jsonpath='{.items[0].metadata.name}' -n manuela-stormshift-messaging) -c anomaly-detection -n manuela-stormshift-messaging
```

Expexted result:
```
 Predict features:  [[ 9.58866665  8.88145877 10.60920998 11.53665955 11.65813195]]
Prediction:  [0]
2020-05-03 17:40:13,481 - werkzeug:_log:113 - INFO:  127.0.0.1 - - [03/May/2020 17:40:13] "POST /predict HTTP/1.1" 200 -
 Predict features:  [[12.59645277 13.17329123 13.91231828 15.80360728 16.36178987]]
Prediction:  [0]
2020-05-03 17:40:15,958 - werkzeug:_log:113 - INFO:  127.0.0.1 - - [03/May/2020 17:40:15] "POST /predict HTTP/1.1" 200 -
 Predict features:  [[10.66335637  9.58866665  8.88145877 10.60920998 11.53665955]]
Prediction:  [0]
2020-05-03 17:40:18,549 - werkzeug:_log:113 - INFO:  127.0.0.1 - - [03/May/2020 17:40:18] "POST /predict HTTP/1.1" 200 -
 Predict features:  [[11.21156663 12.59645277 13.17329123 13.91231828 15.80360728]]
Prediction:  [0]
2020-05-03 17:40:20,601 - werkzeug:_log:113 - INFO:  127.0.0.1 - - [03/May/2020 17:40:20] "POST /predict HTTP/1.1" 200 -

```
