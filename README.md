# DevSecOps on Steroids

### A comprehensive tutorial series on essential DevSecOps tools and best practices.
<p align="center">
  <img src="images/devsecops-steroids.png" width="500"/>
</p>

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

### 4. [Moving to Bottlerocket OS for Enhanced Security & Performance](bottlerocket-os.md)

In this tutorial, I will walk you through migrating your Amazon EKS worker nodes to Bottlerocket OS, troubleshooting Bottlerocket OS, and sharing the challenges I encountered during the process.

### 5. [CI/CD Security: Using Checkov to enforce security with terraform](checkov-ci.md)

The purpose of this tutorial is to provide a solid starting point for enforcing security best practices in your Terraform scripts.

I will walk you through the following steps:
* Integrating Checkov into your gitlab pipeline.
* Enabling specific individual security checks.
* Configuring enforcement to fail the pipeline on failed checks.
* Gradually expanding policy recommendations.

### 6. [Provision a Secure AWS ElastiCache Redis Instance Using Terraform](secured-redis.md)

The goal of this tutorial is to deploy AWS ElastiCache Redis with an emphasis on security best practices.

I will guide you through the following Redis security features:

* Encryption at rest and in transit
* Network access restrictions
* IAM integration and authentication for access control
* Creation of three default users for authorization
* Terraform script example for deploying Redis
* Python script example for interacting with Redis

### 7. [Detect CVEs in Built Images Using Trivy and GitLab CI](trivy-cves.md)

This tutorial focuses on detecting CVEs in built images using Trivy and GitLab CI.

You’ll learn how to integrate Trivy into your GitLab pipeline to detect critical vulnerabilities. I’ll provide code examples demonstrating how we use it at Cynerio and show you how to send Slack notifications when a CVE is detected.

### 8. [Automating the weekly audit of publicly accessible S3 buckets](s3-analyzer.md)

Misconfigured S3 buckets have caused major breaches at companies like Capital One, Facebook, and Verizon — all because no one noticed a public bucket until it was too late.

In my latest blog post, I show how to build a fully automated weekly audit pipeline using:

✅ A Kubernetes CronJob

✅ AWS IAM Access Analyzer

✅ Slack alerts for real-time visibility

to automate weekly checks using Kubernetes and IAM Access Analyzer