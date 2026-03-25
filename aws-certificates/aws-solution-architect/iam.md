# IAM

1. IAM is an IDP, aka, identify provider. It authenticates + authorizes identities.
2. Username is required for an IAM user, but password is optional, as some IAM users only need to access AWS resources via CLI, not AWS Console.
3. An IAM Access Key works like a set of credentials—similar to a username and password. It contains an Access Key ID and a Secret Access Key. The Access Key ID acts as the username, while the Secret Access Key acts as the password. Never share the Secret Access Key.
4. Configure AWS CLI using named profile, which provides the possibility of managing multiple IAM users in different AWS accounts at the same time.
   ```bash
   # Install the AWS CLI following the instructions here: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
   
   # Check installation status
   aws --version
   
   # expected output aws-cli/2.33.4 Python/3.13.11 Darwin/25.2.0 source/arm64
   
   # Configue using name profile
   # (if using aws configure without specificing the --profile option,
   # the default profile will be used)
   aws configure --profile iamadmin-general
   # ... 
   aws configure --profile iamadmin-production
   # ... 
   
   # Test configuration
   aws s3 ls --profile iamadmin-general
   aws s3 ls --profile iamadmin-production
   ```
