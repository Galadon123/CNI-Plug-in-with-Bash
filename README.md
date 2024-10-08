# **Custom Kubernetes CNI Plug-in Setup with Bash**

## **Introduction**

Kubernetes is a powerful platform that automates the deployment, scaling, and management of containerized applications. One of the key components of Kubernetes is its networking model, which ensures seamless communication between containers (pods), services, and nodes. However, understanding the intricacies of how Kubernetes networking works can be challenging. The ability for pods to communicate across nodes, implement load balancing, and enforce network policies is vital for any Kubernetes cluster, and this is where **Container Network Interface (CNI) plug-ins** come into play.

CNI plug-ins are responsible for configuring the networking for containers, assigning IP addresses, and managing network interfaces. There is no single standard implementation for Kubernetes networking, as the networking setup is often influenced by the environment where the cluster is deployed. This flexibility allows for a variety of CNI plug-ins that can meet specific networking requirements.

![](./images/11.drawio.svg)

In this guide, we'll explore the **internals of Kubernetes networking** and create a **custom CNI plug-in** using Bash to manage networking in a Kubernetes cluster. This plug-in will configure pod network interfaces, set up an overlay network for inter-node communication, and handle the allocation of IP addresses. By understanding and implementing a custom CNI plug-in, you'll gain deeper insights into Kubernetes networking and how to manage low-level networking tasks.

## **Setting Up the Infrastructure with Terraform**

To run Kubernetes on AWS, we will first provision the necessary infrastructure using **Terraform**. This setup will create a Virtual Private Cloud (VPC), subnets, security groups, and three EC2 instances that will serve as the **master** and **worker nodes** of our Kubernetes cluster.

![](./images/12.drawio.svg)

### AWS CLI Configuration

Run the following command to configure AWS CLI:

```bash
aws configure
```
![alt text](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/1.png)

This command prompts you for your AWS Access Key ID, Secret Access Key, region, and output format.

### **Terraform Configuration**

Here’s the Terraform script to provision the required AWS infrastructure:

```hcl
# Provider configuration
provider "aws" {
  region = "ap-southeast-1" # Replace with your desired region
}

# Create a key pair and store it locally
resource "tls_private_key" "example" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "my_key_pair" {
  key_name   = "cni"
  public_key = tls_private_key.example.public_key_openssh
}

resource "local_file" "private_key" {
  filename = "${path.module}/cni.pem"
  content  = tls_private_key.example.private_key_pem
  file_permission = "0400"
}

# Create a VPC
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "my-vpc"
  }
}

# Create an Internet Gateway
resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "my-igw"
  }
}

# Create a public subnet
resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone = "ap-southeast-1a" # Change based on your region
  tags = {
    Name = "public-subnet"
  }
}

# Create a route table
resource "aws_route_table" "my_rt" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_igw.id
  }

  tags = {
    Name = "my-rt"
  }
}

# Associate the route table with the public subnet
resource "aws_route_table_association" "my_rt_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.my_rt.id
}

# Create a security group allowing all traffic
resource "aws_security_group" "allow_all_traffic" {
  vpc_id = aws_vpc.my_vpc.id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow-all-traffic"
  }
}

# Define AMI and instance type
variable "ami_id" {
  default = "ami-01811d4912b4ccb26"  # Replace with your desired AMI
}

variable "instance_type" {
  default = "t3.small"
}

# User data script to set hostname
variable "user_data" {
  type = list(string)
  default = [
    <<-EOF
    #!/bin/bash
    hostnamectl set-hostname master
    EOF
    ,
    <<-EOF
    #!/bin/bash
    hostnamectl set-hostname worker-1
    EOF
    ,
    <<-EOF
    #!/bin/bash
    hostnamectl set-hostname worker-2
    EOF
  ]
}

# Create 3 EC2 instances with specific hostnames
resource "aws_instance" "ec2_instances" {
  count = 3

  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public_subnet.id
  vpc_security_group_ids = [aws_security_group.allow_all_traffic.id]
  associate_public_ip_address = true
  key_name      = aws_key_pair.my_key_pair.key_name

  # Use different user_data script to set hostname per instance
  user_data = element(var.user_data, count.index)

  tags = {
    Name = "ec2-instance-${count.index + 1}"
  }
}

# Output the key pair location and instance public IPs
output "private_key_path" {
  value = local_file.private_key.filename
}

output "ec2_public_ips" {
  value = [for instance in aws_instance.ec2_instances : instance.public_ip]
}
```

