
# Working with EFS

## Launch instances in multiple AZs
1. Create a security group

command:
aws ec2 create-security-group --group-name StorageLabs --description "Temporary SG for the Storage Service Labs"
	
response:
{
	"GroupId": "sg-0fefe8c9644d7353d", 	// <SECURITY-GROUP-ID>
	"SecurityGroupArn": "arn:aws:ec2:us-east-1:436488467655:security-group/sg-0fefe8c9644d7353d"
}

2. Add a rule for SSH inbound to the security group

command:
aws ec2 authorize-security-group-ingress --group-name StorageLabs --protocol tcp --port 22 --cidr 0.0.0.0/0

response:
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0c691e585ce53d4dc",
            "GroupId": "sg-0fefe8c9644d7353d",
            "GroupOwnerId": "436488467655",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:436488467655:security-group-rule/sgr-0c691e585ce53d4dc"
        }
    ]
}

3. Launch instance in US-EAST-1A

command:
aws ec2 run-instances --image-id <LATEST-AMI-ID> --instance-type t2.micro --placement AvailabilityZone=us-east-1a --security-group-ids <SECURITY-GROUP-ID>

	<LATEST-AMI-ID>: go EC2, Instances, Launch Instances, AMID ID = ami-0453ec754f44f9a4a
	
	ex:
	aws ec2 run-instances --image-id ami-0453ec754f44f9a4a --instance-type t2.micro --placement AvailabilityZone=us-east-1a --security-group-ids sg-0fefe8c9644d7353d
	
	aws ec2 run-instances --image-id ami-0440d3b780d96b29d --instance-type t2.micro --placement AvailabilityZone=us-east-1a --security-group-ids <SECURITY-GROUP-ID>

4. Launch instance in US-EAST-1B

command:
aws ec2 run-instances --image-id <LATEST-AMI-ID> --instance-type t2.micro --placement AvailabilityZone=us-east-1a --security-group-ids <SECURITY-GROUP-ID>

	ex:
	aws ec2 run-instances --image-id ami-0453ec754f44f9a4a --instance-type t2.micro --placement AvailabilityZone=us-east-1b --security-group-ids sg-0fefe8c9644d7353d
	
	aws ec2 run-instances --image-id ami-0440d3b780d96b29d --instance-type t2.micro --placement AvailabilityZone=us-east-1b --security-group-ids <SECURITY-GROUP-ID>

command:
	q	// exit
	
## Create an EFS File System
1. Add a rule to the security group to allow the NFS protocol from group members

command:
aws ec2 authorize-security-group-ingress --group-id <SECURITY-GROUP-ID> --protocol tcp --port 2049 --source-group <SECURITY-GROUP-ID>

	ex:
	aws ec2 authorize-security-group-ingress --group-id sg-0fefe8c9644d7353d --protocol tcp --port 2049 --source-group sg-0fefe8c9644d7353d

Go EC2, Network & Security, Security Groups, find my Security Group created = StorageLabs, Inbound rules

2. Create an EFS file system through the console, and add the StorageLabs security group to the mount targets for each AZ

steps:
Go EFS, Create file system
	Name = MyEFS
Next, choose Availability zone = us-east-1a, us-east-1b, each one with Security groups = StorageLabs
Next, File system policy - optional is a Resource-Based policy, Create

## Mount using the NFS Client (perform steps on both instances)
For log into both instances.
Have 2 instances and 2 different availability zones that are sharing (read/write) the same shared file system.

EC2 -> instances -> EFS

1. Create an EFS mount point

Go EC2, Instances, Connect 2 instances (us-east-1a, us-east-1b)

command: mkdir ~/efs-mount-point

2. Install NFS client utils
sudo yum -y install nfs-utils

3. Mount using the EFS client

command:
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <EFS-DNS-NAME>:/ ~/efs-mount-point

	<EFS-DNS-NAME>: find in EFS, select my EFS, copy DNS name
	
	ex:
	sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-080f47292dc2567c5.efs.us-east-1.amazonaws.com:/ ~/efs-mount-point

4. Create a file on the file system

command:
cd ~/efs-mount-point	// change directory
sudo touch instance1.txt	// for 2nd instance: sudo touch instance2.txt
ls	// list, can see exisst 2 files instance1.txt, instance2.txt in both instances

5. Add a file system policy to enforce encryption in-transit

Go EFS, File system policy, select my EFS, File system policy, Edit, check "
Enforce in-transit encryption for all clients", Save

6. Unmount (make sure to change directory out of efs-mount-point first)

command:
cd -- 	// change directory
cd -
sudo umount ~/efs-mount-point

3. Rerun step 3: Mount again using the EFS client (what happens?)

command:
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-080f47292dc2567c5.efs.us-east-1.amazonaws.com:/ ~/efs-mount-point

response:
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-080f47292dc2567c5.efs.us-east-1.amazonaws.com:/ ~/efs-mount-point
	we can't mount because simply the NFS tools do not support this option for encryption in transit. Solution is do the next step "Mount using the EFS utils"

## Mount using the EFS utils (perform steps on both instances)
We need to use EFS utils rather than the regular NFS client.

1. Install EFS utils
sudo yum install -y amazon-efs-utils

2. Mount using the EFS mount helper

command:
sudo mount -t efs -o tls <EFS-DNS-NAME>:/ ~/efs-mount-point

	tls = Transport Level Security, enable encryption
	
	ex:
	sudo mount -t efs -o tls fs-080f47292dc2567c5.efs.us-east-1.amazonaws.com:/ ~/efs-mount-point
	
cd ~/efs-mount-point	// change directory
ls	// show instance1.txt, instance2.txt
