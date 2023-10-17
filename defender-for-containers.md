# Defender for Containers


## Introduction

Microsoft Defender for Containers is a cloud-native solution that helps you secure your containers and their applications. It protects your Kubernetes clusters from misconfigurations, vulnerabilities, and threats, whether they are running on Azure, AWS, GCP, or on-premises. With Microsoft Defender for Containers, you can:

- Harden your environment by continuously monitoring your cluster configurations and applying best practices.
- Assess your container images for known vulnerabilities and get recommendations to fix them.
- Protect your nodes and clusters from run-time attacks and get alerts for suspicious activities.
- Discover your data plane components and get insights into your Kubernetes and environment configuration.

Microsoft Defender for Containers is part of Microsoft Defender for Cloud, a comprehensive security solution that covers your cloud workloads and hybrid environments. You can enable it easily from the Azure portal and start improving your container security today. Learn more about Microsoft Defender for Containers from [this article](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-introduction).

## Prerequisites

Make sure you have enabled Microsoft Defender for Container before you follow these steps. You can find out how to do this in the [Enable Microsoft Defender for Containers](https://github.com/pelithne/AKS-security-basics/blob/main/Identity-and-access-mgmt.md#23-enable-microsoft-defender-for-containers) section. 

During this activity you will:

* Scan Container images for vulnerabilities.
* Threat hunting for vulnerabilities in your AKS cluster.

Import the metasploit vulnerability emulator docker image from Docker Hub to your Azure container registry.

## Import Vulnerable images to Container Registry

````bash
az acr import --name $ACRNAME --source docker.io/vulnerables/metasploit-vulnerability-emulator
````
Verify that the docker image is stored in your container registry.

````bash
az acr repository list --name $ACRNAME
````

## Review Microsoft Defender for Containers Assessments
