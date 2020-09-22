#!/bin/bash

#create container f5-hello-world
kubectl create -f f5-hello-world-deployment.yaml
kubectl create -f f5-hello-world-service.yaml

#Scale replicas

kubectl scale deployment.v1.apps/f5-hello-world --replicas=10