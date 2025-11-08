# Highly Available and Secure AWS VPC Architecture — AWS

**Hands-on production deployment — VPC, public/private subnets, NAT, bastion, Auto Scaling, ALB, target groups, NACLs & Security Groups**

---

## Project summary

This project implements a production-grade AWS network and compute architecture: a custom VPC with public and private subnets across two AZs, a bastion host for secure admin access, Auto Scaling group for application instances in private subnets, a public Application Load Balancer (ALB) to serve traffic, NAT Gateways for outbound internet access, and layered security using Security Groups and Network ACLs.


<img width="864" height="678" alt="architecture" src="https://github.com/user-attachments/assets/3257b66e-bce0-4c8f-adef-52ffabfda320" />
<img width="1920" height="971" alt="vpc" src="https://github.com/user-attachments/assets/ebe76813-0e1e-4d08-9eb9-6dc90abe233c" />
<img width="1920" height="967" alt="nacl" src="https://github.com/user-attachments/assets/1c85f6aa-cd8f-4611-82e9-8fef52d56d0c" />
<img width="1920" height="971" alt="sg" src="https://github.com/user-attachments/assets/c3610536-9407-45e6-9ca6-f1bf9a51891f" />
<img width="1920" height="974" alt="Screenshot (90)" src="https://github.com/user-attachments/assets/2b47a433-2db1-4cd0-8b57-d652b2a905a2" />
<img width="1920" height="981" alt="Screenshot (89)" src="https://github.com/user-attachments/assets/0daaa819-1bed-49b3-ad40-26078ccecf87" />
<img width="1920" height="981" alt="Screenshot (94)" src="https://github.com/user-attachments/assets/7d3eb3ae-c3a6-459a-87b9-0fdf05fbed8c" />


---

## Architecture (high level)

* **VPC**: `10.0.0.0/16` (example)
* **Public subnets** (2 AZs): ALB, NAT Gateway(s), Bastion Host
* **Private subnets** (2 AZs): Application EC2 instances (Auto Scaling)
* **Internet Gateway (IGW)** attached to VPC (for ALB public access & NAT)
* **NAT Gateway** in public subnet for private-instance outbound traffic
* **Application Load Balancer (ALB)** in public subnets → forwards to Target Group (instances on port 8000)
* **Bastion host** in public subnet for SSH jump access into private instances
* **Security Groups**:

  * ALB SG: allow inbound 80/443 from Internet
  * Instance SG: allow inbound 8000 only from ALB SG; allow SSH only from Bastion SG (or admin IP)
* **Network ACLs (NACLs)**: subnet-level allow/deny rules; ensure route tables and NACL rules do not block ALB or SSH traffic

---




---

## Step-by-step implementation

### 1. Create VPC and subnets

1. Console: VPC → **Create VPC and more**.
2. Choose IPv4 CIDR (e.g., `10.0.0.0/16`), **2 AZs**, 2 public + 2 private subnets.
3. Leave IPv6 off for this demo.

*Verify:* subnets and route tables created; public subnets route `0.0.0.0/0` → IGW.

---

### 2. Provision NAT Gateway(s) and IGW

1. Create Internet Gateway and **attach** to the VPC.
2. Create Elastic IP(s) and NAT Gateway(s) in public subnet(s).
3. Update private route tables: `0.0.0.0/0` → NAT Gateway.

*Why:* Private instances can reach internet (for updates, APIs) while remaining non-public.

---

### 3. Launch Bastion host (public subnet)

1. EC2 → Launch instance (Ubuntu).
2. Auto-assign Public IP = **Enable**.
3. Security Group: inbound SSH (22) from admin IP only.
4. Place bastion in public subnet.

*Use:* SSH jump host for admin access to private instances. Keep logs/monitoring.

---

### 4. Copying the private key to the Bastion (securely)

If you need the key on the bastion so it can SSH to private instances, copy it securely from your local machine:

1. On your local machine, restrict key permissions:

```bash
chmod 400 ~/Downloads/login-aws.pem
```

2. Copy key to bastion (replace values):

```bash
scp -i ~/Downloads/login-aws.pem ~/Downloads/login-aws.pem ubuntu@<BASTION_PUBLIC_IP>:~
```

3. On the bastion, restrict permissions:

```bash
ssh -i ~/Downloads/login-aws.pem ubuntu@<BASTION_PUBLIC_IP>
# once on bastion
chmod 400 ~/login-aws.pem
```

**Security notes**

* Prefer limiting the key lifecycle on bastion: copy when needed, then remove (`shred` or `rm`) when finished.
* Alternative (safer): use SSH agent forwarding instead of copying the key (see optional section below).

---

### 5. SSH from Bastion to Private Instance

From the bastion (after the key is present and permissions set), SSH into the private instance using its private IP:

```bash
ssh -i ~/login-aws.pem ubuntu@10.x.x.x    # private instance private IP
```

If you copied the key to the bastion, you can also `scp` the key from bastion to the private instance directly (if absolutely necessary):

```bash
# on bastion
scp -i ~/login-aws.pem ~/login-aws.pem ubuntu@10.x.x.x:~
# then on private instance
chmod 400 ~/login-aws.pem
```

