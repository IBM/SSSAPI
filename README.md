# SSSAPI
IBM Spectrum Scale System — API Server Container Manager

Hosts the **SSS API Server container** on an IBM Utility BareMetal node (5149-23E).
The SSS API Server is an HTTPS backend that exposes the `essctl` CLI interface for deploying and managing IBM Storage Scale System (ESS) nodes. It is required for both the RCL service container and the Deploy from GUI container to function.

> **Note:** `SSSRCL`, `SSSAPI`, and `SSSDEPLOYGUI` have been unified into the [SSSMANAGER](https://github.ibm.com/IBMSpectrumScale/SSSMANAGER) repository. Use `SSSMANAGER` for all new deployments. This repository is maintained for standalone use only.

---

## Quick start

```bash
./startAPIContainer
```

This is the **only script you need to run**. It auto-detects system settings, writes a sanitised `apimgr.yml`, and starts the container.

---

## Usage

```text
./startAPIContainer [options]

Options:
  -d, --delete                         Delete the SSS API container and its systemd service unit.
  -v, --verbose                        Display verbose messages in the console.

  Install source (mutually exclusive):
  -f, --file-name FILE                 Restore the container from a local tar file.
                                       (default: SSSAPI.tar)
  -vn, --image-version VER             API server image version to deploy.
                                       (default: 7.0.1.2)

  -p, --port PORT                      Port on which the API server listens.
                                       (default: 46443)
  -i, --campus-interface IF            Name of the interface connected to the campus
                                       network. (default: campus)
  -up, --utilitybaremetal-password PW  Root password for the utilityBareMetal host.
                                       (default: cluster)
  -e, --is-emsvm23e-exist yes|no       Specify whether EMSVM-23E exists.
                                       (default: no)
  -ep, --emsvm23e-password PW          Root password for EMSVM-23E.
```

### Examples

```bash
# Start using default settings (pulls image from IBM registry)
./startAPIContainer

# Start from a local tarball
./startAPIContainer -f SSSAPI.tar

# Start with custom port and utility host password
./startAPIContainer -vn 7.0.1.2 -p 46443 -up mypassword

# Override the image version
./startAPIContainer -vn 7.0.1.1

# Start with EMSVM-23E present
./startAPIContainer -vn 7.0.1.2 -e yes -ep emspassword

# Verbose output
./startAPIContainer -v

# Delete the container
./startAPIContainer -d
```

---

## Prerequisites

### OS and software

- RHEL / Red Hat compatible Linux
- `podman` installed and accessible
- `nmcli` installed and accessible
- User-level systemd lingering enabled (`loginctl enable-linger`)

### Network interfaces

| Interface | Purpose |
|-----------|---------|
| `campus`  | Public network — connects the Utility host to the Internet |
| `virbr1`  | RAS bridge — internal communication with EMSVM |

### OS user

Create the `apiadmin` user before running the script:

```bash
sudo useradd apiadmin
sudo passwd apiadmin
```

No additional sudoers entry is required for `apiadmin`.

### Hosts file

Replace `/etc/hosts` with the provided template (or append its contents):

```bash
sudo cp utilityBareMetal_hosts /etc/hosts
```

Ensure the RAS IP for EMSVM (`10.23.16.2 emsvm`) is correct for your system.

### IBM Container Registry login

```bash
podman login cp.icr.io --username cp --password <API_Key>
```

The API Server image is pulled from:

```
cp.icr.io/cp/scalesystem/sss_api
```

### SSL certificate

When first started, the container generates a self-signed certificate stored at `/home/apiadmin/keys/`. This certificate **must** be replaced with either a CA-signed certificate or a customer-generated self-signed certificate before use in production.

### Pre-configured container users

| User       | Role |
|------------|------|
| `apiadmin` | Superuser — can manage all authorisations |
| `apiuser`  | Non-privileged user |

---

## Configuration

`apimgr.yml` is written automatically by `startAPIContainer`. You can also edit it manually for advanced settings:

```yaml
CONTAINER:
    CONTAINER_HOSTNAME: utilityBareMetal-api-official
    CONTAINER_DOMAIN_NAME: gpfs.local

    UTILITY_HOSTNAME: utilityBareMetal

    CAMPUS_INTERFACE: campus
    CAMPUS_INTERFACE_IP: 9.114.12.77   # ← auto-detected at runtime

    RAS_INTERFACE: virbr1
    RAS_INTERFACE_IP: 10.23.16.1

    API_PORT: 46443                    # HTTPS API port
    SSH_PORT: 20022                    # SSH port (development only)

    IMAGE_NAME: cp.icr.io/cp/scalesystem/sss_api
    IMAGE_VERSION: 7.0.1.2

    LOG:    /home/apiadmin/log
    BKUP:   /home/apiadmin/backup
    BUILDS: /serv
```

> `CAMPUS_INTERFACE_IP` is auto-detected from the campus interface at runtime. The value in the YAML is used as a fallback only.

### Design constraints

- The SSS API Server is designed for a **single-host** environment. It cannot communicate with or exchange state with another SSS API Server instance.
- The API Server is a cut-down version of the SS MoS API Server without IBM Storage Scale cluster management functionality.

---

## Port reference

| Host port | Container port | Purpose |
|-----------|----------------|---------|
| 46443     | 46443          | HTTPS API |
| 20022     | 22             | SSH (development only) |

---

## Systemd service

After a successful start the container is registered as a user-level systemd service that auto-starts on login:

```bash
# Check status
systemctl --user status container-utilityBareMetal-api-official.service

# Start / stop
systemctl --user start  container-utilityBareMetal-api-official.service
systemctl --user stop   container-utilityBareMetal-api-official.service
```

---

## Logs

Local execution logs are written to `./logs/` and are also copied to `/var/log/sssapi/startAPIContainer/` if that directory exists.

---

## Project structure

```text
SSSAPI/
├── startAPIContainer        ← entry point (run this)
├── apimgr.yml               ← configuration file (auto-generated / editable)
├── apimgr                   ← legacy low-level manager (deprecated, use startAPIContainer)
├── utilityBareMetal_hosts   ← /etc/hosts template for the Utility node
└── classes/
    └── apimgr_yml.py        ← config + container lifecycle class
```

---

## Related repositories

| Repository | Purpose |
|------------|---------|
| [SSSMANAGER](https://github.ibm.com/IBMSpectrumScale/SSSMANAGER) | Unified manager for RCL + API + Deploy GUI |
| [SSSRCL](https://github.ibm.com/IBMSpectrumScale/SSSRCL) | RCL container manager |
| [SSSDEPLOYGUI](https://github.ibm.com/IBMSpectrumScale/SSSDEPLOYGUI) | Deploy from GUI container manager |
