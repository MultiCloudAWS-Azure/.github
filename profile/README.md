# 👋 Welcome

We build and ship software that matters—from full-stack apps to multi-cloud infrastructure.

---

## 🍕 Featured project: Pizza App

A **MERN stack** (MongoDB, Express, React, Node.js) application demonstrating a full menu experience and **multi-cloud deployment** on **AWS** and **Azure**.

### What it does

- **Menu** – Browse pizzas with name, ingredients, price, and photos; add new pizzas via the API.
- **REST API** – Back-end serves `GET` / `POST` `/menu/pizzas` with MongoDB persistence.
- **Modern UI** – React 18 front-end (Header, Menu, Footer, Order) that consumes the API and shows a fallback when the backend is unavailable.

### Tech stack

| Layer        | Technologies |
|-------------|--------------|
| **Front-end** | React 18, Create React App, React Testing Library, Jest |
| **Back-end**  | Node.js, Express, Mongoose, CORS |
| **Database**  | MongoDB (Atlas or self-hosted) |
| **Infrastructure** | Terraform (IaC), Packer (base images), Ansible, HashiCorp Vault (secrets) |

### Repositories / components

- **Front-end** – React app; displays the pizza menu and talks to the API. CI/CD: GitHub Actions trigger **multi-cloud instance refresh** (AWS ASG, Azure VMSS) on push to `master`.
- **Back-end** – Express API and Pizza model; env-based MongoDB URI and optional Vault integration for production.
- **IaC Terraform** – Provisions the same app on **AWS** (VPC, ALB, ASG, EC2 from custom AMI) and **Azure** (VNet, VMSS, Standard Load Balancer). Remote state in S3 + DynamoDB lock; secrets (e.g. GitHub token, MongoDB URI) from Vault KV v2.
- **IaC Packer** – Builds **MERN base images** for AWS (EBS AMI) and Azure (Managed Image) from Ubuntu 22.04, provisioned with Ansible for consistent images across clouds.

### Why it matters

The project shows **end-to-end DevOps**: app code, API design, Infrastructure as Code, secret management, and CI/CD for **both** AWS and Azure from a single org. Use it as a reference for multi-cloud MERN deployments or as a template for similar apps.

---

## What we do

- **Applications** – Full-stack and API-first apps (like the Pizza App) and beyond.
- **Multi-cloud** – Design and run workloads on AWS, Azure, and hybrid setups.
- **DevOps & IaC** – Terraform, Packer, Ansible, Vault, and GitHub Actions.
- **Open collaboration** – Code, docs, and practices we’re happy to share.

## Get in touch

- **Discussions** – [GitHub Discussions](https://github.com/orgs/YOUR_ORG/discussions) for questions and ideas.
- **Issues** – Open an issue in the right repo for bugs or feature requests.
- **Explore** – [Organization repositories](https://github.com/orgs/YOUR_ORG/repositories).

---

*Happy building.* 🍕
