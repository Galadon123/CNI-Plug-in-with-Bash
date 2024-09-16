## To enable communication between containers on different vms
### On Worker-1:
1. **Add a Route to the Container Network of Worker-2:**
   ```bash
   sudo ip route add 10.244.2.0/24 via 10.0.0.187  # 10.0.0.187 worker-2 PrivateIP
   ```

2. **Enable IP Forwarding:**
   Ensure that IP forwarding is enabled on Worker-1. This can be done by setting the following sysctl parameter:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

### On Worker-2:
1. **Add a Route to the Container Network of Worker-1:**
   ```bash
   sudo ip route add 10.244.1.0/24 via 10.0.0.144  # 10.0.0.144 worker-1 Private Ip
   ```

2. **Enable IP Forwarding:**
   Ensure that IP forwarding is enabled on Worker-2 with the same command:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

### Verify the Setup
1. **From a container on Worker-1:**
   ```bash
   kubectl exec -it bash-worker-1 -- ping 10.244.2.2
   ```

   Not working