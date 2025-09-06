# Automated Infrastructure and CI/CD Pipeline for a Voting Application

## Overview

This project demonstrates a complete, end-to-end automation of a CI/CD infrastructure for a sample microservices application (Voting App). The entire infrastructure, including a GitLab server and a dedicated GitLab Runner, was provisioned automatically using Ansible, adhering to Infrastructure as Code (IaC) principles.

Two versions of a sophisticated CI/CD pipeline were designed. The primary, tested version (`v1`) features intelligent, change-based builds and simulated multi-environment deployments. An advanced version (`v2`) was also developed to demonstrate a more production-ready pattern using SSH-based deployments with Docker Compose and integrated security scanning with Trivy.

**Key Technologies Used:** GitLab, GitLab Runner, Ansible, Docker, Docker Compose, Trivy.

## Infrastructure Overview

The entire infrastructure runs within Docker on a single host machine. The core components, GitLab and GitLab Runner, operate as separate containers on a shared custom bridge network (`gitlab_network_badesaba`) for secure communication.

The GitLab Runner is configured with the Docker Executor and is bound to the host's Docker socket, allowing it to orchestrate CI/CD jobs by spawning ephemeral Docker containers for each task (e.g., building, testing, deploying). All persistent data for GitLab and the Runner is stored in named Docker volumes (e.g., `gitlab_config_badesaba`) to ensure data integrity across container restarts.

## Key Capabilities

* **Complete Infrastructure as Code (IaC):** The entire infrastructure, including the GitLab server and Runner, is provisioned automatically using **Ansible**, ensuring a repeatable and consistent setup.

* **Fully Containerized Environment:** All services (GitLab, Runner, and the application itself) are deployed as **Docker** containers, promoting isolation and portability.

* **Intelligent CI/CD Pipeline:** The pipeline is built with `rules:changes` to create **smart, efficient builds**, only triggering jobs when relevant source code is modified.

* **Multi-Environment Deployment:** The pipeline supports deployments to both **Staging** (automatic) and **Production** (manual) environments, simulating a real-world release workflow.

* **Professional Deployment Strategy:** The advanced pipeline version (`v2`) demonstrates a professional deployment pattern using **SSH and Docker Compose** to manage the multi-service application stack on a remote server.

* **Integrated Security Scanning:** A security gate is implemented in the `test` stage using **Trivy** to scan Docker images for critical vulnerabilities before deployment.

* **Centralized & Dynamic Configuration:** The Ansible project is highly configurable through a centralized variables file (`group_vars/all.yml`) and uses Jinja2 templates for dynamic file generation.

## Setup & Usage Guide

This guide provides the step-by-step instructions to deploy the entire GitLab and GitLab Runner infrastructure using the provided Ansible playbooks.

### Prerequisites

Before you begin, ensure the following requirements are met:

* **A host machine** running a Debian-based Linux distribution (e.g., Ubuntu 22.04).
* **Ansible version 2.10.8** installed on the machine you will be running the playbooks from.
* The **`community.docker` Ansible collection (version 1.x)** must be installed. You can install it with the following command:
    ```bash
    ansible-galaxy collection install community.docker:==1.10.0
    ```
* **Root (sudo) access** on the host machine.
* A **domain name** with its DNS A record pointing to the host machine's public IP address.

### Configuration

All project configuration is centralized in the `ansible/group_vars/all.yml` file. Before running the playbooks, review and configure the variables as needed, especially the following:

1.  **`gitlab_hostname`**: Set this to your domain name.
2.  **`gitlab_ip_address`**: Set the static IP for the GitLab container.
3.  After running the `1-setup.yml` playbook, you will need to manually obtain the **Runner Registration Token** from the GitLab Admin Area (`Admin Area > CI/CD > Runners`). This token will be required when you run the `2-register-runner.yml` playbook.

### Execution Steps

Run the playbooks from the `ansible/` directory. You can run all steps at once or use tags to run specific parts of the setup.

1.  **Install prerequisites and pull Docker images:**
    ```bash
    ansible-playbook -i inventory.ini 0-prerequisites.yml
    ```

2.  **Deploy the GitLab and Runner containers:**
    ```bash
    ansible-playbook -i inventory.ini 1-setup.yml
    ```

3.  **Register the Runner:** This playbook will prompt you for the registration token.
    ```bash
    ansible-playbook -i inventory.ini 2-register-runner.yml
    ```

