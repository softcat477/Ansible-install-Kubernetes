# Install Kubernetes with Ansible
> The documentation for installing kubectl, kubeadm, and kubelet is like a combination of riddle and maze (always forget where the link is). So I wrote this.
> 

Install these with 🌟`Ansible`:
- docker
- containerd
- kubeadm v1.28
- kubectl
- kubelet

```
$ ansible-playbook playbook-k8s-configure.yml -i <wherever you're ansible inventory is> --diff
```

## What's going on in each file?
### Install container runtime
####  1. Prerequisites for installing container runtime
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic
```bash=
# main.yml
- name: Load overlay and br_netfilter
  include_tasks: loadKernelModule.yml
  loop:
    - overlay
    - br_netfilter
- name: sysctl params required by setup, params persist across reboots
  become: true
  lineinfile:
    path: /etc/sysctl.d/k8s.conf
    state: present
    create: true
    line: "{{ item }}"
  loop:
    - "net.bridge.bridge-nf-call-iptables  = 1"
    - "net.bridge.bridge-nf-call-ip6tables = 1"
    - "net.ipv4.ip_forward                 = 1"
- name: Apply sysctl params without reboot
  become: true
  shell: sysctl --system
```

#### 2. Verify `br_netfilter`, `overlay` modules
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic
```yaml=
# main.yml
- name: Verify that br_netfilter is loaded
  shell: lsmod | grep br_netfilter
  changed_when: false
  ignore_errors: true
  register: out_betfilter

- name: Fail if br_netfilter is not loaded
  fail:
    msg: "br_netfilter is not loaded"
  when: out_betfilter.rc != 0
  
- name: Verify that overlay is loaded
  shell: lsmod | grep overlay
  changed_when: false
  ignore_errors: true
  register: out_overlay

- name: Fail if overlay is not loaded
  fail:
    msg: "overlay is not loaded"
  when: out_overlay.rc != 0
```

#### 3. Verify `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables`, and `net.ipv4.ip_forward` system variables
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic
```yaml=
# main.yml
- name: Verify sysctl
  become: true
  shell: sudo sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
  changed_when: false
  register: out_sysctl

- name: Fail if system variables are not set to 1
  fail:
    msg: "system variable {{ item }} does not end with 1"
  when: item.split("=")[-1].strip() != "1"
  with_items: "{{ out_sysctl.stdout_lines }}"
```

#### 4. Install prerequisites for containerd
Speed-up by checking if docker has been installed. Do no install docker if it's already in the machine.
```yaml=
# installContainerd_Ubuntu.yml
- name: Check if Docker is installed
  command: "docker --version"
  ignore_errors: true
  register: docker_check
  
- name: Install Docker and containerd
  block:
  - name: ...
  - name: ...
  ...
  become: true
  when: docker_check.rc != 0
```

Install `containerd` based on the OS you're using: https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf. This repo only works for `Ubuntu` and `Debian`.

For [Ubuntu](https://docs.docker.com/engine/install/ubuntu/):
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

```yaml=
# installContainerd_Ubuntu.yml
- name: Install Docker and containerd
  block:
  - name: apt-get update
    ...
  - name: Create directory /etc/apt/keyrings
    ...
  - name: Check if the Docker GPG key file exists
    ...
  - name: Fetch the Docker GPG key and save the dearmored key
    ...
  - name: Change the permmision of dearmored key
    ...
  - name: Get version codename
    ...
  - name: Get arch
    ...
  - name: Print codename and arch
    ...
  - name: Add repository to Apt sources
    ...
  - name: Test add repository
  become: true
  register: config_changed
    ...
```

#### 5. Install containerd
Update the apt package and install `Docker` and `containerd`
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
```yaml=
# installContainerd_Ubuntu.yml
- name: Install Docker and containerd
  block:
  ...
  - name: Run apt-get update and install docker packages
    apt:
      update_cache: true
      name:
        - docker-ce
        - docker-ce-cli 
        - containerd.io 
        - docker-buildx-plugin 
        - docker-compose-plugin
  become: true
  register: config_changed
```

#### 6. Configuring the systemd cgroup driver and restart containerd
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd
```yaml=
# installContainerd_Ubuntu.yml
- name: Set /etc/containerd/config.toml
  ansible.builtin.blockinfile:
    path: /etc/containerd/config.toml 
    block: |
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
          SystemdCgroup = true
  become: true
  register: config_changed

- name: Restart containerd
  ansible.builtin.systemd:
    name: containerd
    state: restarted
  become: true
  when: config_changed is changed
```

#### 7.5 (Not in the playbook!) Check port 6443 is free
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports
This step is not in the playbook yet :(

#### 8. Prerequisites for installing kubeadm, kubectl, and kubelet
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
```yaml=
# installKubeadm_Debian.yml
- name: Install packages needed to use the Kubernetes apt repository
  apt:
    update_cache: true
    name:
      - apt-transport-https 
      - ca-certificates 
      - curl 
      - gpg
  become: true

- name: Check if the kubernetes signing key file exists
  stat:
    path: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  register: k8s_gpg_key_check

- name: Download the public signing key for the Kubernetes package repositories
  shell: curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  when: not k8s_gpg_key_check.stat.exists

- name: Add repository for kubernetes 1.28 to Apt sources
  ansible.builtin.lineinfile:
    path: /etc/apt/sources.list.d/kubernetes.list
    line: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /"
    state: present
    create: yes
  become: true
```
#### 9. Update the apt package, install kubelet, kubeadm, and kubectl, and pin their version
https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
```yaml=
# installKubeadm_Debian.yml
- name: Install packages needed to use the Kubernetes apt repository
  apt:
    update_cache: true
    name:
      - kubelet 
      - kubeadm 
      - kubectl
  become: true

- name: Prevent kubernetes being upgraded
  shell: apt-mark hold kubelet kubeadm kubectl
  become: true
```