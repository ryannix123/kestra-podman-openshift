<p align="center">
  <img src="https://kestra.io/logo.svg" alt="Kestra Logo" width="300"/>
</p>

<p align="center">
  <img alt="Kestra version" src="https://img.shields.io/badge/kestra-latest-blueviolet?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0xMiAyQzYuNDggMiAyIDYuNDggMiAxMnM0LjQ4IDEwIDEwIDEwIDEwLTQuNDggMTAtMTBTMTcuNTIgMiAxMiAyeiIvPjwvc3ZnPg=="/>
  <img alt="PostgreSQL" src="https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white"/>
  <img alt="Podman" src="https://img.shields.io/badge/Podman-compatible-892CA0?logo=podman&logoColor=white"/>
  <img alt="License" src="https://img.shields.io/badge/license-Apache%202.0-blue"/>
</p>

# Running Kestra with Podman for Event-Driven Terraform & Ansible

## Why Kestra for Terraform and Ansible?

Kestra is a powerful workflow orchestration platform that provides significant benefits when used with infrastructure-as-code tools like Terraform and Ansible:

- **Unified Orchestration**: Combine Terraform, Ansible, and other DevOps tools into a single declarative workflow, eliminating tool silos.
- **Event-Driven Infrastructure**: Trigger infrastructure changes automatically in response to events (webhooks, file changes, schedules).
- **Improved Visibility**: Track Terraform plans, Ansible playbook executions, logs, and state changes through Kestra's intuitive UI.
- **Error Handling**: Implement sophisticated retry mechanisms, notifications, and conditional workflows.
- **State Management**: Securely pass variables and outputs between tasks (e.g., Terraform outputs to Ansible variables).
- **GitOps Integration**: Synchronize your infrastructure-as-code with Git for version control and auditability.

> 🚀 **Deploying on OpenShift?** See the [OpenShift Helm deployment guide](./kestra-openshift-deployment/README.md).

---

## Prerequisites