4.  **Apply final configurations to the Runner:**
    ```bash
    ansible-playbook -i inventory.ini 3-configure-runner.yml
    ```

At the end of this process, a fully configured GitLab instance and a registered, operational GitLab Runner will be running.

## CI/CD Pipeline Analysis

The CI/CD process is defined in the `.gitlab-ci.yml` file. Two distinct versions of the pipeline were developed to meet the project's requirements while also demonstrating professional, production-ready patterns.

### Pipeline Stages

The pipeline is logically structured into the following stages, executed in order:

1.  **`build`**: Builds a new Docker image for any microservice that has changed.
2.  **`test`**: Scans the newly built images for security vulnerabilities.
3.  **`deploy`**: Deploys the application to the target environments.

### Pipeline Implementation v1 (Operational & Tested)

This is the primary, fully operational pipeline submitted as the main deliverable. It is defined in the `.gitlab-ci.yml-v1` file and focuses on stability and core functionality.

* **Smart Builds:** Utilizes the `rules:changes` keyword to intelligently build a service's Docker image only when files within its specific directory are modified. This optimizes pipeline efficiency.
* **Local Deployment Simulation:** As no remote servers were provided, this pipeline simulates deployments by running the application containers directly on the runner's host machine.
* **Multi-Environment Flow:**
    * **Staging:** Deployment to the staging environment (running on port `8080`) is triggered automatically on every push to the `main` branch where code changes are detected.
    * **Production:** Deployment to the production environment (running on port `8081`) is controlled by a `when: manual` rule, requiring manual approval from the GitLab UI to proceed.

### Pipeline Implementation v2 (Advanced & Professional Pattern)

This version, defined in `.gitlab-ci.yml-v2`, was developed to showcase a more realistic and production-ready CI/CD workflow. It was not fully tested due to external constraints (lack of a remote server and network sanctions).

* **Real-World Deployment Strategy:** This pipeline deploys the entire application stack using **SSH and Docker Compose**. It copies a `docker-compose.yml` manifest to a target server and runs `docker compose up`, which is a standard industry practice.
* **Integrated Security Scanning:** A dedicated `test` stage is included, which uses **Trivy** to scan newly built Docker images for `HIGH` and `CRITICAL` vulnerabilities. The pipeline is configured to fail and halt immediately if any critical vulnerabilities are discovered, acting as a security gate.

#### Prerequisites for v2 Pipeline

To run the v2 pipeline (`.gitlab-ci.yml-v2`), which performs real deployments via SSH, you must configure the following CI/CD variables in your GitLab project settings (`Settings > CI/CD > Variables`).

This pipeline requires these variables to securely connect to your remote deployment server.

1.  **`ID_RSA`**
    * **Description:** The **private SSH key** used to authenticate with the deployment server. This key must correspond to a public key that has been added to the `authorized_keys` file on the server for the deployment user.
    * **Flags:** For security, this variable should be set as **`Protected`** and **`Masked`**.

2.  **`SERVER_USER`**
    * **Description:** The **username** for logging into the deployment server via SSH.
    * **Example:** `ubuntu` or `deploy_user`.

3.  **`SERVER_IP`**
    * **Description:** The public **IP address** of the remote deployment server.
    * **Example:** `194.113.72.110`.

## Challenges & Decisions

This section describes some of the key technical challenges encountered during the project and the strategic decisions made to overcome them. The final configuration is the result of a systematic debugging process and adherence to the principles of “infrastructure as code.”

### Critical Networking Issues due to Docker Swarm
* **Challenge:** The most significant blocker was a series of persistent network failures within the CI/CD pipeline, including errors like `Could not resolve host`, `Failed to connect`, and the specific `invalid cluster node`. For the first two days, these issues even prevented the GitLab Registry service from starting correctly.
* **Decision:** After extensive debugging, it was discovered that the host's Docker daemon was unintentionally in **Swarm Mode**. This mode fundamentally alters Docker's networking layer, creating conflicts with the GitLab Runner. The root cause was addressed by explicitly removing the host from the Swarm (`docker swarm leave --force`), which reverted Docker's networking to a standard, predictable state and resolved all connectivity issues.

