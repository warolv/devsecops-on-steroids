# DevSecOps on Steroids

### A comprehensive tutorial series on essential DevSecOps tools and best practices.

![devsecops-steroids](images/devsecops-steroids.png =400x400)
![Alt Text](https://your-image-url.com/image.png 

### 1. [Secure namespaces in Kubernetes cluster using RBAC](secure-ns-k8s-rbac.md)

In this tutorial, I will show how to secure ‘system‘ namespaces in EKS cluster by user access.

> "List of namespaces with limited access: [kube-system, monitoring]


### 2. [Threat detection with Falco and EKS Audit Logs](falco-k8s-audit-logs.md)

In this tutorial, I will show how to:

* Install Falco to your EKS cluster using helm chart
* Enable EKS audit logs for your cluster
* Detect security events based on EKS audit logs activity.
* Overview of most useful rules for my opinion

### 3. [Threat detection with Falco on EKS and using kernel module driver](falco-syscalls.md)

In previous tutorial I used **Falco** plugin for **K8s Audit Logs**

And for this tutorial I will use **driver** called **Kernel Module**, in order to monitor system events from the kernel and trying to detect malicious behaviors on Linux systems.
