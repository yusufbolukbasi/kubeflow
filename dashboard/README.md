# Kubernetes Dashboard

## Introduction

Kubernetes Dashboard is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.

## Getting Started

This documentation assumes that already a kubernetes cluster is set and have access wih the cluster admin permissions.

## Installation Steps 
* To create the dashboard required components as pods, secret, services, service account etc. run this command at bash 
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
* To add this Cluster Role Binding to the cluster run to give permisson dashboard pods to see other pods and their activities 
```
kubectl apply -f cluster_role_binding.yaml
```
* To forward the kubernetes-dashboard service port run this below code 
```
kubectl port-forward -n kubernetes-dashboard services/kubernetes-dashboard <port-to-forward>:443 --address=<ip-address>
```
Kubectl will make Dashboard available at https://\<ip-address\>:\<port-to-forward\>/

**IMPORTANT:** [Kubernetes dashboard](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md) does not support login with "http://..".
If your login view displays below error, this means that you are trying to log in over HTTP, and it has been disabled for the security reasons.

* To login the dashboard run this below command to create token for service.
```
kubectl create token -n kubernetes-dashboard kubernetes-dashboard
```
Then copy the token and paste it into the Enter token field on the opened login screen
