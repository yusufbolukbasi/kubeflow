# Kubernetes Monitoring

## Getting Started

This documentation assumes that already a kubernetes cluster is set and have access wih the cluster admin permissions.

## Installation Steps 
* To run the node-exportre, kube-state-metrics- grafana and prometheus yamls please run the below code 
```
kubectl create -k .
```
* To forward the Grafana service port run this below code 
```
kubectl port-forward -n monitoring  services/grafana <port-to-forward\>:3000 --address=<ip-address\>
```
Kubectl will make Grafana available at https://\<ip-address\>:\<port-to-forward\>/

The Grafana UI can be accessed with the default credentials :
username: admin
password: admin
