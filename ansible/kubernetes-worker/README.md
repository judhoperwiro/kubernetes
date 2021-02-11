## Ansible Module for installation Kubernetes Worker 1.17.0
​
#### Prerequisite

- ansible 2.9
- vim
- copy kubernetes join token command to inventory file

### 1.) Create Directory for Kubernetes
```shell
mkdir -p /data/ansible/kubernetes/source/
cd /data/ansible/kubernetes
vi install-kubernetes-worker-1-17-0.yaml
vi inventory
```
### 2.) Run Ansible Playbook  
```shell
ansible-playbook -i inventory install-kubernetes-worker-1-17-0.yaml