### **Apply Terraform Script**

1. Install Terraform on your local machine.
2. Initialize the Terraform configuration:
   ```bash
   terraform init
   ```
3. Apply the Terraform configuration:
   ```bash
   terraform apply
   ```
   ![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/2.png)
 
   This will create a VPC, subnet, Internet gateway, route tables, and three EC2 instances. After Terraform completes, it will output the public IPs and the path to the SSH private key for accessing the instances.



## **Setting Up Kubernetes Cluster**

### **SSH into EC2 Instances**

Use the private key from the Terraform output to SSH into each EC2 instance:

```bash
ssh -i cni.pem ubuntu@<instance-public-ip>
```

![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/3.png)

### **Configure Each Node**

On each VM (master, worker-1, worker-2), run the following commands to set up Kubernetes:

```bash
# Update and install required packages
sudo apt-get update
sudo apt-get install -y docker.io apt-transport-https curl jq nmap iproute2

# Add Kubernetes APT repository and install Kubernetes components
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo sysctl -w net.ipv4.ip_forward=1
```

### **Initialize Kubernetes Cluster**

On the **master node**, initialize the Kubernetes cluster:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

This command initializes the Kubernetes cluster on the master node. The --pod-network-cidr=10.244.0.0/16 flag specifies the range of IP addresses that will be used for the pod network.

Next, use the `kubeadm join` command in both worker nodes to add the **worker nodes** to the cluster.

![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/4.png)

### **Testing the cluster**

Now, it’s time to check whether our cluster is working properly. The first thing we need to do is to configure kubectl to connect to the newly created cluster. In order to do this, run the following commands from the master VM:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now, you should be able to use kubectl from the master VM. Let’s use the kubectl get nodes command to check the status of the cluster nodes.

```bash
kubectl get nodes
```
![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/5.png)

As you can see from the output, both master and worker nodes are currently in the “NotReady” state. This is expected, because we haven’t configured any networking plug-in yet. If you try to deploy a pod at this time, your pod will forever hang in the “Pending” state, because the Kubernetes schedule will not be able to find any “Ready” node for it.

### **Create the Custom CNI Plug-in**

To implement networking for Kubernetes pods, we will create a custom CNI plug-in in Bash.

#### Create the CNI configuration on each node (master,worker-1,worker-2):

   ```bash
   sudo nano /etc/cni/net.d/10-bash-cni-plugin.conf
   ```

   Add the following content:
   ```json
   {
       "cniVersion": "0.3.1",
       "name": "mynet",
       "type": "bash-cni",
       "network": "10.244.0.0/16",
       "subnet": "<node-cidr-range>"
   }
   ```

   Replace `<node-cidr-range>` with:
   - `10.244.0.0/24` for the **master** node.
   - `10.244.1.0/24` for **worker-1**.
   - `10.244.2.0/24` for **worker-2**.


   This file contains the configuration for the custom **CNI plug-in**. The configuration defines the CNI version, the name of the network (`mynet`), the type of CNI plug-in (`bash-cni`), and the network and subnet details. It specifies how the custom CNI plug-in should behave, including the IP ranges for the pods on each node.

   The key sections include:
   - `cniVersion`: Specifies the version of the CNI specification.
   - `name`: Name of the CNI network.
   - `type`: The type of CNI plug-in, in this case, a custom `bash-cni` plug-in.
   - `network` and `subnet`: Define the network and subnet ranges that the plug-in will use to allocate IP addresses to pods.

   Now the nodes will be in ready state

   ```bash
   kubectl get nodes
   ```
   ![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/6.png)

