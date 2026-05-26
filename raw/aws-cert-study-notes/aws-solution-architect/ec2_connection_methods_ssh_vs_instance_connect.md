# EC2: SSH vs EC2 Instance Connect

EC2 supports several connection methods (SSH, EC2 Instance Connect, Session Manager via SSM, EC2 Instance Connect Endpoint, Serial Console, RDP for Windows, etc.). This note covers the two demoed in the video — **SSH** and **EC2 Instance Connect** — which have very different trust models and network paths.

---

## 1. SSH (Local SSH Client)

Classic key-based SSH from your machine to the instance on port 22.

- **Auth:** SSH key pair — `.pem` file on your local machine.
- **Network path:** your machine → EC2 instance (direct).
- **Security group requirement:** must allow **your local IP** on port 22.
- **Scaling pain:** every user needs a copy of the private key. Hard to manage for teams.

### ⚠️ `.pem` Key Permissions

SSH refuses to use a private key that is readable by other users on your machine. If permissions are too open, you'll get a `"permissions are too open"` / `UNPROTECTED PRIVATE KEY FILE` error.

Fix it so only you can read the key:

- **macOS / Linux** — `chmod 400 <key>.pem` (read-only, owner only).
- **Windows** — file **Properties → Security**: remove all users except your own.

Rule of thumb: the private key must be **owner-readable only**.

---

## 2. EC2 Instance Connect

Browser-based connection from the AWS console — no local key needed.

- **Auth:** AWS **IAM identity policy** (user/group/role permissions), not an SSH key.
- **Scales naturally:** grant/revoke access via IAM instead of distributing key files.
- **Username:** AWS guesses from the AMI; for custom AMIs you may need to set it manually.

### How it actually works (the important part)

Even though you click "Connect" in your browser, the SSH connection to your instance is **not** coming from your machine. The real path is:

```
Your browser  →  AWS EC2 Instance Connect service  →  Your EC2 instance
```

The Instance Connect service pushes a short-lived public key to the instance and opens the SSH session from **AWS-owned IPs**, not yours. This single fact drives the security group rule below.

📖 Reference: [EC2 Instance Connect — connection methods](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html)

---

## ⚠️ Security Group Best Practice for Port 22

**Avoid `0.0.0.0/0` on port 22.** Opening SSH to the entire internet is the classic bad-practice finding.

What to allow instead depends on which method you use:

|Connection method|Allow on port 22|
|---|---|
|Local SSH client|Your own IP (e.g. **My IP**)|
|EC2 Instance Connect|The **AWS Instance Connect service IP range** for your region|
|Both|Both of the above|

If you lock the SG down to **My IP** only, Instance Connect breaks — because the actual connection originates from AWS, not from you.

### Getting the Instance Connect IP range

AWS publishes all service IP ranges as a single JSON document:

🔗 **https://ip-ranges.amazonaws.com/ip-ranges.json**

To find the right CIDR, filter the `prefixes` array for entries where:

- `service` = `EC2_INSTANCE_CONNECT`
- `region` = your instance's region

Then add the matching `ip_prefix` value(s) to your security group as the source for port 22. This keeps SSH access scoped instead of open to the world.

> The same JSON file is the canonical source for any "what IPs does AWS service X use?" question — useful well beyond Instance Connect.

---

## Summary

- **SSH** — key-based, connects from _your machine_, doesn't scale well across teams, watch your `.pem` permissions.
- **EC2 Instance Connect** — IAM-based, connects from _AWS service IPs_, scales well, requires the service IP range in your SG.
- For any port-22 method, **avoid `0.0.0.0/0`** in production — scope the source to My IP, the Instance Connect range, or both, depending on which methods you actually use.
- For private instances or fleets at scale, other methods (Session Manager, Instance Connect Endpoint) are often a better fit than either of the above.