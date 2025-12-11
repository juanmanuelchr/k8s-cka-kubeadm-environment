# Local Environment based on kubeadm

The idea of this environment is to have a local sandbox for preparing for Kubernetes certifications based on kubeadm (https://kubernetes.io/docs/reference/setup-tools/kubeadm/)

## Setup Instructions

### Prerequisites

- Basic knowledge of Linux and container concepts
- Terminal/Command Line familiarity
- Git installed
- [Multipass](https://multipass.run/) (for creating local and light Virtual Machines)

### Environment Setup

1. **Install Multipass:**
   - `brew install multipass` command-line tool
   - Minikube

2. **Setup Steps:**
   ```bash
   # Start Minikube with Docker driver
   cd setup
   ./reset_lab_environment.sh  # This will initialize Minikube with Docker driver
   ```

### Setting Up Lab Environments

#### Option 1: Using the TUI Launcher (Recommended)

The repository includes a Text-based User Interface (TUI) launcher for easy navigation and management of the lab environments:

```bash
cd setup
./lab_launcher.sh        # Launches the interactive TUI
```

This launcher provides a menu-driven interface to:
- Start/reset the Minikube environment
- Set up individual lab environments for each domain
- Check the status of resources in each namespace
- View node status

#### Option 2: Using Individual Setup Scripts

The repository contains setup scripts for each domain:

```bash
cd setup
./01_setup_storage_lab.sh        # Sets up Storage lab environment
./02_setup_workloads_lab.sh      # Sets up Workloads lab environment
./03_setup_networking_lab.sh     # Sets up Networking lab environment
./04_setup_troubleshooting_lab.sh # Sets up Troubleshooting lab environment
./05_setup_cluster_arch_lab.sh   # Sets up Cluster Architecture lab environment
```

To reset your environment at any time:

```bash
cd setup
./reset_lab_environment.sh
```

## How to Use This Repository

### Directory Structure

```
CKA_LAB/
├── 01_Storage/                # Storage domain exercises
│   ├── README.md              # Task instructions
│   ├── solutions/             # Official solution files
│   └── user_solutions/        # Your solutions (gitignored)
├── 02_Workloads/              # Workloads domain exercises
│   ├── ...
├── 03_Networking/             # Networking domain exercises
│   ├── ...
├── 04_Troubleshooting/        # Troubleshooting domain exercises
│   ├── ...
├── 05_Cluster_Architecture/   # Cluster Architecture domain exercises
│   ├── ...
└── setup/                     # Setup scripts
    ├── 01_setup_storage_lab.sh
    ├── ...
    ├── reset_lab_environment.sh
    └── verify_solutions.sh    # Script to verify your solutions
```