### Create the network

 bridge for each node:

   ```bash
   sudo brctl addbr cni0
   sudo ip link set cni0 up
   sudo ip addr add <bridge-ip>/24 dev cni0
   ```

   Replace `<bridge-ip>` with:
   - `10.244.0.1` for the **master** node.
   - `10.244.1.1` for **worker-1**.
   - `10.244.2.1` for **worker-2**.

   These commands set up a new network bridge named `cni0` on a Linux system:

   1. **`sudo brctl addbr cni0`**: Creates a new bridge interface named `cni0` using the `brctl` utility.
   2. **`sudo ip link set cni0 up`**: Activates the `cni0` bridge interface, making it operational.
   3. **`sudo ip addr add <bridge-ip>/24 dev cni0`**: Assigns an IP address (specified by `<bridge-ip>`) with a subnet mask of `/24` to the `cni0` bridge, enabling it to communicate on the network.
   

### **Create the Custom CNI Plug-in Script**

On each node, save the CNI script to `/opt/cni/bin/bash-cni`:

```bash
sudo nano /opt/cni/bin/bash-cni
```

Insert the following Bash script:

```bash
#!/bin/bash -e

if [[ ${DEBUG} -gt 0 ]]; then set -x; fi

exec 3>&1
exec &>> /var/log/bash-cni-plugin.log

IP_STORE=/tmp/reserved_ips

echo "CNI command: $CNI_COMMAND"
stdin=$(cat /dev/stdin)
echo "stdin: $stdin"

allocate_ip(){
    for ip in "${all_ips[@]}"
    do
        reserved=false
        for reserved_ip in "${reserved_ips[@]}"
        do
            if [ "$ip" = "$reserved_ip" ]; then
                reserved=true
                break
            fi
        done
        if [ "$reserved" = false ] ; then
            echo "$ip" >> $IP_STORE
            echo "$ip"
            return
        fi
    done
}

case $CNI_COMMAND in
ADD)
    network=$(echo "$stdin" | jq -r ".network")
    subnet=$(echo "$stdin" | jq -r ".subnet")
    subnet_mask_size=$(echo $subnet | awk -F  "/" '{print $2}')

    base_ip=$(echo "$subnet" | awk -F '/' '{print $1}' | awk -F '.' '{print $1"."$2"."$3}')
    all_ips=($(seq -f "$base_ip.%g" 2 254))

    gw_ip="$base_ip.1"
    reserved_ips=$(cat $IP_STORE 2> /dev/null || printf "$base_ip.0\n$gw_ip\n")
    reserved_ips=(${reserved_ips[@]})
    printf '%s\n' "${reserved_ips[@]}" > $IP_STORE
    container_ip=$(allocate_ip)

    mkdir -p /var/run/netns/
    ln -sfT $CNI_NETNS /var/run/netns/$CNI_CONTAINERID

    rand=$(tr -dc 'A-F0-9' < /dev/urandom | head -c4)
    host_if_name="veth$rand"
    ip link add $CNI_IFNAME type veth peer name $host_if_name

    ip link set $host_if_name up
    ip link set $host_if_name master cni0

    ip link set $CNI_IFNAME netns $CNI_CONTAINERID
    ip netns exec $CNI_CONTAINERID ip link set $CNI_IFNAME up
    ip netns exec $CNI_CONTAINERID ip addr add $container_ip/$subnet_mask_size dev $CNI_IFNAME
    ip netns exec $CNI_CONTAINERID ip route add default via $gw_ip dev $CNI_IFNAME

    mac=$(ip netns exec $CNI_CONTAINERID ip link show $CNI_IFNAME | awk '/ether/ {print $2}')
    echo "{
      \"cniVersion\": \"0.3.1\",
      \"interfaces\": [                                            
          {
              \"name\": \"$CNI_IFNAME\",
              \"mac\": \"$mac\",                            
              \"sandbox\": \"$CNI_NETNS\" 
          }
      ],
      \"ips\": [
          {
              \"version\": \"4\",
              \"address\": \"$container_ip/$subnet_mask_size\",
              \"gateway\": \"$gw_ip\",          
              \"interface\": 0 
          }
      ]
    }" >&3

;;

DEL)
    ip=$(ip netns exec $CNI_CONTAINERID ip addr show $CNI_IFNAME | awk '/inet / {print $2}' | sed s%/.*%% || echo "")
    if [ ! -z "$ip" ]
    then
        sed -i "/$ip/d" $IP_STORE
    fi
;;

GET)
    echo "GET not supported"
    exit 1
;;

VERSION)
    echo '{
      "cniVersion": "0.3.1", 
      "supportedVersions": [ "0.3.0", "0.3.1", "0.4.0" ] 
    }' >&3
;;

*)
  echo "Unknown CNI command: $CNI_COMMAND"
  exit 1
;;

esac
```

