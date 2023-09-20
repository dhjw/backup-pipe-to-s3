## Backup archive and encrypt with pipe to Amazon S3
The archive is never stored locally, saving disk space. This is great for VPSs with limited storage, or anyone who doesn't want to deal with copies of large amounts of sensitive data.

### Setup
- Change to root with `sudo su -`, then install the requirements (example for Ubuntu/Debian)
	```
	apt install ccrypt pigz pv pip
	pip install awscli
	```
- On the AWS website/console, create an S3 [bucket](https://console.aws.amazon.com/s3/buckets/) named something like `myserver-backups`
- Also create an IAM [user](https://console.aws.amazon.com/iam/home#/users) with the appropriate policy to access the bucket:
  - Click `Create user`
  - Name the user something like `myserver-backups-user` and click `Next`
  - Choose `Attach policies directly` and click `Next`
  - Click `Create user` at the bottom (no permissions yet)
  - Click the user to edit it
  - Click the `Add permissions` dropdown and choose `Create inline policy`
  - Click `JSON`
  - Copy and paste the following, changing `myserver-backups` to your bucket name
	```
	{
		"Version": "2012-10-17",
		"Statement": [
			{
				"Effect": "Allow",
				"Action": [
					"s3:ListBucket"
				],
				"Resource": [
					"arn:aws:s3:::myserver-backups"
				]
			},
			{
				"Effect": "Allow",
				"Action": [
					"s3:PutObject",
					"s3:GetObject",
					"s3:DeleteObject"
				],
				"Resource": [
					"arn:aws:s3:::myserver-backups/*"
				]
			}
		]
	}
	```
  - After entering the JSON, click `Next`
  - Give the policy a name like `myserver-backups-policy`, then click `Create policy`
  - Click `Create access key`
  - Set `Use case` to `Command Line Interface (CLI)`
  - Check the confirmation box and click `Next`
 
- Run `aws configure` to add the public and private access keys to aws-cli. They will be saved to `/root/.aws/credentials`. 
  
  You can test authorization with a command like:
	```
	aws s3 ls s3://myserver-backups
	```
- Create a simple encryption key:
	```
	openssl rand -hex 64 > /root/.bk
	chmod 600 /root/.bk
	cat /root/.bk
	```
	Make sure to save the key to other computers, a keepass2 vault on the cloud, a usb stick, etc.

- Download and configure the `backup` script and `backup.cfg` (careful, this will destroy any existing backup.cfg). First change to root with `sudo su -`, then:
	```
	cd /usr/local/bin
	wget https://raw.githubusercontent.com/dhjw/backup-pipe-to-s3/main/backup
	wget https://raw.githubusercontent.com/dhjw/backup-pipe-to-s3/main/backup.cfg.example -O backup.cfg
	chmod +x backup
	chmod 600 backup.cfg
	nano backup.cfg # or gedit, etc.	
	```
- Replace the example files and exclusions arrays with your own. If there are no exclusions leave it defined but empty.
- You can likely save money with storage class ONEZONE_IA (min 30-day storage) or GLACIER_IR (min 90-day storage) depending on how many backups you prefer to keep ([Pricing](https://aws.amazon.com/s3/pricing/)). Basically, at the time of writing, you can store 90 days of backups instead of 30, with extra redundancy, for only 20% more by using GLACIER_IR instead of ONEZONE_IA. Set the `numkeep` variable in backup.cfg to match your plan.

That's the end of the setup instructions.

### Usage
- Run `backup` manually
- Set up a cron job as root, e.g.:
	```
	SHELL=/bin/bash
	PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
	MAILTO=root

	# m h  dom mon dow   command
	0 0 * * * backup > /var/log/backup.log
	```

### Download and extract the last backup (with pipe)
First change to root with `sudo su -`, then:

```
tmpdir=$(mktemp -d -p /root)
cd $tmpdir
source /usr/local/bin/backup.cfg
file=$(aws s3 ls s3://$bucket/backup_$(hostname)_ | sort | tail -n1 | awk '{print $4}')
echo "Downloading and extracting $file"
aws s3 cp "s3://$bucket/$file" - | pv | ccdecrypt -k $keyfile | tar xzf -
ls
```
When you're done, remove the temporary folder
```
rm -rf $tmpdir
```

### Update
Copy the latest `backup` script file over your local copy:
```
sudo wget https://raw.githubusercontent.com/dhjw/backup-pipe-to-s3/main/backup -O /usr/local/bin/backup
```