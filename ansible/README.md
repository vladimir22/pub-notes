## Ansible playbooks
How to Run:

```sh
ANSIBLE_DIR=/mnt/d/Project/github/vladimir22/pub-notes/ansible
cd $ANSIBLE_DIR

export ANSIBLE_STDOUT_CALLBACK=yaml
ansible-playbook -i inventory ./playbooks/restore.yaml
```