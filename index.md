# Welcome, Single ALB solution in EKS fargate for multiple ingress

Documentation/Steps to enable single alb solution in EKS fargate for multiple ingress/service running different namespace.  Now we have option to reuse an existing ALB instead of creating a new ALB per Ingress.

## Requirement 
- Need to deploy multiple service in kubernetes with single alb. 
- Service/Ingress running on different fargate profile should communicate each other

### What is an Application Load Balancer?
An Application Load Balancer or ALB is a bridge between inbound traffic and several targets (for example several pods for one application). The objective is to have applications with high availability.


### Limitations

  - Each service/ingress running in kubernetes with different namespaces/ fargate profile will create new alb
  - More cost and more public IP needed

### Solution
 We will now going to achieve this with 2 ingress controllers: the ALB ingress controller and the Nginx Ingress controller. 
 #### Why do we have to use 2 ingress controllers? 
 Because if we use the nginx ingress controller, we can not connect it directly to an ALB and if we only use the ALB ingress controller, you will have an ALB instance for every ingress resource in the cluster. 
 
But the requirement is one load balancer all services running in cluster. In the case in which we have both: it is important to know the nginx ingress controller will manage all ingresses resources of your applications in your EKS cluster and the ALB ingress controller will manage the life cycle of the Application Load Balancer instance.

![alt text](https://github.com/rajeshwrn/alb-nginx-controller/blob/master/alb-architecture.webp?raw=true "alb-architecture")

To create alb ingress in aws eks fargate use the below aws doc,
https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html

creating second ingress controller - nginx
Before that, understand currently the eks fargate will not support nginx official helm chart. so we need to do small tweak on the helm values.

- Port type should be NodePort
- allowPrivilegeEscalation should be false to enable the node creation in fargate profile
- Override defalut container ports 80 to 8080 and 443 to 8443
- Traffic policy should be local

### Installation

Use the below helm command to deploy nginx controller in your cluster.

```
helm install nginx-ingress hkube/nginx-ingress --version 1.31.1001 --set-string controller.service.externalTrafficPolicy=Local --set-string controller.service.type=NodePort --set controller.publishService.enabled=true --set serviceAccount.create=true --set rbac.create=true --set-string controller.config.server-tokens=false --set-string controller.config.use-proxy-protocol=false --set-string controller.config.compute-full-forwarded-for=true --set-string controller.config.use-forwarded-headers=true --set controller.metrics.enabled=true --set controller.autoscaling.maxReplicas=1 --set controller.autoscaling.minReplicas=1 --set controller.autoscaling.enabled=true --namespace kube-system -f nginx-values.yaml 
```

Use below value file for helm deployment
> nginx-values.yaml 

```
controller: 
  extraArgs: 
    http-port: 8080 
    https-port: 8443 
  containerPort: 
    http: 8080 
    https: 8443 
  service: 
    ports: 
      http: 80 
      https: 443 
    targetPorts: 
      http: 8080 
      https: 8443 
  image: 
    allowPrivilegeEscalation: false
```

### Connect the ALB to the Nginx Ingress controller

To connect the ALB to the nginx ingress controller, we need to create a kubernetes ingress resource in the namespace kube-system with the following configuration:

> kubectl apply -f alb-ingress-connect-nginx.yaml 

```
apiVersion: extensions/v1beta1 
kind: Ingress 
metadata: 
  annotations: 
    #alb.ingress.kubernetes.io/certificate-arn: <CERTIFICATE_ARN> 
    alb.ingress.kubernetes.io/healthcheck-path: /healthz 
    alb.ingress.kubernetes.io/scheme: internet-facing 
    alb.ingress.kubernetes.io/target-type: ip 
    alb.ingress.kubernetes.io/subnets: <subnets> 
    kubernetes.io/ingress.class: alb  
  name: alb-ingress-connect-nginx 
  namespace: kube-system 
spec: 
  rules: 
    - http: 
        paths: 
          - backend: 
              serviceName: nginx-ingress-controller 
              servicePort: 8080 
            path: /* 
```

Now we are ready with alb it will communicate to the nginx ingress controller.

Validate the changes by accessing the ingress address, it will be the public facing dns address and we can access the services running in cluster this dns.
> kubectl get ingress -n kube-system



Deploy your application with following ingress annonation and service port should be 'ClusterIP'
 ```
 annotations:
    kubernetes.io/ingress.class: "nginx"
```