---


---

### 7. Build Launch Template / Auto Scaling Group (private instances)

1. Create a **Launch Template** with:

   * AMI: Ubuntu
   * Instance type: `t2.micro` (or desired)
   * User data: bootstrap script to install Nginx or run the app
   * Attach instance role (least privilege)
2. Create **Auto Scaling Group**:

   * Desired capacity: `2`
   * Subnets: private subnets (both AZs)
   * Health checks: ALB target group (if attached)

*Tip:* Keep instances without public IPs for security.

---

### 8. Application server setup

* Install production web server (Nginx) or app container. Example:

```bash
sudo apt update
sudo apt install -y nginx
sudo cp ~/index.html /var/www/html/index.html
sudo systemctl restart nginx
```

* Ensure app listens on the port you register in the target group (e.g., `8000` or `80` behind Nginx).

---

### 9. Create Target Group and ALB

1. EC2 → Target Groups → Create (Protocol HTTP, Target port `8000`).
2. Register Auto Scaling instances as targets.
3. Create ALB (internet-facing) in public subnets, listener on port `80` (or `443` for HTTPS).
4. Listener rule: forward to target group (port `8000`).

**Important:** Use listener on public port (80/443). ALB forwards to instance port 8000. You don’t need an ALB listener on 8000 for public access.

---

### 10. Security Groups & NACLs (correct configuration)

* **ALB SG**: inbound `80` (0.0.0.0/0), outbound to target group SG.
* **Instance SG**: inbound `8000` only from ALB SG (use security-group source); SSH inbound only from Bastion SG or admin IP.
* **NACLs**: subnet-level allow/deny rules — ensure they do not block ALB listeners or SSH. Do not rely on NACLs alone for instance security.



### 11. Testing & verification

1. Access ALB DNS (public) in browser: `http://<alb-dns>` → shows application content.
2. If ALB returns errors:

   * Check ALB security group allows inbound listener port.
   * Check target group health checks (instance responding on the target port).
   * Check instance security group allows traffic from ALB SG.
   * Check NACLs don’t block traffic (look for deny rules with higher priority).

---

## What I built / tested

* Created a **multi-AZ VPC** with public and private subnets.
* Deployed **bastion host** for secure SSH access to private instances.
* Demonstrated copying the private key to the bastion and SSHing from bastion → private instance (and recommended agent forwarding).
* Launched an **Auto Scaling group** with two private EC2 instances running the app.
* Provisioned **NAT Gateway** for outbound internet from private subnets.
* Configured an **Application Load Balancer** (internet-facing) with listener port 80 forwarding to instance port 8000.
* Tested security: Security Groups & NACLs behavior.
* Demonstrated ALB health-check behavior.

=

### 🧠 What I Learned

This project helped me understand how to deploy and manage an **application in a secure AWS VPC architecture** using both **public and private subnets**. I went beyond just a basic hands-on setup and focused on a **production-like environment**.

Key takeaways:

* **VPC Networking:** Created a custom VPC with public and private subnets, attached an Internet Gateway, and configured proper routing so public and private instances could communicate securely.
* **Bastion Host Access:** Learned how to connect to private instances through a bastion (jump) host by securely transferring key pairs using `scp` and then SSH’ing from the bastion into the private EC2 instance.
* **File Transfer & Permissions:** Understood the `scp` command deeply — how to copy files between local and remote machines, and how permissions (`chmod 400`) affect SSH key access.
* **Application Hosting:**

  * Used **Nginx HTTP Server** to host an `index.html` file inside the private instance.
  * Observed how the private server could respond to traffic coming through a Load Balancer.
  * Learned that if the Load Balancer listens on port 80 while the app runs on 8000, the listener must map correctly or the Security Group must open port 80 to make it accessible.
* **Production Readiness:**

  * Understood why production setups usually use **Nginx** or a reverse proxy in front of application servers.
  * Learned how to configure listeners and target groups in a Load Balancer to match the actual app port.
  * Ensured security by restricting SSH and HTTP access using Security Groups instead of opening all ports.
* **Real AWS Flow:**
  Practically saw how traffic flows — from the **internet → Load Balancer → Bastion Host → Private Instance → Application Server** — and how every step affects connectivity and security.

---



---

## What I realized (observations / lessons)

* Misconfiguring NACLs (or having a deny-all rule with higher priority) will silently block traffic even when Security Groups are correct.
* ALB listeners should remain on public ports (80/443) while target groups map to application ports — creating a listener on the application port is usually unnecessary and confusing.
* Use bastion hosts or SSH agent forwarding for admin access — avoid leaving private keys on bastion hosts longer than necessary.
* Automating infrastructure (Terraform/CloudFormation) prevents manual misconfiguration in production.

---

## Troubleshooting checklist

* ALB shows “Listener port not reachable” → Check ALB SG inbound rules.
* Targets show `unhealthy` → curl instance on target port; verify app is listening; confirm security group allows ALB SG.
* SSH works but `scp` permission denied → copy to `~` or `/tmp`, then `sudo mv` if needed; ensure `chmod 400` on keys.
* If you copied the key to bastion and can’t SSH private instance, ensure key permissions on bastion and on private are `chmod 400`.