- [Podman](https://podman.io/docs/installation) — container management tool
- [podman-compose](https://github.com/containers/podman-compose) — multi-container orchestration
- Terraform and/or Ansible installed locally (for workflow execution)

[Podman Desktop](https://podman-desktop.io/) is recommended for an easier container management experience but is not required.

---

## 1. Create Persistent Storage Volumes

```bash
podman volume create kestra-postgres-data
podman volume create kestra-storage

# Verify
podman volume ls
```

---

## 2. Create a Working Directory

```bash
mkdir -p ~/kestra
cd ~/kestra
```

---

## 3. Create `podman-compose.yml`

```bash
cat > ~/kestra/podman-compose.yml << 'EOF'
version: '3'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: kestra
      POSTGRES_PASSWORD: k3str4
      POSTGRES_DB: kestra
    volumes:
      - kestra-postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kestra"]
      interval: 30s
      timeout: 10s
      retries: 5

  kestra:
    image: kestra/kestra:latest
    pull_policy: always
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8080:8080"
      - "8081:8081"
    environment:
      KESTRA_CONFIGURATION: |
        datasources:
          postgres:
            url: jdbc:postgresql://postgres:5432/kestra
            driverClassName: org.postgresql.Driver
            username: kestra
            password: k3str4
        kestra:
          repository:
            type: postgres
          queue:
            type: postgres
          storage:
            type: local
            local:
              basePath: "/app/storage"
          server:
            gracefulShutdown: PT50S
    volumes:
      - kestra-storage:/app/storage
    command: server standalone

volumes:
  kestra-postgres-data:
    external: true
  kestra-storage:
    external: true
EOF
```

---

## 4. Start Kestra

```bash
cd ~/kestra
podman-compose -f podman-compose.yml up -d
```

Alternatively, using a Podman pod directly:

```bash
podman pod create --name kestra-pod -p 8080:8080 -p 8081:8081

podman run -d --pod kestra-pod \
  --name kestra-postgres \
  -e POSTGRES_USER=kestra \
  -e POSTGRES_PASSWORD=k3str4 \
  -e POSTGRES_DB=kestra \
  -v kestra-postgres-data:/var/lib/postgresql/data \
  postgres:16

podman run -d --pod kestra-pod \
  --name kestra \
  -e KESTRA_CONFIGURATION="datasources:
  postgres:
    url: jdbc:postgresql://localhost:5432/kestra
    driverClassName: org.postgresql.Driver
    username: kestra
    password: k3str4
kestra:
  repository:
    type: postgres
  queue:
    type: postgres
  storage:
    type: local
    local:
      basePath: /app/storage" \
  -v kestra-storage:/app/storage \
  kestra/kestra:latest server standalone
```

---

## 5. Verify Kestra is Running

```bash
# Check status
podman-compose -f podman-compose.yml ps

# Follow logs
podman-compose -f podman-compose.yml logs -f kestra
```

---

## 6. Access the Kestra UI

```
http://localhost:8080
```

On first launch, Kestra presents a setup wizard to create an admin user.

---

## 7. Git Integration

```yaml
id: terraform_from_git
namespace: infrastructure
tasks:
  - id: workspace
    type: io.kestra.plugin.core.flow.WorkingDirectory
    tasks:
      - id: clone_repo
        type: io.kestra.plugin.git.Clone
        url: https://github.com/your-org/terraform-configs
        branch: main

      - id: terraform_init
        type: io.kestra.plugin.scripts.shell.Commands
        commands:
          - terraform init
```

**Authentication options:**

```yaml
# Username/token
- id: clone_repo
  type: io.kestra.plugin.git.Clone
  url: https://github.com/your-org/private-repo
  username: git_username
  password: "{{ secret('GIT_TOKEN') }}"

# SSH key
- id: clone_repo
  type: io.kestra.plugin.git.Clone
  url: git@github.com:your-org/private-repo.git
  privateKey: "{{ secret('SSH_PRIVATE_KEY') }}"
```

**GitOps sync:**

```yaml
id: sync_from_git
namespace: system
tasks:
  - id: sync
    type: io.kestra.plugin.git.SyncFlows
    url: https://github.com/your-org/kestra-flows
    branch: main
    targetNamespace: infrastructure
    delete: true
```

---

## 8. Event-Driven Workflow Examples

### Terraform — Scheduled Execution

```yaml
id: terraform_example
namespace: infrastructure
description: "Event-driven Terraform workflow"

triggers:
  - id: schedule
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 0 * * *"

tasks:
  - id: terraform_init
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - terraform init
    workingDirectory: /path/to/terraform

  - id: terraform_plan
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - terraform plan -out=tfplan
    workingDirectory: /path/to/terraform

  - id: terraform_apply
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - terraform apply -auto-approve tfplan
    workingDirectory: /path/to/terraform
```

### Ansible — Webhook Trigger

```yaml
id: ansible_example
namespace: infrastructure
description: "Event-driven Ansible workflow"

triggers:
  - id: webhook
    type: io.kestra.plugin.core.trigger.Webhook

tasks:
  - id: run_playbook
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - ansible-playbook -i inventory.yml playbook.yml
    workingDirectory: /path/to/ansible
```

### Conditional Dispatch

```yaml
tasks:
  - id: check_condition
    type: io.kestra.plugin.core.flow.Switch
    value: "{{ trigger.event.type }}"
    cases:
      INFRASTRUCTURE_UPDATE:
        - id: run_terraform
          type: io.kestra.plugin.scripts.shell.Commands
          commands:
            - terraform apply
      CONFIGURATION_UPDATE:
        - id: run_ansible
          type: io.kestra.plugin.scripts.shell.Commands
          commands:
            - ansible-playbook playbook.yml
```

---

## 9. Data Management

### Backup

```bash
# PostgreSQL data
podman run --rm \
  -v kestra-postgres-data:/data \
  -v $(pwd):/backup \
  alpine tar -czvf /backup/postgres-backup.tar.gz /data

# Kestra storage
podman run --rm \
  -v kestra-storage:/data \
  -v $(pwd):/backup \
  alpine tar -czvf /backup/kestra-storage-backup.tar.gz /data
```

### Restore

```bash
podman run --rm \
  -v kestra-postgres-data:/data \
  -v $(pwd):/backup \
  alpine sh -c "rm -rf /data/* && tar -xzvf /backup/postgres-backup.tar.gz -C /"
```

### Copy files into a running container

```bash
podman cp ~/path/to/file kestra:/app/storage/
```

---

## 10. Upgrading Kestra

```bash
cd ~/kestra
podman-compose pull
podman-compose down
podman-compose up -d
```

---

## 11. Stopping Kestra

```bash
podman-compose -f podman-compose.yml down
# or
podman pod stop kestra-pod && podman pod rm kestra-pod
```

---

## 12. Troubleshooting

| Symptom | Check |
|---|---|
| Container won't start | `podman logs kestra` |
| Database connection errors | `podman-compose ps` — is postgres healthy? |
| Port 8080 already in use | `lsof -i :8080` and kill conflicting process |
| Volume permission errors | `podman volume inspect kestra-storage` — check SELinux context |
| Podman version mismatch (macOS) | `podman machine info` — client and VM versions must match |

---

## 13. Security Considerations

- Change default passwords before any internet-exposed deployment
- Use Kestra's built-in [Secrets](https://kestra.io/docs/concepts/secret) for credentials rather than plaintext env vars
- Consider network segmentation between Kestra and PostgreSQL
- Enable HTTPS via a reverse proxy (nginx, Caddy) for production

---

## 14. Additional Resources

- [Kestra Documentation](https://kestra.io/docs)
- [Kestra Helm Deployment on OpenShift](./kestra-openshift-deployment/README.md)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Ansible Documentation](https://docs.ansible.com/)
- [Podman Documentation](https://docs.podman.io/en/latest/)