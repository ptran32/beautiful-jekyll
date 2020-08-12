---
layout: post
title: Continuous k8s deployment with ArgoCD
subtitle:
tags: [containers]
---

My first post, thanks to Covid19, I've got more times to do what I like, testing new stuffs. I've decided to post small posts on what I'm interested right now.
I would like to write about an amazing tool in the k8s world: ArgoCD
I started to look at this tool earlier this year for my personal knowledge, and I wanted to share this post, so some people could have an overview of the tool, and test it. Enjoy :)


# Continuous k8s deployment with ArgoCD

![Image of argo-ui-schema](https://github.com/ptran32/argocd-deployment/blob/master/img/argo-ui-schema.png?raw=true)

When it comes to continuous deployment on a kubernetes cluster, you have a plenty of choices.
Google could help you to find out, but the user feedback, experience and maturity of the project will be determinent for you to choose the best tool for the right job.

Mentionning some of them:
- spinnaker
- flux
- codefresh
- gitlab
- jenkins
- argocd

The two I'm confident to work with and which require the less amount of work are: flux and argocd.

Both project members are actually working to together on a new common project: https://github.com/argoproj/gitops-engine So keep an eye on it ;)

I found that Argo is a more advanced product over Flux for these reasons :

- Recently joins the CNCF
- Can handle multiple file type: jsonnet, kustomize, helm, and yaml
- More detailed documentation
- Can use argo workflow to orchestrate multiple k8s jobs (handling with failure, retry and jobs dependencies)
- Can use argo rollout to manage a canary or blue green deployment.
- Nice and clean UI with cool features: upgrade, rollback, architecture topology, diff between two versions
- Integreated SSO (ldap, github, OIDC, Oauth2 etc.)

ArgoCD can enforce the changes if the code repo is different from what is actually on the k8s cluster, which is great to avoid any drift.


In a future, I would like to continue this post and write something for argo rollouts, or maybe istio. We'll see ;)


# Project using argoCD

This project will deploy one application on minikube using argoCD and will be accessible via ingress endpoint.

You can follow the instructions here:

[https://github.com/ptran32/argocd-deployment](https://github.com/ptran32/argocd-deployment)