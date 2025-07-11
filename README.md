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

- secrets are logged
- configure container restart? does it make sense?
- prom integration
- how do you connect to the cluster? see below

# Own Observations

```
When a follower dies:

1. It remains part of the cluster
2. It needs to restart, then unseal
3. Vault operator raft list-peers : shows that things are good again

When a follower leaves the cluster:
1. vault operator raft remove-peer leavinghost
2. To rejoin, content of /vault/data needs to be removed.
3. Restart node, join with vault operator raft join http://leaderhost:8200
4. Do the unsealing on leaverhost
5. Profit

When the leader dies:
- it’s the same thing as when a follower dies, but a new leader is negotiated 
- you restart, unseal and then continue as a follower
```

# Connecting to the cluster

Connecting to a Vault cluster with a Raft storage backend from outside (e.g., from a client application or external system) requires accounting for high availability, fault tolerance, and the possibility that a particular node might be down. In a Raft setup, Vault nodes form a cluster where one node acts as the leader, and others are followers. Clients typically connect to the leader for read/write operations, but you need a strategy to handle node failures and ensure reliable connectivity. Below is a detailed guide on how to connect from the outside while addressing the possibility of a node being down:

### Key Concepts
- **Leader Election**: In a Raft cluster, one node is the leader, handling all write operations and most read operations (unless configured for read scaling). If the leader goes down, Raft automatically elects a new leader among the remaining nodes, provided the cluster maintains a quorum (majority of nodes).
- **Client Connection**: External clients should not rely on a single node’s address, as any node could be down or not the leader. Instead, use mechanisms to route traffic to the active leader or a healthy node.
- **High Availability**: Use load balancing, DNS, or Vault’s built-in retry mechanisms to ensure clients can connect even if a node is down.

### Steps to Connect from the Outside
1. **Use a Load Balancer**:
   - **Recommended Approach**: Deploy a load balancer (e.g., AWS ELB, HAProxy, NGINX, or a cloud-native service like GCP’s Load Balancer) in front of the Vault cluster to distribute traffic across all healthy nodes.
   - **Configuration**:
     - Configure the load balancer to perform health checks on each Vault node. Vault provides a health endpoint (`/v1/sys/health`) that returns HTTP status codes:
       - `200`: Active node (leader or follower in a healthy state).
       - `429`: Standby node (healthy but not the leader).
       - `472`: Sealed node.
       - `503`: Unhealthy or unreachable node.
     - Set up the load balancer to prefer sending traffic to nodes returning `200` (the leader) or `429` (standby nodes, which can forward requests to the leader).
     - Example HAProxy configuration:
       ```hcl
       frontend vault_frontend
           bind *:8200
           mode tcp
           default_backend vault_backend

       backend vault_backend
           mode tcp
           server vault1 <node1-ip>:8200 check
           server vault2 <node2-ip>:8200 check
           server vault3 <node3-ip>:8200 check
       ```
   - **Benefits**: The load balancer automatically routes traffic away from a downed node and directs requests to the current leader or a healthy follower, which forwards to the leader if needed.

2. **Configure DNS for Failover**:
   - Use a DNS record (e.g., a CNAME or A record) pointing to the load balancer or a set of Vault nodes.
   - If you don’t use a load balancer, configure DNS with multiple A records (one per node) and rely on clients to retry connections if a node is down. However, this is less reliable than a load balancer.
   - Example DNS setup:
     ```
     vault.example.com  IN  A  <node1-ip>
     vault.example.com  IN  A  <node2-ip>
     vault.example.com  IN  A  <node3-ip>
     ```
   - Use a low TTL (e.g., 60 seconds) to allow quick updates if nodes change.

3. **Client-Side Retry Logic**:
   - Configure your Vault client (e.g., Vault CLI, SDK, or application) to handle node failures gracefully:
     - **Vault CLI**: Use the `-address` flag to point to the load balancer’s address (e.g., `vault -address=https://vault.example.com:8200 read secret/data/my-secret`).
     - **Vault SDKs**: Most Vault SDKs (e.g., Go, Python) support automatic retries and leader redirection. For example, in the Go SDK:
       ```go
       import "github.com/hashicorp/vault/api"

       config := &api.Config{
           Address: "https://vault.example.com:8200",
           MaxRetries: 3,
       }
       client, err := api.NewClient(config)
       if err != nil {
           log.Fatal(err)
       }
       ```
     - Set a reasonable retry count (e.g., 3) and timeout (e.g., 5-10 seconds) to handle temporary node outages.
   - If a node is down, the client will either be redirected to the leader (via HTTP 307 redirects from standby nodes) or fail over to another node if using a load balancer or DNS.

4. **Enable Vault’s Auto-Redirection**:
   - Vault nodes automatically redirect requests to the leader if a client hits a standby node (HTTP 307). Ensure your client respects these redirects.
   - If using a load balancer, it should handle this transparently by routing to a healthy node based on health checks.

5. **Handle Sealed or Down Nodes**:
   - If a node is down or sealed, the load balancer or client retry logic should skip it. Ensure your cluster has enough nodes (e.g., 3 or 5) to maintain a quorum even if one node is down.
   - Monitor the cluster with `vault operator raft list-peers` to verify the leader and node health. If a node is consistently down, investigate and recover it (per your previous question about rejoining a node).

6. **Secure Connections**:
   - Always use TLS for external connections to Vault (port 8200 by default). Configure your load balancer or clients to use `https://` and validate the Vault server’s TLS certificate.
   - If using a custom CA, ensure clients trust the CA certificate.

### Example Workflow
- **Setup**: A 3-node Vault Raft cluster (`node1:8200`, `node2:8200`, `node3:8200`) behind an HAProxy load balancer at `vault.example.com:8200`.
- **Client Connection**: A client application uses the Vault Go SDK to connect to `https://vault.example.com:8200`.
- **Node Down**: If `node2` is down, HAProxy’s health checks detect it (via `/v1/sys/health` returning 503 or no response) and routes traffic to `node1` or `node3`.
- **Leader Redirection**: If the client hits a standby node (e.g., `node3`), Vault redirects the request to the leader (e.g., `node1`) via HTTP 307.
- **Fallback**: If the leader changes (e.g., `node1` goes down), Raft elects a new leader, and the load balancer continues routing to healthy nodes.

### Additional Notes
- **Minimum Cluster Size**: A 3-node cluster is recommended for fault tolerance. With one node down, the remaining two nodes can maintain a quorum (2/3). A 5-node cluster tolerates two failures but requires more resources.
- **Monitoring**: Use Vault’s telemetry (e.g., via Prometheus) to monitor node health, leader changes, and request latency. Set up alerts for node outages or sealed states.
- **Firewalls**: Ensure external traffic can reach the Vault nodes (port 8200 for client requests, 8201 for Raft replication between nodes). Restrict access to trusted networks or IPs.
- **Documentation**: These recommendations align with HashiCorp’s Vault Raft deployment guide (`https://developer.hashicorp.com/vault/docs/configuration/storage/raft`).

If you have a specific setup (e.g., cloud provider, load balancer type, or client application), let me know, and I can provide more tailored guidance or troubleshoot connectivity issues!