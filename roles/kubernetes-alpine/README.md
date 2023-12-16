Kubernetes Alpine
=========

Install kubernetes on alpine nodes with flannel CNI.

Requirements
------------

Install python jmespath and cryptography.

Role Variables
--------------

kubernetes_node_type:    (master/worker)
kubernetes_first_master: (machine name)
kubernetes_pods_cidr:    (kubeadm pod cidr)
kubernetes_service_cidr: (kubernetes service cidr)
kubernetes_cni_yaml:     (yaml url of flannel)

Dependencies
------------

Install ansible galaxy requirements, community.general, community.crypto

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yaml
---
- name: Kubernetes master
  hosts: masters:workers
  become_method: su
  vars:
    kubernetes_first_master: "{{ groups['masters'][0] }}"
  roles:
  - role: kubernetes-alpine
```

License
-------

GPLv3

Author Information
------------------

David Moreno (dmorgam), SRE and Kubernetes Admin

