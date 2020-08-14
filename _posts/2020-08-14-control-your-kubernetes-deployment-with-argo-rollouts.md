---
layout: post
title: Control your kubernetes deployment with argo rollouts
subtitle:
tags: [logs]
---

Kubernetes is a fantastic tool, it makes workload deployment really easier but have limitations when you want advance scheduling options.

Before going further, you'll need to read [https://github.com/ContainerSolutions/k8s-deployment-strategies](https://github.com/ContainerSolutions/k8s-deployment-strategies). It will provide you a comprehensive explanation on available options for native kubernetes deployment.

We are going to explore how we can manage canary and blue-green deployment using [argo rollouts](https://argoproj.github.io/argo-rollouts/).

![Argo Logo](https://argoproj.github.io/argo-rollouts/assets/logo.png)


## Concept and challenge

Kubernetes out of the box has an update strategy set to "rollingUpdate" (or ramped upgrade according to the link above), which means it's gonna update a pod one by one by making sure the new pod is ready (see [readiness](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/))

But what if you need more control on your deployment? You're gonna need to add additional tools.

I'll explain how to make a canary and blue green deployment using argo rollouts.

Note that Argo rollouts has some advanced feature like "Analysis" which is a way to test your application during the upgrade via a prometheus query and make automatic decisions according to the output.

## Recall some definitions

#### Canary deployment 

is when you upgrade a small subset of an app, and allow a small traffic on it (eg: 5% of the total traffic)

#### Blue green deployment

is when you have two version of a same application running side by side. Blue is the live environment, green is the idle environment.
By doing that, you can test the new application behaviour, validate everything is ok, and switch all the traffic to the green environment when ready. 

## Argo rollouts introduction

![Argo canary](https://media2.giphy.com/media/Y2zXL1wpgQtSyVifBK/giphy.gif)


### Installation

Argo rollout is deployed on your cluster with a controller and a set of CRD. Install is pretty simple, it's only two commands.

```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
```

### Management

kubectl will work as usual. For convenience, they also provide a [CLI](https://argoproj.github.io/argo-rollouts/features/kubectl-plugin/) which interface with kubectl.

```
kubectl argo rollouts list rollouts 
```

### Convert existing manifest

It's easy to convert you actual yaml manifests, by changing three fields:

- Replacing the apiVersion from apps/v1 to argoproj.io/v1alpha1
- Replacing the kind from Deployment to Rollout
- Replacing the deployment strategy with a blue-green or canary strategy

## Argo rollouts deployment

Note that canary and blue green deployment can be perform with kubernetes without additional tool. But it has to done in multiple steps and manually.

- [Canary upgrade using native kubernetes](https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary/native)

- [Blue-green upgrade using native kubernetes](https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/blue-green/single-service)

Argo provide an integrated flow to perform these actions, plus add more capabilities that we'll see in the next sections


## Deploy a demo app and upgrade it

The app is written an http server written in go and which output the pod name and version of the app.

We will deploy the v1 of the app, then the v2 using both canary and blue green deployment. Output version is defined by 'VERSION' environment variable in the yml file.

```
Host: my-app-5ff5499f6b-bjrvt, Version: v1.0.
```

### Using Canary

Argo provide these capabilities for canary:

- fine grained control: You can define diffent batch during the whole upgrade (eg: upgrade only 20%, then 40%, then the rest of the pods)
- speed control: you can wait a specific amount of time between these batches, and also put a wait for a human validation

For simplicity, I've put the rollout (see it like a 'deployment kind' replacement) and service in the same file.

The rollout below define those steps:

- deploy 20% of the whole replicas (set to 10)
- Pause the deployment, until a user resume the deployment
- Continue the with 40%, Wait for 10s.
- Continue with 60%, wait for 10s
- Continue with 80%, wait for 10s
- Continue to reach 100% (this step is invisible and is actually after the last pause step)



```
# Content of rollout.yml v1

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {}
      - setWeight: 40
      - pause: {duration: 10}
      - setWeight: 60
      - pause: {duration: 10}
      - setWeight: 80
      - pause: {duration: 10}
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v1.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5

---

apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: my-app
```

I can now apply the manifest, and it should create the rollout object and the service as well. When you apply the manifest for the first time, the rollout will create all the replicas defined in the yaml file.

```
kubectl apply -f rollout.yml


kubectl get rollout
NAME     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE
my-app   10        10        10           10
```

Here the graphical reprentation using [kubeview](https://github.com/benc-uk/kubeview). This is totally optionnal, I just spin it up for fun. 

![Kubeview screenshot](https://github.com/ptran32/ptran32.github.io/blob/master/_posts/img/07-argorollouts.png?raw=true)



Change the label version to v2.0.0.0 and env variable VERSION to v2.0.0. 

```
# Content of rollout.yml v1 (truncated)

...
  template:
    metadata:
      labels:
        app: my-app
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v2.0.0
```

```
kubectl apply -f rollout.yml

```

https://media2.giphy.com/media/Y2zXL1wpgQtSyVifBK/giphy.gif


## Conclusion

This is a sample of what you can do with log collection in a kubernetes cluster, there's actually a lot of engineering behind like define a standard for logs coming from different formats across your platform or high availability setup.

Peace out ;)

## Useful link

[Fluentd vs Fluentbit ](https://fluentbit.io/documentation/0.8/about/fluentd_and_fluentbit.html)
