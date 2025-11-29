# Expense Application Deployment with Ansible

## Project Overview

This project automates the deployment of a three-tier web application called "expense". It uses Ansible to provision and configure the necessary components: a MySQL database, a Node.js backend, and an Nginx frontend. The automation handles everything from package installation and configuration to application code deployment.

## Project Structure

The project is organized using standard Ansible best practices, with a clear separation of concerns into roles.

-   `ansible.cfg`: The main Ansible configuration file.
-   `inventory.ini`: Defines the server inventory, grouping hosts into `[mysql]`, `[backend]`, and `[frontend]`. It also contains a security issue by storing a plaintext password.
-   `main.yaml`: A generic playbook that can run any of the roles based on a `component` variable.
-   `mysql.yaml`, `backend.yaml`, `frontend.yaml`: Individual playbooks to run the deployment for each specific component.
-   `roles/`: This directory contains the core logic for the deployment.
    -   `mysql/`: Role to set up the MySQL database.
    -   `backend/`: Role to set up the Node.js backend application.
    -   `frontend/`: Role to set up the Nginx frontend.
    -   `common/`: A shared role for tasks common to both the frontend and backend, such as downloading and extracting the application code.

## Deployment Flow

The deployment is executed by running the corresponding playbook for each tier.

### 1. MySQL (`mysql.yaml`)

-   **Host:** `mysql.learndevops.online`
-   **Tasks:**
    -   Installs Python libraries required for MySQL automation (`PyMySQL`, `cryptography`).
    -   Installs and starts the `mysql-server` package.
    -   Sets the MySQL root password, securely loaded from an Ansible Vault file (`roles/mysql/vars/vault.yaml`).

### 2. Backend (`backend.yaml`)

-   **Host:** `backend.learndevops.online`
-   **Tasks:**
    -   Configures the system to use Node.js 20.
    -   Installs Node.js and MySQL client packages.
    -   Creates a dedicated `expense` user to run the application.
    -   Includes the `common` role to download the backend code from an S3 bucket (`https://expense-builds.s3.us-east-1.amazonaws.com/expense-backend-v2.zip`) and place it in `/app`.
    -   Installs Node.js dependencies using `npm`.
    -   Sets up a `systemd` service to manage the backend process, using the `backend.service.j2` template.
    -   Imports the database schema into the MySQL server by executing the `/app/schema/backend.sql` script.
    -   Enables and restarts the `backend` service.

### 3. Frontend (`frontend.yaml`)

-   **Host:** `frontend.learndevops.online`
-   **Tasks:**
    -   Installs and starts the `nginx` web server.
    -   Includes the `common` role to download the frontend code from an S3 bucket (`https://expense-builds.s3.us-east-1.amazonaws.com/expense-frontend-v2.zip`) and place it in `/usr/share/nginx/html`.
    -   Configures Nginx using the `expense.conf.j2` template. This likely sets up a reverse proxy to forward requests to the backend server.
    -   Restarts the `nginx` service to apply the new configuration.

## How to Run

To deploy a specific component, you would run a command like:

```bash
ansible-playbook -i inventory.ini frontend.yaml
```

Alternatively, using the main playbook:

```bash
ansible-playbook -i inventory.ini main.yaml --extra-vars "component=frontend"
```

## Security Considerations

-   The `inventory.ini` file currently contains a plaintext password for the `ansible_user`. This is a significant security risk.
-   **Ansible Vault is used in this project** to securely manage sensitive data, specifically the MySQL root password, which is loaded from `roles/mysql/vars/vault.yaml`. This is a good security practice as it encrypts sensitive variables, preventing them from being exposed in plain text within the repository.
-   **It is highly recommended** to extend the use of Ansible Vault to encrypt all other sensitive information, such as the `ansible_password` in `inventory.ini`, to enhance the overall security posture of the project.
-   Consider using SSH keys for authentication instead of passwords where possible for increased security.