### **Bash CNI Plug-in Script Overview**

The `bash-cni` script handles various network tasks for Kubernetes pods, such as assigning IP addresses, configuring network interfaces, and deleting the configuration when the pod is removed. Below are the details of each section of the script:

#### IP Allocation (ADD Command)

This section is responsible for assigning an IP address to the newly created pod. The script reads the subnet configuration and allocates an IP address that is not already in use. It ensures that `.0` and `.1` addresses are reserved (for network and gateway purposes), and allocates an available IP from the remaining pool.

- `allocate_ip`: This function generates a list of available IPs and checks whether an IP is already reserved. If it finds an unused IP, it assigns it to the pod.
- `network` and `subnet`: The network and subnet are extracted from the `stdin` input to determine the appropriate range for IP allocation.
- The allocated IP is added to the `/tmp/reserved_ips` file to ensure no conflicts occur with future IP assignments.

#### Network Interface Creation

Once the IP is allocated, the script configures the network interfaces for the pod. It creates a virtual Ethernet pair (`veth`) to connect the host to the container’s network namespace. One end of the `veth` pair is added to the bridge (`cni0`) on the host, and the other end is moved to the pod's network namespace.

- `ip link add`: This command creates the `veth` pair.
- `ip link set`: These commands configure the link for the network interface, attaching one end to the host bridge and setting up the other inside the container.

#### Gateway and Routing Setup

After configuring the network interfaces, the script sets up routing for the pod. It assigns the default gateway for the pod, allowing it to route traffic outside of its local subnet (e.g., to other pods or external networks).

- `ip netns exec`: This runs the `ip` commands inside the pod’s network namespace, ensuring the configuration applies directly to the pod.
- The script also assigns a default route using the gateway IP (`gw_ip`), enabling the pod to communicate beyond its local network.

#### MAC Address and Result Formatting

The script retrieves the **MAC address** of the pod's network interface and formats the result in JSON according to the CNI specification. This JSON output includes details about the interface name, MAC address, assigned IP, and gateway. The result is returned to Kubernetes, which integrates the information into the overall cluster network configuration.

- `mac`: Extracts the MAC address of the pod’s interface.
- The final output is a JSON object that Kubernetes reads to understand the pod’s network setup.

#### IP Deletion (DEL Command)

The `DEL` command is triggered when a pod is deleted, and it cleans up the network configuration. Specifically, it removes the IP address from the `/tmp/reserved_ips` file, making it available for future pods. This prevents IP conflicts and ensures efficient reuse of addresses.

- `sed -i "/$ip/d"`: This command deletes the IP address from the reserved IP list when the pod is removed.

#### Versioning (VERSION Command)

This section outputs the supported versions of the CNI specification. It is a basic function that ensures compatibility between the CNI plug-in and the Kubernetes cluster.

- `cniVersion`: Specifies the CNI version supported by the plug-in.
- `supportedVersions`: Lists the CNI versions the plug-in can work with, allowing for compatibility with different Kubernetes setups.

### Make the script executable:

```bash
sudo chmod +x /opt/cni/bin/bash-cni
```

### Deploy Pods for Testing

Create a Kubernetes deployment to test the CNI plug-in. Use the following YAML configuration (`deploy.yaml`) in master nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-worker-1
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  nodeSelector:
    kubernetes.io/hostname: worker-1
