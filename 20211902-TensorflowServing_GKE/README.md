# Tensorflow Serving on Kubernetes

## Part I: Tensorflow Serving docker

First, we will learn about Tensorflow Serving, what is, how to run it and some basic configurations.

### Introduction    

TensorFlow Serving is a flexible, high-performance serving system for machine learning models, designed for production environments. TensorFlow Serving makes it easy to deploy new algorithms and experiments, while keeping the same server architecture and APIs. TensorFlow Serving provides out-of-the-box integration with TensorFlow models, but can be easily extended to serve other types of models and data.

To note a few features:

-   Can serve multiple models, or multiple versions of the same model
    simultaneously
-   Exposes both gRPC as well as HTTP inference endpoints
-   Allows deployment of new model versions without changing any client code
-   Supports canarying new versions and A/B testing experimental models
-   Adds minimal latency to inference time due to efficient, low-overhead
    implementation
-   Features a scheduler that groups individual inference requests into batches
    for joint execution on GPU, with configurable latency controls
-   Supports many *servables*: Tensorflow models, embeddings, vocabularies,
    feature transformations and even non-Tensorflow-based machine learning
    models


For more information: [Tensorflow Serving](https://github.com/tensorflow/serving)

### Running on docker 

Extracted from Tensorflow Serving documentation:

```bash
# Download the TensorFlow Serving Docker image and repo
docker pull tensorflow/serving

git clone https://github.com/tensorflow/serving
# Location of demo models
TESTDATA="$(pwd)/serving/tensorflow_serving/servables/tensorflow/testdata"

# Start TensorFlow Serving container and open the REST API port
docker run -t --rm -p 8501:8501 \
    -v "$TESTDATA/saved_model_half_plus_two_cpu:/models/half_plus_two" \
    -e MODEL_NAME=half_plus_two \
    tensorflow/serving &

# Query the model using the predict API
curl -d '{"instances": [1.0, 2.0, 5.0]}' \
    -X POST http://localhost:8501/v1/models/half_plus_two:predict

# Returns => { "predictions": [2.5, 3.0, 4.5] }
```

## Part II-a: Create Image to run on GKE

### Commit image for deployment

First we run a serving image as a daemon:

```shell
docker run -d --name serving_base tensorflow/serving
```

Next, we copy the half_plus_two model data to the container's model folder:

```shell
docker cp "$TESTDATA/saved_model_half_plus_two_cpu"  serving_base:/models/half_plus_two
```

Finally we commit the container to serving the ResNet model:

```shell
docker commit --change "ENV MODEL_NAME half_plus_two" serving_base halfplustwo_serving
```

Now let's stop the serving base container

```shell
docker kill serving_base
docker rm serving_base
```

### Start the server

Now let's start the container with the half_plus_two model so it's ready for serving,
exposing the gRPC port 8501:

```shell
docker run -p 8501:8501 -t halfplustwo_serving &
```

### Query the model using the predict API:
```
curl -d '{"instances": [1.0, 2.0, 5.0]}' \
    -X POST http://localhost:8501/v1/models/half_plus_two:predict
```

### Upload the Docker image

Let's now push our image to the
[Google Container Registry](https://cloud.google.com/container-registry/docs/)
so that we can run it on Google Cloud Platform.

First we tag the `halfplustwo_serving` image using the Container Registry
format and our project name,

```shell
export PROJECT_ID=""
docker tag halfplustwo_serving gcr.io/$PROJECT_ID/tensorflow-serving/halfplustwo
```

Next, we configure Docker to use gcloud as a credential helper:

```shell
gcloud auth configure-docker
```

Next we push the image to the Registry,

```shell
docker push gcr.io/$PROJECT_ID/tensorflow-serving/halfplustwo
```

## Part II-b: Deploy on GKE

In this section we use the container image built in previous part to deploy a serving
cluster with [Kubernetes](http://kubernetes.io) in the
[Google Cloud Platform](http://cloud.google.com).


### GCloud project login

```shell
gcloud auth login --project $PROJECT_ID
```

### Create a container cluster

First we create a
[Google Kubernetes Engine](https://cloud.google.com/container-engine/) cluster
for service deployment.

```shell
$ gcloud container clusters create halfplustwo-serving-cluster --num-nodes 3 --machine-type=n1-standard-1 --zone=us-east1-b
```


Set the default cluster for gcloud container command and pass cluster
credentials to [kubectl](http://kubernetes.io/docs/user-guide/kubectl-overview/).

```shell
gcloud config set container/cluster halfplustwo-serving-cluster
gcloud container clusters get-credentials halfplustwo-serving-cluster
```

### Create Kubernetes Deployment and Service

The deployment consists of 2 replicas of `halfplustwo_inference` server controlled by
a [Kubernetes Deployment](http://kubernetes.io/docs/user-guide/deployments/).
The replicas are exposed externally by a
[Kubernetes Service](http://kubernetes.io/docs/user-guide/services/) along with
an
[External Load Balancer](http://kubernetes.io/docs/user-guide/load-balancer/).

We create them using the example Kubernetes config
[resnet_k8s.yaml](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/example/resnet_k8s.yaml) and modifying to get a halfplustwo_k8s.yaml.

```shell
kubectl create -f halfplustwo_k8s.yaml
```

With output:

```console
deployment "halfplustwo-deployment" created
service "halfplustwo-service" created
```

To view status of the deployment and pods:

```console
$ kubectl get deployments
NAME                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
halfplustwo-deployment    3         3         3            3           5s
```

```console
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
halfplustwo-deployment-bbcbc   1/1       Running   0          10s
halfplustwo-deployment-cj6l2   1/1       Running   0          10s
halfplustwo-deployment-t1uep   1/1       Running   0          10s
```

To view status of the service:

```console
$ kubectl get services
NAME                    CLUSTER-IP       EXTERNAL-IP       PORT(S)     AGE
halfplustwo-service       10.239.240.227   104.155.184.157   8501/TCP    1m
```

It can take a while for everything to be up and running.

```console
$ kubectl describe service halfplustwo-service
Name:           halfplustwo-service
Namespace:      default
Labels:         run=halfplustwo-service
Selector:       run=halfplustwo-service
Type:           LoadBalancer
IP:         10.239.240.227
LoadBalancer Ingress:   104.155.184.157
Port:           <unset> 8500/TCP
NodePort:       <unset> 30334/TCP
Endpoints:      <none>
Session Affinity:   None
Events:
  FirstSeen LastSeen    Count   From            SubobjectPath   Type        Reason      Message
  --------- --------    -----   ----            -------------   --------    ------      -------
  1m        1m      1   {service-controller }           Normal      CreatingLoadBalancer    Creating load balancer
  1m        1m      1   {service-controller }           Normal      CreatedLoadBalancer Created load balancer
```

The service external IP address is listed next to LoadBalancer Ingress.

To add more pods (go from 2 to 3):

```console
$ kubectl scale deployment halfplustwo-deployment --replicas 3
```

```console
$ kubectl get deplyment halfplustwo-deployment
```

```console
NAME                                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
halfplustwo-deployment                3         3         3            3           15m
```