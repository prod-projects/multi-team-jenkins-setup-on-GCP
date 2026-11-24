# Multi-Team Jenkins Architecture on Google Kubernetes Engine (GKE)

## Overview

This document outlines the architecture, deployment steps, networking configuration, and operational guidelines for running isolated, team-dedicated Jenkins instances on **Google Kubernetes Engine (GKE) Autopilot**.

Instead of maintaining a single monolithic Jenkins controller shared across the organization—which often leads to plugin conflicts, resource contention ("noisy neighbors"), and complex RBAC—this setup provisions independent Jenkins controllers for each engineering vertical (**Product**, **Service**, and **R&D**). All provisioning and configuration are performed natively through the **Google Cloud Console** without requiring CLI or Infrastructure-as-Code (IaC) tooling.

## Architecture Summary

### 1. Environment & Cluster Specifications

* **Cloud Provider:** Google Cloud Platform (GCP)

* **Cluster Type:** GKE Autopilot (`jenkins-cluster`)

* **Region:** `<gcp-region>` (e.g., `asia-south1`)

* **Namespace:** `default`

### 2. Workload & Service Isolation

Each team operates on an independent Kubernetes Deployment and external LoadBalancer Service:

| Team | Deployment Name | Container Port | Service Port | Static External IP | Accessible Hostname (`sslip.io`) |
| ----- | ----- | ----- | ----- | ----- | ----- |
| **Product** | `jenkins-product` | `8080` | `80` -> `8080` | `<PRODUCT_STATIC_IP>` | `http://jenkins-product.<PRODUCT_STATIC_IP>.sslip.io` |
| **Service** | `jenkins-service` | `8080` | `80` -> `8080` | `<SERVICE_STATIC_IP>` | `http://jenkins-service.<SERVICE_STATIC_IP>.sslip.io` |
| **R&D** | `jenkins-randd` | `8080` | `80` -> `8080` | `<RANDD_STATIC_IP>` | `http://jenkins-randd.<RANDD_STATIC_IP>.sslip.io` |

## Step-by-Step Implementation Guide (Google Cloud Console)

### Phase 1: Cluster Provisioning

1. Navigate to **Kubernetes Engine > Clusters** in the Google Cloud Console.

2. Click **Create** and select **GKE Autopilot**.

3. Name the cluster `jenkins-cluster` and select your target GCP region.

4. Retain default networking settings and deploy the cluster.

### Phase 2: Deploying Team Workloads

Repeat the following steps for `jenkins-product`, `jenkins-service`, and `jenkins-randd`:

1. Navigate to **Kubernetes Engine > Workloads** and click **Deploy**.

2. Select **Existing container image** and specify the official LTS image: `jenkins/jenkins:lts`.

3. Set the **Application name** (e.g., `jenkins-product`) and target cluster (`jenkins-cluster`).

4. Ensure the replica count is set to `1` (single controller pod per team) and click **Deploy**.

### Phase 3: Exposing Services & Port Mapping

By default, Jenkins listens on port `8080` inside the container. Exposing it via a standard HTTP port (`80`) requires explicit port forwarding:

1. Open the deployed workload under **Kubernetes Engine > Workloads**.

2. Click **Actions > Expose** (or navigate to **Gateways, Services & Ingress**).

3. Configure the **Service type** as **Load balancer**.

4. Set the port mapping:

   * **Port:** `80`

   * **Target port:** `8080`

   * **Protocol:** `TCP`

5. Click **Expose** and note the assigned **External endpoint** IP address.

> **Verification Note:** If accessing the external endpoint returns `ERR_CONNECTION_REFUSED`, inspect the Service's **Ports** section and confirm **Target port** is explicitly set to `8080` rather than `80`.

### Phase 4: Static IP Reservation & DNS Mapping

To prevent IP changes across service updates and provide developer-friendly URLs without managing custom DNS zones initially:

1. Navigate to **VPC network > IP addresses**.

2. Locate the ephemeral external IP assigned to the team's LoadBalancer in your cluster's region.

3. Click the **Actions (`⋮`)** menu on that row and select **Promote to static IP address**.

4. Assign a descriptive name (e.g., `jenkins-product-ip`, `jenkins-service-ip`, `jenkins-randd-ip`).

5. Construct the wildcard DNS URL using `sslip.io`:

   ```text
   http://<team-deployment-name>.<static-external-ip>.sslip.io
   ```

### Phase 5: Initial Jenkins Unlock & Handoff

1. Navigate to **Kubernetes Engine > Workloads** and select the team's deployment.

2. Under **Managed pods**, click the active pod and open the **Logs** tab.

3. Locate the startup log block containing the 32-character **Initial Admin Password**.

4. Open the team's `sslip.io` URL in a browser, paste the password, and select **Install suggested plugins**.

5. Create a dedicated admin account for the respective team lead and share the endpoint and credentials securely (never commit credentials, tokens, or secrets to public repositories).

## Roadmap & Next Steps

* [ ] **Complete R&D Environment:** Promote the external IP for `jenkins-randd` to static, map `jenkins-randd.<RANDD_STATIC_IP>.sslip.io`, and complete initial plugin setup.

* [ ] **HTTPS / TLS Termination:** Configure a GKE **Ingress** resource with **Google-managed SSL certificates** (or migrate to an organization-owned domain) to serve all instances over `https://`.

* [ ] **Persistent Storage Verification:** Ensure PersistentVolumeClaims (PVCs) are bound to `/var/jenkins_home` so job configurations and build histories persist across pod restarts.

* [ ] **GitHub Organization Integration:** Install GitHub Branch Source plugins on each controller and configure webhooks for team repositories.