---
title: "Beware of Identical Service Selectors"
date: 2018-10-08
draft: false
tags: ["selectors", "labels", "services", "kubernetes"]
---
Label selectors on your Kubernetes Services should generally be unique.
The Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/) docs
outline occasions when selectors might not apply.
But if you use selectors, you want each Service to select a unique backend.
Otherwise, your traffic may not route properly

### Case Study
50% of routine HTTP requests failed to reach the target backend inside of a Kubernetes cluster.

HTTP requests for resources like` http://integration-tachyon/images/headshot.jpg` never arrived at the backend
for `integration-tachyon`. 

When an HTTP request *did* make it to a backend, the container responded correctly.

After fruitless efforts focusing on the application, we dumped the Kubernetes Service configuration for the cluster.

```bash
$ kubectl get service -o wide
NAME                 TYPE       CLUSTER-IP     PORT(S)       AGE  SELECTOR
integration-graphql  ClusterIP  10.35.245.134  4000/TCP      1d   app=graphql,env=test
integration-tachyon  NodePort   10.35.255.87   80:32169/TCP  1d   app=tachyon,env=test
wptest-graphql       ClusterIP  10.35.253.123  4000/TCP      1d   app=graphql,env=test
wptest-tachyon       NodePort   10.35.246.140  80:31311/TCP  1d   app=tachyon,env=test
``` 

The problem is apparent in the `SELECTOR` column: we had different services with identical selectors. 

There were two pairs of identical selectors. 
Hence, 50% of the HTTP requests did not make it to the correct backend. 
Half of the traffic arrived at another instance of the application (our logs verified this).

The fix was to update our labels and selectors to be sufficiently unique.