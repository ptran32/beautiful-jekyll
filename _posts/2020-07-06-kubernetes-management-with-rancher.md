---
layout: post
title: Kubernetes management with Rancher
subtitle:
tags: [containers]
---

Kubernetes deployment is "easy" with all tools available, but it's difficult to manage, Rancher can help you to deploy and manage a production ready kubernetes platform.

![Rancher Logo](https://rancher.com/img/brand-guidelines/assets/logos/png/color/rancher-logo-stacked-color.png)

Kubernetes became the number one tool everyone wants to use, and many companies are just not ready to switch to it, here's the reasons why:

- Choose the tool only for the hype without evaluate pros and cons
- Don't take the time to train the team
- Devops practices not in place
- Work incrementally and learn from experience and mistakes

There's many tools to deploy a kubernetes cluster, from bare metal to cloud providers: kubeadm, kubespreay, kops, ecs, eks, aks, fargate.
 
But I didn't found any tool that includes all features a company could looks for (cluster management, authentification, dashboards, monitoring, support, security).

I actually discovered Rancher few years ago, and at this time, it was a great tool, but still not mature enough for me.

**So, what is rancher ?**

"Rancher is a complete software stack for teams adopting containers. It addresses the operational and security challenges of managing multiple Kubernetes clusters across any infrastructure, while providing DevOps teams with integrated tools for running containerized workloads."

Here are some available features:

- Integration with main cloud providers (AWS, GCP, Azure)
- The possibility to add ANY cluster and let rancher manage it.
- Fast and reliable deployment with RKE (lightweight k8s distribution running in docker containers)
- Fast deployment with K3s (Deploy k8s from a single binary)
- Complete documentation (backups, upgrades, network matrix, installation...)
- Paid support if needed
- Istio integration
- Nginx controller integration
- Clean UI with integrated dashboards, and management features: add cluster, upgrade
- Authentification out of the box with LDAP/Active directory
- Prometheus and Grafana integration in one click (uses metrics-server)
- Fast and reliable deployment
- CIS benchmarch integration
- Terraform provider for rke
- Smooth certification rotation

**How does it works ?**

There's only two ways to install Rancher:

- a standalone docker container
- inside kubernetes (single mode or multi nodes)

The second option is the best approach if you need high availability.

Rancher can install and manage new kubernetes cluster, but how do we deploy the very first cluster ? Well, the good practice is to deploy the first cluster with RKE where we're gonna install Rancher on top of it.

In summary, this is a two steps process:

- Install RKE
- Install Rancher on top of it

## RKE Install

The only thing you need to install RKE is a SSH connection and docker installed on target nodes (minus some minor system pre-configuration). 

Remember, RKE itself is a kubernetes cluster deployed inside docker containers.

RKE uses the roles specified by user to manage the nodes:

- controlplane
- worker
- etcd

It only needs two commands to create the rke cluster:

```
rke config (generate the configuration (node adresses, ssh key to use, images to use, kubernetes version, roles etc.)
```

```
rke up (connect to target hosts using the above config and install rke)
```
It will generate a kubeconfig file that you need in order to connect to the cluster.

## Rancher install

You're gonna need helm 3 installed and deploy rancher with it.

```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.my.org
```

And voila ! You're done ;)

## Conclusion

Rancher is a great product because it does all the heavy lifting of installation and lifecycle management of a complex tool which is kubernetes.

## Useful links

[Rancher documentation](https://rancher.com/docs/rancher/v2.x/en/)

[Certified Rancher Operator: Level 1](https://academy.rancher.com/courses/course-v1:RANCHER+K101+2019/about)



<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://ptran32.github.io/2020-07-06-kubernetes-management-with-rancher/"
  },
  "headline": "Kubernetes management with rancher",
  "description": "Introduction to rancher and how to manage a kubernetes cluster with it",
  "image": "https://rancher.com/img/brand-guidelines/assets/logos/png/color/rancher-logo-stacked-color.png",  
  "author": {
    "@type": "Person",
    "name": "Patrice"
  },  
  "publisher": {
    "@type": "Organization",
    "name": "Patrice",
    "logo": {
      "@type": "ImageObject",
      "url": ""
    }
  },
  "datePublished": "2020-07-06",
  "dateModified": "2020-07-06"
}
</script>