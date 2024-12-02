# Amazon EBS Volume Lab

## Launch Instances in two AZs in EC2

1. Launch an instance using the Amazon Linux AMI in us-east-1a
steps:
	EC2 console, Instance, Launch Instances
	Name = 1A
	Key pair name - required = Proceed without a key pair
	Network settings:
		Subnet = us-east-1a
		Common security groups = WebAccess
	Launch instance
	
2. Launch another instnace using the Amazon Linux AMI in us-east-1b
steps:
	EC2 console, Instance, Launch Instances
	Name = 1B
	Key pair name - required = Proceed without a key pair
	Network settings:
		Subnet = us-east-1b
		Common security groups = WebAccess
	Launch instance
	
## Create and Attach an EBS Volume
1. Create a 10GB gp2 volume in us-east-1a with a name tag of 'data-volume'
steps:
	EC2, Elastic Block Store, Volume, show root volumes for our instances
	Create Volume
	
2. List non-loopback block devices on instance
sudo lsblk -e7

steps:
	EC2, Instances, select my instance, Connect
	Connect Type, Connect using EC2 Instance Connect
		command:
		sudo lsblk -e7 // list non loop back block devices on the instance
		
3. Attach the volume to the instance in us-east-1a
4. Rerun the command to view block devices
steps:	
	EC2, Volumes, select my volume, Actions/Attach volume, select the instance, Attach volume
		command:
		sudo lsblk -e7 // now we can see my new volumen attached
		
## Create a filesystem and mount the volume
1. Create a filesystem on the EBS volume
sudo mkfs -t ext4 /dev/xvdf		// mkfs = make file system 

2. Create a mount point for the EBS volume
sudo mkdir /data		// create /data directory

ls / 		// list directory
clear

3. Mount the EBS volume to the mount point
sudo mount /dev/xvdf /data
4. Make the volume mount persistent
Run: 'sudo nano /etc/fstab' 		// open file
then add '/dev/xvdf /data ext4 defaults,nofail 0 2' and save the file

cat /etc/fstab		// see my entry

## Add some data to the volume

1. Change to the /data mount point directory
2. Create some files and folders
steps:
	cd /data		// go directory
	touch testfile.txt	// create file
	sudo touch testfile.txt 	// give permission
	ls
	sudo mkdir myfolder

to prove that when we move a file to another instance, we should see the same data.
We have data on the drive, drive mounted to our Linux instance.

## Take a snapshot and move the volume to us-east-1b

1. Take a snapshot of the data volume
2. Create a new EBS volume from the snapshot in us-east-1b
3. Mount the new EBS volume to the instance in us-east-1b
4. Change to the /data mount point and view the data
steps:
	EC2, Volumes, select volume, Actions/Create Snapshot
	Snapshots
