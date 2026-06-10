# EC2 Bootstrapping with User Data

> **Scope:** Core SAA material. This note covers the concept of bootstrapping and the user-data mechanism that implements it in EC2.

---

## 1. What Bootstrapping Is

**Bootstrapping** is a general systems-automation concept: a process that lets a system **self-configure** when it first comes online. It's not AWS-specific — it exists outside EC2 as a general term.

In EC2 specifically, bootstrapping enables **build automation**: instead of an instance launching into the default state of its AMI, you can direct it to perform software installs and post-install configuration so it comes into service in a **pre-configured state**.

This matters for solutions architects because it's the foundation for adding **automation** to designs.

---

## 2. EC2 User Data — The Mechanism

Bootstrapping in EC2 is enabled via **EC2 user data**.

### How it's delivered

- User data is injected into the instance the **same way metadata is** — via the metadata service.
- It's accessed at the metadata IP `169.254.169.254` (Cantrill's mnemonic: _"169.254 repeating"_).
    - Metadata path: `/latest/meta-data/`
    - User data path: `/latest/user-data/`
    - _(These IPs/paths are fixed AWS conventions — the IP itself is a known constant worth recognizing.)_

> 🎯 **Trigger keyword:** "run a script at launch" / "configure an instance on first boot" → **user data**.

### How it executes

- Anything you pass in is **executed by the instance's OS**, run by a process on the OS as the **root user**.
- It is executed **only ONCE, at initial launch time.**
- A service on the OS watches the metadata IP for user data; if found, it runs it on launch.

> ⚠️ **Trap — "only once":** If you stop the instance, **edit** the user data, and start it again, the _new_ data is **visible** inside the instance, but it is **NOT re-executed**. Execution happens only on the **first** launch. After launch, user data is only useful for passing data _in_.

### EC2 treats it as opaque

- EC2 **does not validate or interpret** user data in any way.
- Pass in random data → it passes it in. Pass in commands that wipe the boot volume → the instance runs them.
- The **instance OS must understand** what's passed in, because EC2 just hands it over blindly.

### Outcome states after user data runs

|Outcome|Instance state|Status checks|Result|
|---|---|---|---|
|Script succeeds|Running|Pass|Configured & ready for service|
|Script errors|**Still running**|**Still pass**|**Bad config** — instance works mechanically but isn't set up as intended|

> ⚠️ **Trap:** A failed bootstrap does **not** fail the instance. Because user data is separate from EC2, the instance still runs and passes status checks — you just get a broken/misconfigured box. This is the powerful-but-risky nature of user data.

### Key constraints

