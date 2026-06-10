# Enhanced Bootstrapping with CFN-INIT

> **Scope note:** CFN-INIT, creation policies, and CFN signals are _slightly beyond_ the SAA exam's technical depth — you won't be tested on the implementation detail, but the architecture helps with automation-style questions. Understand the concepts; don't memorize the YAML.
> 
> _Prerequisite: this builds on EC2 user-data bootstrapping — see the companion note._

---

## 1. What CFN-INIT Is

`cfn-init` is a **helper script** (AWS's term) pre-installed on EC2 OSes like Amazon Linux. In practice it behaves more like a **simple configuration management system** — much more powerful than raw user data.

It's a way to pass **complex bootstrapping instructions** into an EC2 instance, with its configuration defined inside a CloudFormation template.

---

## 2. The Central Distinction — Procedural vs. Desired State

||User Data|CFN-INIT|
|---|---|---|
|**Model**|**Procedural** only — a script run line-by-line by the OS|Can be **procedural** _or_ **desired state**|
|**Desired state meaning**|n/a|You declare _what you want_; cfn-init does whatever is needed to reach it|

**Desired-state example:** Tell cfn-init you want Apache version X.

- Already installed at version X → **nothing happens.**
- Not installed → it **installs** it.
- Older version present → it **updates** to X.

> 🎯 This is the key conceptual difference. User data **does**; cfn-init **ensures a state**.

### What cfn-init can do

- Ensure **packages** are installed (version-aware)
- Manipulate OS **groups** and **users**
- Download **sources** and extract them locally (can use authentication)
- Create **files** with specified content, permissions, ownership
- Run **commands** and **test** that conditions hold afterward
- Control **services** (started / enabled on boot)

---

## 3. How It Works

### Where the config lives & how it runs

- Directives are defined in a special section of a logical resource in the **CloudFormation template**: `AWS::CloudFormation::Init` (under the resource's `Metadata`).
- `cfn-init` itself is **invoked from the user data** of the instance — so the standard "user data runs once at launch" rule still applies to the line that _kicks off_ cfn-init.
- The command is passed `--stack` (stack ID) and `--region`. CloudFormation **substitutes** these placeholders (e.g. `${AWS::StackId}`, `${AWS::Region}`) with real values before they reach the instance, so cfn-init knows **which stack/region** to pull config from.

### The flow

1. Template creates a **stack** → stack creates an **EC2 instance**.
2. Instance's **user data** runs `cfn-init` on first launch.
3. `cfn-init` calls the **CloudFormation service**, retrieves the desired-state config from the resource's `Metadata`.
4. `cfn-init` implements that desired state on the instance.

### configSets and config keys

The template's `Metadata` defines one or more **configSets**, each listing ordered **config keys** that run in sequence. In the WordPress demo, the configSet `wordpress_install` runs five keys in order:

`install_cfn → software_install → configure_instance → install_wordpress → configure_wordpress`

> 🎯 **Trigger keyword:** "desired-state config", "complex bootstrapping", "ensure a specific package version" → **cfn-init**.

### Coping with stack updates (CFN-HUP)

- User data runs **only once**. cfn-init can do **more**: it can be configured to **watch for metadata changes** and **re-run** when they occur.
- This is done by a process called **CFN-HUP**, set up to watch the CloudFormation stack and re-trigger `cfn-init` whenever the resource's metadata changes.

> ⚠️ **Comparison trap:** "needs to re-apply config on stack update / not just at first launch" → that's **cfn-init + cfn-hup**, **NOT** user data (user data is one-shot).

### Architecture

![[cfn-init.png]]

**Reading the diagram:**

- **Context — the top row** shows the high-level flow you already know: **Template → Stack → Instance** (solid pink arrows). The template is used to create the stack, which creates the EC2 instance.
- **Left — the Template's contents** (top code block): the `EC2Instance` logical resource has a `CreationPolicy`, plus `Metadata → AWS::CloudFormation::Init` holding the **configSets** and the five config keys (`install_cfn`, `software_install`, `configure_instance`, `install_wordpress`, `configure_wordpress`). This is the desired-state definition.
- **Bottom — the UserData block** (second code block): `Fn::Base64: !Sub` wraps a short script that runs `cfn-init` (highlighted) and then `cfn-signal`. This is the only procedural part — it kicks everything off at launch.
- **The numbered dotted arrows trace the runtime sequence:**
    - **① (orange, instance → up):** the launched instance runs the `cfn-init` line from its user data.
    - **② (orange, cfn-init line → stack):** `cfn-init` uses the substituted `${AWS::StackId}` / `${AWS::Region}` to consult the **stack** and pull down the desired-state config from the metadata.
    - **③ (blue, stack → template/back):** the config is retrieved and applied to the instance, bringing it to desired state.
- **Footer takeaway — "WORKS WITH STACK UPDATES":** unlike one-shot user data, this loop can re-run when the template's metadata changes (via cfn-hup), re-applying desired state.

---

## 4. Creation Policies & CFN-SIGNAL

This addresses a gap: by default, **CloudFormation doesn't know if bootstrapping actually succeeded.**

### The problem

- CloudFormation creates the instance, passes in user data — but it **doesn't understand the user data**.
- If the bootstrap fails, the instance is **still marked `CREATE_COMPLETE`**, regardless of internal config state.
- This is fine for a _blank_ resource (if EC2/S3 says it provisioned OK, it did). It's **not** fine when there's **extra in-instance configuration** happening.

### The solution: Creation Policy + CFN-SIGNAL

- A **CreationPolicy** is attached to a logical resource with a **timeout** (demo uses `PT15M` = 15 min).
- With it, CloudFormation creates a **hold point**: it does **NOT** move the resource to `CREATE_COMPLETE` when EC2 reports the launch succeeded. Instead it **waits for a signal from the resource itself.**
- The instance runs `cfn-signal`, which is given the stack ID, resource name, and region (substituted by CloudFormation).

### Reading the signal command

```bash
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}
```

- **`-e $?`** captures the **exit code of the previous command** (the `cfn-init` run).
    - `cfn-init` OK → exit `0` → success signal sent.
    - `cfn-init` errors → error code sent.

### What CloudFormation does with it

|Signal received|Result|
|---|---|
|Success code|Resource → `CREATE_COMPLETE`|
|Error code|Resource → error|
|**No signal within timeout (15 min)**|CloudFormation **assumes failure** → resource errors, stack doesn't complete|

> 🎯 **Why it matters:** With a creation policy you **explicitly know** the in-instance config finished — you're no longer trusting "EC2 says it's running 2/2 checks." You'll see creation policies on **EC2 instances** and **Auto Scaling Groups** in more complex templates later in the course.

### Architecture

![[cfn-init-CreationPolicy-and-signals.png]]

**Reading the diagram — it contrasts two rows, _without_ vs _with_ a creation policy:**

- **Top row (no creation policy):** Template → Stack → Instance. The instance launches and is marked complete (green tick), but the yellow **"?"** box on the far right is the problem — CloudFormation has **no idea** whether the in-instance bootstrapping actually worked. It trusts EC2's "provisioned" report and moves on.
- **Bottom row (with creation policy):** the `EC2Instance` resource now carries `CreationPolicy → ResourceSignal → Timeout: PT15M` (left code block). The flow Template → Stack → Instance is the same, **but** the stack now **waits** at a hold point instead of auto-completing.
- **The orange dotted arrow (instance/user-data → back to the Stack):** traces the **signal**. The `cfn-signal -e $?` line in the UserData block (highlighted at the bottom) sends the result of `cfn-init` back to the stack. `-e $?` = the exit code of the preceding `cfn-init` command.
- **Resolution:** only when the stack receives a **success** signal does the resource move to `CREATE_COMPLETE` (green-tick instance, bottom right). On an error signal it fails; on **no signal within the 15-minute timeout**, CloudFormation assumes failure.
- **Footer takeaway:** the creation policy converts the top row's blind "?" into the bottom row's explicit, instance-confirmed completion.

---

## 5. Demo — Inspecting & Diagnosing

When you `curl` the user data on a cfn-init instance, it's **short** — only the lines that invoke cfn-init and cfn-signal — and the `${...}` placeholders are **already replaced** with the real stack ID and region (proof of CloudFormation substitution before injection).

```bash
# IMDSv2: token first, then read user data
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v \
  http://169.254.169.254/latest/user-data/
```

### Log files (`/var/log/`)

|Log file|Source|
|---|---|
|`cloud-init.log` / `cloud-init-output.log`|User-data / cloud-init processing|
|`cfn-init-cmd.log`|Output of the **cfn-init** process — shows each config key running _(present when cfn-init is used)_|

```bash
sudo cat /var/log/cfn-init-cmd.log    # see each config key execute
```

> ⚠️ Exact `cfn-init*` log filenames can vary slightly by helper-script version — worth confirming if it comes up.

---

## 6. Demo Artifact — `cfn-init-template.yaml`

Installs/configures the same WordPress stack as the user-data demo, but expressed as **desired state**. Key structural elements:

- **`CreationPolicy: ResourceSignal: Timeout: PT15M`** — the 15-min hold point.
- **`Metadata: AWS::CloudFormation::Init:`** with configSet `wordpress_install` → 5 config keys:
    - `install_cfn` — writes `cfn-hup.conf` + auto-reloader hook; runs `cfn-hup` service (the update-watcher).
    - `software_install` — `commands` to `dnf install` packages; `services` to enable+run `httpd`/`mariadb`.
    - `configure_instance` — `files` creates the `/etc/update-motd.d/40-cow` banner; `commands` set DB root password (with a `test:` guard) + `update-motd`.
    - `install_wordpress` — `sources` auto-downloads & extracts `wordpress.org/latest.tar.gz` into `/var/www/html`; `files` stage the wp-config and db.setup scripts.
    - `configure_wordpress` — `files` stage a permissions-fix script; `commands` create the DB, move WordPress files, tidy up, run config + permissions.
- **`UserData: Fn::Base64: !Sub`** — only **two** lines actually run at launch:
    
    ```bash
    #!/bin/bash -xe/opt/aws/bin/cfn-init  -v  --stack ${AWS::StackId} --resource EC2Instance --configsets wordpress_install --region ${AWS::Region}/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}
    ```
    
    → `cfn-init` does the work; `cfn-signal` reports the result back to satisfy the creation policy.

> 💡 **Note (illustrative values):** `t3.micro`, region "Northern Virginia", the `a4l-vpc1-*` import names, and the hardcoded passwords are **demo-specific** — patterns to recognize, not values to memorize.

### Full template

```yaml
Description:  A4L CFN-INIT Wordpress Template 
  Requires A4L VPC Template to run
Parameters:
  LatestAmiId:
    Description: AMI for Instance (default is latest AmaLinux2023)
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64'
  DBName:
    AllowedPattern: '[a-zA-Z][a-zA-Z0-9]*'
    ConstraintDescription: must begin with a letter and contain only alphanumeric
      characters.
    Default: a4lwordpress
    Description: The WordPress database name
    MaxLength: '64'
    MinLength: '1'
    Type: String
  DBPassword:
    AllowedPattern: '[a-zA-Z0-9]*'
    ConstraintDescription: must contain only alphanumeric characters.
    Description: The WordPress database admin account password
    MaxLength: '41'
    MinLength: '8'
    NoEcho: 'true'
    Default: 'Sup3rS3cr3tP4ssw0rd'
    Type: String
  DBRootPassword:
    AllowedPattern: '[a-zA-Z0-9]*'
    ConstraintDescription: must contain only alphanumeric characters.
    Description: MySQL root password
    MaxLength: '41'
    MinLength: '8'
    NoEcho: 'true'
    Default: 'Sup3rS3cr3tP4ssw0rd'
    Type: String
  DBUser:
    AllowedPattern: '[a-zA-Z][a-zA-Z0-9]*'
    ConstraintDescription: must begin with a letter and contain only alphanumeric
      characters.
    Description: The WordPress database admin account username
    Default: a4lwordpress
    MaxLength: '16'
    MinLength: '1'
    Type: String
Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          wordpress_install:
            - install_cfn
            - software_install
            - configure_instance
            - install_wordpress
            - configure_wordpress
        install_cfn:
          files:
            /etc/cfn/cfn-hup.conf:
              content: !Sub |
                [main]
                stack= ${AWS::StackId}
                region=${AWS::Region}
              group: root
              mode: '000400'
              owner: root
            /etc/cfn/hooks.d/cfn-auto-reloader.conf:
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.EC2Instance.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource EC2Instance --configsets wordpress_install --region ${AWS::Region}
              group: root
              mode: '000400'
              owner: root
          services:
            sysvinit:
              cfn-hup:
                enabled: true
                ensureRunning: true
                files:
                - /etc/cfn/cfn-hup.conf
                - /etc/cfn/hooks.d/cfn-auto-reloader.conf
        software_install:
          commands:
            0_manual_installs:
              command: dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel cowsay -y
          services:
            sysvinit:
              httpd:
                enabled: true
                ensureRunning: true
              mariadb:
                enabled: true
                ensureRunning: true
        configure_instance:
          files:
            /etc/update-motd.d/40-cow:
              content: !Sub |
                #!/bin/sh
                cowsay "Amazon Linux 2023 AMI - Animals4Life"
              group: root
              mode: '000755'
              owner: root
          commands:
            01_set_mysql_root_password:
              command: !Sub |
                mysqladmin -u root password '${DBRootPassword}'
              test: !Sub |
                $(mysql ${DBName} -u root --password='${DBRootPassword}' >/dev/null 2>&1 </dev/null); (( $? != 0 ))
            02_updatemotd:
              command: update-motd
        install_wordpress:
          sources:
            /var/www/html: http://wordpress.org/latest.tar.gz
          files:
            /tmp/create-wp-config:
              content: !Sub |
                #!/bin/bash -xe
                cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
                sed -i "s/'database_name_here'/'${DBName}'/g" wp-config.php
                sed -i "s/'username_here'/'${DBUser}'/g" wp-config.php
                sed -i "s/'password_here'/'${DBPassword}'/g" wp-config.php                                    
              group: root
              mode: '000500'
              owner: root
            /tmp/db.setup:
              content: !Sub |
                CREATE DATABASE ${DBName};
                CREATE USER '${DBUser}'@'localhost' IDENTIFIED BY '${DBPassword}';
                GRANT ALL ON ${DBName}.* TO '${DBUser}'@'localhost'; 
                FLUSH PRIVILEGES;
              group: root
              mode: '000400'
              owner: root
        configure_wordpress:
          files:
            /tmp/permissionsfix:
              content: !Sub |
                usermod -a -G apache ec2-user   
                chown -R ec2-user:apache /var/www
                chmod 2775 /var/www
                find /var/www -type d -exec chmod 2775 {} \;
                find /var/www -type f -exec chmod 0664 {} \;
              group: root
              mode: '000500'
              owner: root
          commands:
            01_create_database:
              command: !Sub |  
                mysql -u root --password='${DBRootPassword}' < /tmp/db.setup
              test: !Sub |
                $(mysql ${DBName} -u root --password='${DBRootPassword}' >/dev/null 2>&1 </dev/null); (( $? !=0))
            02_move_wordpress:
              command: !Sub |
                cp -rvf /var/www/html/wordpress/* /var/www/html/
            03_tidyup:
              command: !Sub |
                rm -R /var/www/html/wordpress
            04_configure_wordpress:
              command: /tmp/create-wp-config
              cwd: /var/www/html
            04_fix_permissions:
              command: /tmp/permissionsfix
    Properties:
      InstanceType: "t3.micro"
      ImageId: !Ref LatestAmiId
      SubnetId: !ImportValue a4l-vpc1-subnet-weba
      SecurityGroupIds: 
        - !ImportValue a4l-vpc1-default-instance-sg
      Tags:
        - Key: Name
          Value: A4L-Wordpress
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackId} --resource EC2Instance --configsets wordpress_install --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}
```

---

## 7. User Data vs CFN-INIT — Quick Comparison

|Feature|**User Data**|**CFN-INIT**|
|---|---|---|
|Configuration model|Procedural (script, line-by-line)|Procedural **and** desired-state|
|Execution frequency|**Once**, at first launch only|Can **re-run** on stack updates (via cfn-hup)|
|Version-aware package install|No (you script it manually)|Yes (declare desired version)|
|Where config is stored|The user data block itself|`AWS::CloudFormation::Init` metadata in the template|
|How it's invoked|OS runs it directly at launch|Invoked **from** user data (`cfn-init` command)|
|Knows if config succeeded?|No (instance marked complete regardless)|Yes, when paired with **CreationPolicy + cfn-signal**|
|Complexity|Simple|More complex, more powerful|
|Exam relevance|**Core**|Architecture awareness only|

**One-line mental model:**

- **User data** = "run this script once, hope it works."
- **cfn-init** = "here's the state I want; achieve it, re-apply on change, and report back whether you succeeded."

---

## 8. Exam Recap

- **cfn-init** = desired-state configuration management — can be **version-aware** and **re-run on updates** (via cfn-hup); user data is one-shot procedural.
- Config lives in **`AWS::CloudFormation::Init`** metadata; cfn-init is **invoked from the user data**.
- CloudFormation **substitutes** `${AWS::StackId}` / `${AWS::Region}` before injecting, so cfn-init can reach the right stack.
- **CreationPolicy + cfn-signal** = the only way CloudFormation **knows** in-instance bootstrapping actually succeeded — it **waits for a signal**, with a **timeout** (assumes failure if none arrives).
- **`-e $?`** in cfn-signal passes the exit code of the preceding cfn-init run as the success/failure signal.
- Creation policies appear on **EC2 instances** and **Auto Scaling Groups**.
- Diagnose with `/var/log/cfn-init-cmd.log`.

---

_Source: Adrian Cantrill SAA course — Advanced EC2 section (Enhanced Bootstrapping with CFN-INIT; demo CFN-INIT and CFN Creation Policies). Demo artifact: `cfn-init-template.yaml`._