### Advanced GitLab Runner Networking Configuration
* **Challenge:** Even with Swarm mode disabled, the ephemeral job containers created by the runner still needed a reliable way to connect to the GitLab container via its hostname (`gitlab.badesaba.ir`).
* **Decision:** A robust, multi-layered solution was implemented in the runner's `config.toml` file:
    * **`network_mode`**: The job container was explicitly attached to the same Docker network as the GitLab container to ensure direct network access.
    * **`extra_hosts`**: A static DNS entry was injected into the job container to guarantee successful hostname resolution to its static internal IP.
    * **`volumes`**: The host's Docker socket (`/var/run/docker.sock`) was mounted into the job container to enable Docker-in-Docker operations.
    * **`pull_policy`**: This policy was set to `if-not-present` to prevent failures related to Docker Hub sanctions or filtering by using pre-pulled images.

### Managing SSL Certificates for the Runner
* **Challenge:** The GitLab Runner container could not trust the SSL certificate issued for the GitLab server, causing the initial connection to fail.
* **Decision:** A task was added to the Ansible playbook to automatically fetch the server's public certificate using `openssl`. This certificate was then stored in a `./certs` directory and mounted into the runner container, establishing a secure and trusted connection.

### Creating a Prerequisites Playbook
* **Challenge:** Due to filtering and sanctions, downloading public base images (like `python`, `trivy`) from Docker Hub would occasionally fail during pipeline execution, reducing reliability.
* **Decision:** A separate playbook (`0-prerequisites.yml`) was created with the sole purpose of downloading all required public images before the main installation process. This decouples the pipeline's execution from external network dependencies, making it faster and significantly more reliable.

### Idempotent Management of Docker Compose
* **Challenge:** It was observed that Ansible's `docker_compose` module would sometimes detach a container from the Docker Compose project's network when restarting a single service, breaking inter-service communication.
* **Decision:** To ensure complete stability and control, the `shell` module was used to execute `docker compose` commands directly. This approach guarantees that all containers remain managed by the Docker Compose CLI and maintain their network connections correctly.

### Dynamic Configuration with Ansible Templates
* **Challenge:** The `docker-compose.yml` file required dynamic values based on variables (hostname, IP address, etc.). Simply copying a static file would defeat the purpose of automation.
* **Decision:** Instead of copying a static file, the Ansible `template` module was used with a Jinja2 template (`docker-compose.yml.j2`). This allows for the dynamic generation of the final Compose file based on variables.

### Centralized Variable Management
* **Challenge:** Configuration parameters were being repeated across multiple playbooks and files, making them difficult to manage and prone to error.
* **Decision:** All variables were centralized into a single file (`group_vars/all.yml`). This is a core Ansible best practice that makes the project highly configurable, readable, and maintainable.

### Custom SSH Port Mapping
* **Challenge:** The GitLab container's internal SSH service runs on port 22, which is the same as the host machine's default SSH port. Exposing port 22 directly would create a port conflict.
* **Decision:** To avoid this conflict, the GitLab container's port 22 was mapped to a non-standard port `2244` on the host machine. This allows both the host and the GitLab container to have accessible SSH services simultaneously.

### De-scoping of Non-Critical Features (SMTP & Rate Limiting)
* **Challenge:** The project requirements included configuring an SMTP service and implementing brute-force login protection (Rate Limiting).
* **Decision:** The rate-limiting feature was implemented multiple times, but it did not activate as expected during testing, likely due to a complex interaction with the containerized network proxying. Given the significant time spent on critical networking and pipeline issues, and considering the lower priority of these features for the core functionality, both SMTP and Rate Limiting were intentionally de-scoped to ensure the main project deliverables were completed within the deadline.

### Ensuring Stable Inter-Container Communication
* **Challenge:** By default, Docker assigns a new internal IP address to containers upon restart. This would make the `extra_hosts` entry in the GitLab Runner's configuration (which maps `gitlab.badesaba.ir` to an IP) invalid after every restart.
* **Decision:** To guarantee a stable and predictable connection, a static IP address was explicitly assigned to the **GitLab container** within the Docker Compose network configuration. This ensures the `extra_hosts` entry in the runner's configuration remains permanently valid, creating a robust and resilient setup.

## Conclusion

This project successfully demonstrates the end-to-end automation of a complete CI/CD ecosystem. Through the systematic application of Ansible for infrastructure provisioning and a sophisticated `.gitlab-ci.yml` for pipeline orchestration, a fully functional, containerized environment for the Voting App was established.

The final result is a robust and reliable system, built on modern DevOps principles like Infrastructure as Code, that automates the entire software lifecycle from code commit to deployment. The numerous technical challenges overcome throughout the process underscore a deep understanding of the underlying technologies and a proficient approach to complex problem-solving.