- **Not secure:** anyone who can access the instance OS can read the user data. **Don't pass long-term credentials** through it.
    - _(Cantrill's demo intentionally does this as bad practice, to later contrast with proper secret-handling AWS services.)_
- **Size limit: 16 KB.** For anything larger, pass in a small script that **downloads** the larger payload.
- **Base64:** EC2 expects user data as base64. The **console UI auto-converts** plain text for you (tick the "already base64 encoded" box only if your input is already encoded). **CloudFormation does not** — you must run plain text through a base64 function (`Fn::Base64`).

### Architecture

![[ec2-bootstrapping.png]]

**Reading the diagram:**

- **Top row (the normal launch path):** an **AMI** (left) launches an **Instance** (centre), which gets an attached EBS volume (the green box hanging below it). This part is the standard launch you already know.
- **Top-right:** the **EC2 service** delivers **User Data** into the instance (blue arrow). A process inside the OS picks it up.
- **Following the pink path down and around:** the instance reaches the **"Execute User Data"** step (clock icon = this happens at launch). From there the path **forks into two outcomes**:
    - **Bad Config** (sad-face instance, upper right) — the user data errored. Note the instance is _still there and running_; it's just misconfigured.
    - **Running / Ready for Service** (green text, bottom left) — the user data succeeded and the instance is properly configured.
- **Footer takeaway:** the same launch can end in either state, and EC2 itself can't tell the difference — both are "running" with passing status checks. The fork is the whole point of the diagram.

---

## 3. Boot Time to Service Time (Exam Concept)

A common exam angle: **how quickly can an instance be brought into service?**

**Boot time to service time** = total time from launch until the instance can accept customer connections. It includes:

1. Time for AWS to provision the instance, **plus**
2. **Post-launch time** — time for software updates, installs, and configuration before it's ready.

- Plain AWS-provided AMI → measured in **minutes**.
- Manual post-launch configuration → minutes to **hours** for complex setups.

### Three ways to handle post-launch work

|Approach|What it does|Trade-off|
|---|---|---|
|**Bootstrapping**|Automates installs/config _after_ launch|Flexible, but still incurs post-launch time|
|**AMI baking**|Front-loads work into a pre-baked AMI|Removes post-launch time, but **less flexible** (config is frozen into the AMI)|
|**Both combined** _(optimal)_|Bake the **time-intensive** part into the AMI, bootstrap the **final config**|Reduces boot-time-to-service-time _and_ keeps configurability|

> 🎯 **Exam pattern:** "minimize boot time to service time while keeping flexibility" → **combine AMI baking + bootstrapping**. e.g., a 90%-install / 10%-config app → bake the 90%, bootstrap the 10%.

### Visualizing the three approaches

![[boot-time-to-service-time-ami-baking-vs-ec2-bootstrapping.png]]

**Reading the diagram (three rows, top to bottom):**

- **Row 1 — plain AMI, no automation:** AMI → Instance takes **minutes**, then there's still **Post Launch Time** (the open-ended pink arrow with a clock) for you to configure things manually. The arrow has no endpoint — that manual tail is the slow, variable part.
- **Row 2 — bootstrapping:** AMI → Instance in **minutes**, then **Bootstrapping** does the post-launch config automatically. Faster and repeatable than the manual tail, but the config work still happens _after_ launch.
- **Row 3 — AMI baking:** time is spent **up front** ("AMI Bake" clock) to produce a pre-configured AMI; that AMI → Instance in **minutes** with little/no post-launch work. Fast to service, but the config is frozen into the image.
- **The dotted box around rows 2 & 3 labelled "Optimal":** combining both — bake the time-intensive parts into the AMI _and_ bootstrap the final config — is the recommended pattern, getting low boot-time-to-service-time _and_ flexibility.

---

## 4. Demo — Inspecting User Data & Metadata (IMDSv2)

Amazon Linux 2023 uses **IMDSv2 (token-based)**, so you must first fetch a token, then pass it as a header on every request.

```bash
# 1. Get a session token (TTL 21600s = 6 hours)
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`

# 2. View instance METADATA using the token
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v \
  http://169.254.169.254/latest/meta-data/

# 3. View USER DATA — same path, swap meta-data → user-data
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v \
  http://169.254.169.254/latest/user-data/
```

> 🎯 **Key takeaway:** **User data is a component of the metadata service** — same IP, just `/user-data/` instead of `/meta-data/`.

### Bootstrap log files (`/var/log/`)

Useful for diagnosing **bootstrapping failures**:

|Log file|Source|
|---|---|
|`cloud-init.log`|User-data / cloud-init processing|
|`cloud-init-output.log`|**Output** of the executed user-data commands (shows actual command output)|

```bash
cd /var/log
ls -la
sudo cat cloud-init-output.log      # diagnose user-data bootstrap
```

> ⚠️ **Trap:** A successful **cow-themed login banner** (the MOTD) appearing on connect is the visual cue that the bootstrap script reached its final step — but always check `cloud-init-output.log` to confirm _all_ steps ran without error.

---

## 5. Demo Artifact — `userdata_AL2023.txt`

The demo installs/configures WordPress (MariaDB + Apache + PHP + WordPress + a Cowsay login banner) as a procedural bash script (`#!/bin/bash -xe`). It's pasted into the **console** User Data box (UI auto-base64-encodes it), or wrapped in `Fn::Base64` inside a CloudFormation template.

> ⚠️ **Bad practice flagged in the course:** DB passwords are **hardcoded in plain text** in the script (`DBPassword='...'`). Intentional teaching example — readable by anyone with OS/user-data access.

Step outline:

1. Set DB name / user / passwords as shell variables
2. `dnf install` web + DB software (wget, php-*, httpd, mariadb105-server, cowsay)
3. `systemctl enable` + `start` httpd and mariadb (enable = start on boot)
4. Set MariaDB root password
5. Download + extract WordPress into `/var/www/html`
6. Configure `wp-config.php` (sed-replace placeholders) + fix permissions
7. Create the WordPress DB/user via a `/tmp/db.setup` script
8. Write the Cowsay MOTD banner + `update-motd`

> 💡 **Note (illustrative values):** `t2.micro`/`t3.micro`, region "Northern Virginia", the `a4l-*` resource names, and the hardcoded passwords are **demo-specific** — patterns to recognize, not values to memorize. Pick whatever instance type is **free-tier eligible** in your account/region.

### Full script

```bash
#!/bin/bash -xe

# STEP 1 - Setpassword & DB Variables
DBName='a4lwordpress'
DBUser='a4lwordpress'
DBPassword='4n1m4l$4L1f3'
DBRootPassword='4n1m4l$4L1f3'

# STEP 2 - Install system software - including Web and DB
dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel cowsay -y
# STEP 3 - Web and DB Servers Online - and set to startup
systemctl enable httpd
systemctl enable mariadb
systemctl start httpd
systemctl start mariadb
# STEP 4 - Set Mariadb Root Password
mysqladmin -u root password $DBRootPassword
# STEP 5 - Install Wordpress
wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz
# STEP 6 - Configure Wordpress
cp ./wp-config-sample.php ./wp-config.php
sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php
# Step 6a - permissions 
usermod -a -G apache ec2-user   
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;
# STEP 7 Create Wordpress DB
echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
mysql -u root --password=$DBRootPassword < /tmp/db.setup
sudo rm /tmp/db.setup
# STEP 8 COWSAY
echo "#!/bin/sh" > /etc/update-motd.d/40-cow
echo 'cowsay "Amazon Linux 2023 AMI - Animals4Life"' >> /etc/update-motd.d/40-cow
chmod 755 /etc/update-motd.d/40-cow
update-motd
```

### Using it from CloudFormation

You can also provide user data inside a CloudFormation template instead of the console. The key difference: the console auto-base64s for you, but in a template you must wrap plain text with the `Fn::Base64` function. The end result is identical — an instance bootstrapped during launch, just deployed via CloudFormation automation.

---

## 6. Exam Recap

- **Bootstrapping** = self-configure at launch; in EC2 it's done via **user data**.
- User data is **executed once at first launch**, run **as root**, and is **opaque to EC2** (not validated).
- Failed bootstrap → instance **still runs & passes checks** → silent **bad config**.
- User data is **not secure** (don't store long-term creds) and is capped at **16 KB**.
- Console **auto-base64s** user data; **CloudFormation requires `Fn::Base64`**.
- Edit + restart does **not** re-run user data — it's one-shot at first launch.
- **Boot time to service time**: minimize with **AMI baking + bootstrapping combined**.
- **IMDSv2**: token first (`PUT .../api/token`), then header on every request. **User data is part of the metadata service** (`/latest/user-data/`).
- Diagnose with `/var/log/cloud-init-output.log`.

---

_Source: Adrian Cantrill SAA course — Advanced EC2 section (Bootstrapping EC2 using User Data; demo PART 1 & 2). Demo artifact: `userdata_AL2023.txt`._