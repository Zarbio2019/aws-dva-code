# IMDS v1: Instance Metadata
allow IMD version, operate without authentication token

## Example commmands to run:

0. Create an instance:

Go EC2, Instances, Launch instances
	Name = IMDSv1
	Key pair name - required = Proceed without a key pair (Not recommended)
	Common security groups = WebAccess // this allow port 22 (for remote secure shell) and port 80 (to run a web server)
	Metadata accessible = Enabled
	Metadata version = V1 and V2 (token optional) **
		EC2 recommends using metadata version 2 unless you explicitly require metadata version 1.

	Connect, Connect using EC2 Instance Connect, Connect
	
1. Get the instance ID:
curl http://169.254.169.254/latest/meta-data/instance-id
	curl = utility for making http connections
	http://169.254.169.254/latest/meta-data/instance-id = local address for the metadata service

response:
i-023a98468d5e5b9d0[ec2-user@ip-172-31-30-30 ~]$	// i-023a98468d5e5b9d0 = instance ID
	
2. Get the AMI ID:
curl http://169.254.169.254/latest/meta-data/ami-id

response:
ami-0453ec754f44f9a4a[ec2-user@ip-172-31-30-30 ~]$	// ami-0453ec754f44f9a4a = AMI ID

obs: retrieve instance information:
curl http://169.254.169.254/latest/meta-data

3. Get the instance type:
curl http://169.254.169.254/latest/meta-data/instance-type

4. Get the local IPv4 address:
curl http://169.254.169.254/latest/meta-data/local-ipv4

5. Get the public IPv4 address:
curl http://169.254.169.254/latest/meta-data/public-ipv4


# IMDS v2: Instance Metadata
have default settings, don't allow IMD version.
we have to use token based authentication.
To retrieve info like IMDS v1 it return Unauthorized because you need authorization token

0. Create an instance:

Go EC2, Instances, Launch instances
	Name = IMDSv2
	Key pair name - required = Proceed without a key pair (Not recommended)
	Common security groups = WebAccess // this allow port 22 (for remote secure shell) and port 80 (to run a web server)
									   // both 0.0.0.0/0 = any source
	Metadata accessible = Enabled
	Metadata version = V2 only (token required) **
		For V2 requests, you must include a session token in all instance metadata requests. Applications or agents that use V1 for instance metadata access will break.
	
	Connect, Connect using EC2 Instance Connect, Connect
	
## Step 1 - Create a session and get an authorization token

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
	// authorization token is stored in the environment variable TOKEN

echo $TOKEN		// get the value

## Step 2 - Use the token to request metadata

1. Get the instance ID:
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
	// with TOKEN

response:
i-023a98468d5e5b9d0[ec2-user@ip-172-31-30-30 ~]$	// i-023a98468d5e5b9d0 = instance ID

2. Get the AMI ID:
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id

response:
ami-0453ec754f44f9a4a[ec2-user@ip-172-31-30-30 ~]$	// ami-0453ec754f44f9a4a = AMI ID

# Use metadata with user data to configure the instance

This script installs a web server and uses instance metadata to retrieve information about the instance and then output the information on a webpage.

```bash		// is a bash script
#!/bin/bash

# Update system and install httpd (Apache)
yum update -y		// update the latest patches
yum install -y httpd	// to install Apache web service

# Start httpd service and enable it to start on boot
systemctl start httpd
systemctl enable httpd

# Fetch metadata using IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AMI_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id)
INSTANCE_TYPE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type)

# Create a web page to display the metadata
cat <<EOF > /var/www/html/index.html
<html>
<head>
    <title>EC2 Instance Metadata</title>
</head>
<body>
    <h1>EC2 Instance Metadata</h1>
    <p>Instance ID: $INSTANCE_ID</p>
    <p>AMI ID: $AMI_ID</p>
    <p>Instance Type: $INSTANCE_TYPE</p>
</body>
</html>
EOF
```

steps:
Use the connection of IMDS v2
command: nano script.sh		// to create a script
paste this bash script:

#!/bin/bash

# Update system and install httpd (Apache)
yum update -y
yum install -y httpd

# Start httpd service and enable it to start on boot
systemctl start httpd
systemctl enable httpd

# Fetch metadata using IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AMI_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id)
INSTANCE_TYPE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type)

# Create a web page to display the metadata
cat <<EOF > /var/www/html/index.html
<html>
<head>
    <title>EC2 Instance Metadata</title>
</head>
<body>
    <h1>EC2 Instance Metadata</h1>
    <p>Instance ID: $INSTANCE_ID</p>
    <p>AMI ID: $AMI_ID</p>
    <p>Instance Type: $INSTANCE_TYPE</p>
</body>
</html>
EOF

command:
chmod +x script.sh 	// make the script executable
ls -la script.sh		// have the permissions you need read/write
sudo ./script.sh		// run, install (AMI, Apache Web Service), create the custom web page with the metadata on it

In EC2, Instances, select my IMDSv2 instance, copy the Public IPv4 address and paste in the browser and show the web page with info of Instance ID, AMI ID, Instance Type.

# Using Metadata and User data together
EC2, Instances, Launch Instances
	Name = user-data-test
	Key pair name - required = Proceed without a key pair (Not recommended)
	Select existing security group = WebAccess
	Advanced details, User data - optional, paste this script:

#!/bin/bash

# Update system and install httpd (Apache)
yum update -y
yum install -y httpd

# Start httpd service and enable it to start on boot
systemctl start httpd
systemctl enable httpd

# Fetch metadata using IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AMI_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id)
INSTANCE_TYPE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type)

# Create a web page to display the metadata
cat <<EOF > /var/www/html/index.html
<html>
<head>
    <title>EC2 Instance Metadata</title>
</head>
<body>
    <h1>EC2 Instance Metadata</h1>
    <p>Instance ID: $INSTANCE_ID</p>
    <p>AMI ID: $AMI_ID</p>
    <p>Instance Type: $INSTANCE_TYPE</p>
</body>
</html>
EOF

// This user data only happens the first time the instance is run
