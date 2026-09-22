# ansible-role-nvidia-gpu

Installs the NVIDIA GPU driver and NVIDIA Container Toolkit on Ubuntu GPU worker nodes.
Supports two tagged phases for Kubernetes bootstrap workflows:

| Tag | When to run | Actions |
|---|---|---|
| `nvidia_driver` | Before Kubespray join | CUDA repo, driver 580, persistenced, optional reboot, `nvidia-smi` check |
| `nvidia_containerd` | After Kubespray join | Container toolkit, `nvidia-ctk`, containerd restart |

## Requirements

- Ubuntu 24.04 (Noble) on x86_64
- NVIDIA GPU hardware present
- Root/sudo access
- Ansible >= 2.16 (uses `deb822_repository`)
- For `nvidia_containerd`: containerd already installed (e.g. by Kubespray) and NVIDIA driver loaded

## Role variables

| Variable | Description | Default |
|---|---|---|
| `nvidia_driver_package` | Driver metapackage | `nvidia-driver-580` |
| `nvidia_driver_pinning_package` | Branch pinning package | `nvidia-driver-pinning-580` |
| `nvidia_driver_hold_packages` | Packages to `apt-mark hold` | `[nvidia-driver-580, nvidia-driver-pinning-580]` |
| `nvidia_expected_gpu_count` | GPUs expected from `nvidia-smi` (`0` disables the check) | `0` |
| `nvidia_driver_reboot_required` | Reboot when driver packages change or module is not loaded | `true` |
| `nvidia_driver_reboot_timeout` | Reboot wait timeout (seconds) | `600` |
| `nvidia_cuda_keyring_checksum` | Optional `sha256:…` checksum for the CUDA keyring `.deb` | `""` |
| `nvidia_container_runtime` | Runtime for `nvidia-ctk` | `containerd` |
| `nvidia_containerd_configure_runtime` | Run `nvidia-ctk runtime configure`; set `false` when Kubespray (`containerd_additional_runtimes`) owns the containerd config | `true` |
| `nvidia_containerd_config_path` | containerd config file checked for changes | `/etc/containerd/config.toml` |
| `nvidia_driver_proc_path` | Path checked to confirm driver module is loaded | `/proc/driver/nvidia/version` |

See [`defaults/main.yml`](defaults/main.yml) for repository URLs and remaining defaults.

## Example playbook

```yaml
- name: Install NVIDIA GPU driver
  hosts: geom-gpu-nodes
  become: true
  vars:
    nvidia_expected_gpu_count: 2
  roles:
    - ansible-role-nvidia-gpu
  tags: [nvidia, nvidia_driver, gpu]

- name: Configure NVIDIA container runtime for containerd
  hosts: geom-gpu-nodes
  become: true
  roles:
    - ansible-role-nvidia-gpu
  tags: [nvidia, nvidia_containerd, gpu]
```

Run phases separately:

```bash
ansible-playbook playbooks/cluster_hardening.yaml --tags nvidia_driver --limit geom-node125
ansible-playbook playbooks/geom_gpu_nodes.yml --tags nvidia_containerd --limit geom-node125
```

## Tests

Syntax check:

```bash
python3 -m venv .venv
.venv/bin/pip install ansible ansible-lint yamllint
ANSIBLE_ROLES_PATH=.. .venv/bin/ansible-playbook -i tests/inventory tests/test.yml --syntax-check
```

Molecule (containerd phase idempotence with mocked GPU tooling):

```bash
pip install "ansible>=9" molecule molecule-plugins[docker]
molecule test -s default
```

## License

Apache-2.0
