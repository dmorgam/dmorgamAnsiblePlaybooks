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
kubernetes_apiserver_extra_sans: (comma separated extra dns or endpoints for apiserver)
kubernetes_apiserver_ha_endpoint: (stable load balanced endpoint, for example, one of the extra sans dns)

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
    kubernetes_apiserver_extra_sans: "k8s-cluster.local"
    kubernetes_apiserver_ha_endpoint: "k8s.local"
  roles:
  - role: kubernetes-alpine
```

License
-------

GPLv3

Author Information
------------------

David Moreno (dmorgam), SRE and Kubernetes Admin

