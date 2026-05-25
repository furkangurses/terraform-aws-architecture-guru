# ☁️ Terraform AWS Infrastructure Mastery

![Terraform](https://img.shields.io/badge/terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Infrastructure as Code](https://img.shields.io/badge/IaC-Enabled-success?style=for-the-badge)

## 📌 Overview

This repository contains professional-grade Infrastructure as Code (IaC) configurations developed alongside the **Pearson Terraform in AWS: From Basics to Guru** specialization. 

The goal of this project is to demonstrate a scalable, modular, and secure approach to provisioning AWS cloud environments. It covers everything from foundational Terraform workflows to complex, multi-region networking architectures (including VPC Peering and Transit Gateways), utilizing industry best practices.

## 🏗️ Architecture & Course Progression

The repository is structured logically to reflect the progression from basic provisioning to advanced infrastructure management:

### Unit 1: Foundations & Core Workflow
- **Core Commands:** Utilization of `init`, `plan`, `apply`, and `destroy`.
- **Compute & Security:** Provisioning EC2 instances, configuring Security Groups, and managing SSH key pairs.
- **Automation:** Implementing **Cloud-Init** for automated bootstrap and system configuration upon instance launch.
- **State Management:** Understanding local vs. remote state files.

### Unit 2: Advanced Networking
- **VPC Design:** Building custom Virtual Private Clouds, public/private subnets, and routing tables.
- **Multi-Region Deployments:** Architecting resilient infrastructure across multiple AWS regions.
- **Connectivity:** Deploying and evaluating **VPC Peering** and **AWS Transit Gateway** for complex network topologies.

### Unit 3: Modularity & Advanced Concepts
- **Variables & Outputs:** Dynamic resource creation using input variables and complex data types.
- **Terraform Modules:** Writing DRY (Don't Repeat Yourself) code by creating and consuming reusable modules.
- **Advanced Expressions:** Utilizing functions, meta-arguments (like `count` and `for_each`), and dynamic blocks.
- **Multi-Cloud Awareness:** High-level implementation strategies for interacting with multiple providers (AWS, Azure, GCP).
