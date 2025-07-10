# Vault Cluster Setup with Ansible and Docker Compose

Destroy, re-create:
```
ansible-playbook -i inventory.yml main.yaml -e destroy=true
```

Just create:
```
ansible-playbook -i inventory.yml main.yaml
```

Offene Themen:

- I'm seing occasional podman issues as such: ERRO[0000] Refreshing volume d2eba286c16b048cb951a7f21d142d24d21ed9936f97a76710db057376c4befb: acquiring lock 2 for volume d2eba286c16b048cb951a7f21d142d24d21ed9936f97a76710db057376c4befb: file exists .
- secrets are logged