---
apiVersion: v1
kind: Pod
metadata:
  name: bash-worker-1
spec:
  containers:
  - name: ubuntu
    image: smatyukevich/ubuntu-net-utils
    command:
      - "/bin/bash"
      - "-c"
      - "sleep 10000"
  nodeSelector:
    kubernetes.io/hostname: worker-1
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-worker-2
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  nodeSelector:
    kubernetes.io/hostname: worker-2
---
apiVersion: v1
kind: Pod
metadata:
  name: bash-worker-2
spec:
  containers:
  - name: ubuntu
    image: smatyukevich/ubuntu-net-utils
    command:
      - "/bin/bash"
      - "-c"
      - "sleep 10000"
  nodeSelector:
    kubernetes.io/hostname: worker-2
```

#### Apply the deployment:

```bash
kubectl apply -f deploy.yaml
```

Here, we are deploying four simple pods. Two goes on the worker-1 and the remaining two on the worker-2. 

Now, let’s run kubectl get pod to make sure that all pods are healthy and then get the pods IP addresses using the following command:

```bash
kubectl get pods -o wide
```

![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/7.png)

In your case, the result might be different.

Next, you have to get inside inside the bash-worker-1 pod.

```bash
kubectl exec -it bash-worker-1 -- bash
```

From inside of the pod, you can ping various addresses to verify network connectivity. 

![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/8.png)

As you can see, the only thing that actually works is a container to host communication. 

### **Fixing Container-to-Container Communication**

Container-to-container communication issues were caused by the default `iptables` rule, which blocks traffic by setting the `FORWARD` chain policy to `DROP`. To resolve this, add the following rules on both master and worker nodes to allow traffic within the pod CIDR range:

```bash
sudo iptables -t filter -A FORWARD -s 10.244.0.0/16 -j ACCEPT
sudo iptables -t filter -A FORWARD -d 10.244.0.0/16 -j ACCEPT
```

Here we add specific iptables rules that allow forwarding of traffic within the entire pod CIDR range (10.244.0.0/16), ensuring that packets between containers can be forwarded. 

Now try to ping other containers in the same host (e.g `worker-1` containers)

```bash
kubectl exec -it bash-worker-1 -- bash
```
![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/9.png)

ping was successful.

### **Fixing External Access Using NAT**

Containers in a private subnet (e.g., 10.244.0.0/24) cannot access the Internet because their IP addresses are not routable outside their private network. To enable this access, Network Address Translation (NAT) is used to map the private IPs to the public IP of the host.

**How NAT Works:**

- **Outbound Traffic:** When a packet from a container tries to reach the Internet, its source IP address (from the private subnet) is replaced with the host's public IP address. This is done by NAT on the host VM. The original IP address is saved in a table so that responses can be routed back to the correct container.

- **Inbound Traffic:** When a response packet comes back to the host, NAT looks up the saved table, replaces the public IP with the original private IP, and forwards the packet to the appropriate container.

**Setting Up NAT:**

- On the **master node**:

  ```bash
  sudo iptables -t nat -A POSTROUTING -s 10.244.0.0/24 ! -o cni0 -j MASQUERADE
  ```

- On **worker nodes 1**:

  ```bash
  sudo iptables -t nat -A POSTROUTING -s 10.244.1.0/24 ! -o cni0 -j MASQUERADE  # worker-1
  ```
- On **worker nodes 2**:

  ```bash
  sudo iptables -t nat -A POSTROUTING -s 10.244.2.0/24 ! -o cni0 -j MASQUERADE  # worker-2
  ```

  After setting up these NAT rules, containers will be able to access the Internet

  ![](https://raw.githubusercontent.com/Galadon123/CNI-Plug-in-with-Bash/f0cb507bd65dc68805d769c7294162194d8922de/images/10.png)
 
## **Conclusion**

This guide takes you through the full process of provisioning infrastructure using Terraform, deploying a Kubernetes cluster, and implementing a **custom CNI plug-in** for pod networking.
