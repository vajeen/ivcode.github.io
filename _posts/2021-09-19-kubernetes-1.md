---
layout: post
title: Kubernetes-01
subtitle: Code 
categories: Kubernetes
tags: [kubernetes, youtube, tutorial]
---

## Nginx configuration

*Note: Default location /etc/nginx/nginx.cong.*

    load_module '/usr/lib64/nginx/modules/ngx_stream_module.so';

    worker_processes 4;
    worker_rlimit_nofile 40000;

    events {
        worker_connections 8192;
    }

    stream {
        upstream k3snodes {
            least_conn;
            server 192.168.100.51:6443 max_fails=3 fail_timeout=5s;
            server 192.168.100.52:6443 max_fails=3 fail_timeout=5s;
            server 192.168.100.53:6443 max_fails=3 fail_timeout=5s;
        }
        server {
            listen 6443;
            proxy_pass k3snodes;
        }
    }

## Install k3sup
    curl -sLS https://get.k3sup.dev | sh -
    install k3sup /usr/local/bin/

## Install kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

## Install First master
    k3sup install \
    	--host=node1 \
    	--user=root \
    	--cluster \
    	--tls-san 192.168.100.50 \
     	--k3s-extra-args="--node-taint node-role.kubernetes.io/master=true:NoSchedule"

*Note: if you want a specific kubernetes version you can use --k3s-version=v1.19.4+k3s1*    

## Join master nodes
    k3sup join \
    	--host=node2 \
    	--server-user=root \
    	--server-host=192.168.100.51 \
    	--user=root \
    	--server \
    	--k3s-extra-args="--node-taint node-role.kubernetes.io/master=true:NoSchedule"

## Join worker nodes
    k3sup join \
    	--host=worker2 \
    	--server-user=root \
    	--server-host=192.168.100.51 \
    	--user=root

## testdeployment.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: mysite
    labels:
        app: mysite
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: mysite
    template:
        metadata:
        labels:
            app: mysite
        spec:
        containers:
            - name: mysite
            image: kellygriffin/hello:v1
            ports:
                - containerPort: 80
        tolerations:
        - key: "node.kubernetes.io/unreachable"
            operator: "Exists"
            effect: "NoExecute"
            tolerationSeconds: 2
        - key: "node.kubernetes.io/not-ready"
            operator: "Exists"
            effect: "NoExecute"
            tolerationSeconds: 2

## testdeplpyment-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: mysite-service
    spec:
    type: NodePort
    selector:
        app: mysite
    ports:
        - protocol: TCP
        port: 8080
        targetPort: 80
        nodePort: 30000