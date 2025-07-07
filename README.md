# Vault Cluster Setup with Ansible and Docker Compose


Destroy, re-create:
```
ansible-playbook -i inventory.yml main.yaml -e destroy=true
```

Just create:
```
ansible-playbook -i inventory.yml main.yaml
```

1. create and start all nodes
2. init cluster if necessary
3. unseal leader
4. non-leaders join cluster
5. unseal non-leaders

