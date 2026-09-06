# PBL-1 — Remaining Execution Steps

**Environment:** ap-south-1 · account 872792314971 · instance `i-0016a9e06a7e80388` · EIP `65.2.47.225` · bucket `pbl1-lab-ra2311003011574`

Six screenshots left. Order below is optimised for wall-clock time — start VLE 3 first because it has the longest wait, and capture VLE 1 while it boots.

---

# Step 1 — VLE 3: Infrastructure via CLI (~15 min, mostly waiting)

Run on your **Mac**, not the instance.

## 1.1 Verify CLI identity

```bash
aws sts get-caller-identity
```

Must return `arn:aws:iam::872792314971:user/lab-user`. If it errors, re-run `aws configure` and confirm the region is `ap-south-1`.

## 1.2 Write the script

```bash
cat > provision.sh <<'SCRIPT'
#!/bin/bash
# Reusable AWS infrastructure provisioning script.
# Every run creates a uniquely named, independent environment.
set -euo pipefail

REGION=${REGION:-ap-south-1}
PREFIX=${PREFIX:-cli-lab}
NAME="${PREFIX}-$(date +%s)"
STATE="${NAME}.state"
OWNER=RA2311003011574

export AWS_DEFAULT_REGION=$REGION
save() { echo "$1=$2" >> "$STATE"; }

echo ">>> Environment: $NAME  (region $REGION)"
echo "NAME=$NAME" > "$STATE"
save REGION "$REGION"

echo ">>> VPC"
VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
      --query Vpc.VpcId --output text)
aws ec2 create-tags --resources $VPC \
  --tags Key=Name,Value=$NAME-vpc Key=Owner,Value=$OWNER Key=Project,Value=PBL1
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames
save VPC "$VPC"

echo ">>> Subnet, internet gateway and routing"
SUBNET=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 \
         --query Subnet.SubnetId --output text)
aws ec2 create-tags --resources $SUBNET --tags Key=Name,Value=$NAME-subnet
save SUBNET "$SUBNET"

IGW=$(aws ec2 create-internet-gateway \
      --query InternetGateway.InternetGatewayId --output text)
aws ec2 create-tags --resources $IGW --tags Key=Name,Value=$NAME-igw
aws ec2 attach-internet-gateway --vpc-id $VPC --internet-gateway-id $IGW
save IGW "$IGW"

RT=$(aws ec2 create-route-table --vpc-id $VPC \
     --query RouteTable.RouteTableId --output text)
aws ec2 create-tags --resources $RT --tags Key=Name,Value=$NAME-rt
aws ec2 create-route --route-table-id $RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW >/dev/null
aws ec2 associate-route-table --route-table-id $RT --subnet-id $SUBNET >/dev/null
aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch
save RT "$RT"

echo ">>> Security group"
SG=$(aws ec2 create-security-group --group-name $NAME-sg \
     --description "Created by provision.sh" --vpc-id $VPC \
     --query GroupId --output text)
aws ec2 create-tags --resources $SG --tags Key=Name,Value=$NAME-sg
MYIP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 22 --cidr ${MYIP}/32 >/dev/null
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 >/dev/null
save SG "$SG"

echo ">>> Key pair"
aws ec2 create-key-pair --key-name $NAME-key \
  --query KeyMaterial --output text > $NAME-key.pem
chmod 400 $NAME-key.pem
save KEY "$NAME-key"

echo ">>> AMI lookup"
AMI=$(aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-kernel-6.1-x86_64" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)
echo "    AMI=$AMI"

echo ">>> EC2 instance"
ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
     --key-name $NAME-key --security-group-ids $SG --subnet-id $SUBNET \
     --user-data "#!/bin/bash
dnf install -y nginx
systemctl enable --now nginx
echo '<h1>Provisioned entirely by AWS CLI</h1><p>$OWNER</p>' \
  > /usr/share/nginx/html/index.html" \
     --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$NAME-web},{Key=Owner,Value=$OWNER},{Key=Project,Value=PBL1}]" \
     --query 'Instances[0].InstanceId' --output text)
save INSTANCE "$ID"

echo ">>> S3 bucket"
BUCKET="$NAME-bucket"
aws s3 mb s3://$BUCKET
save BUCKET "$BUCKET"

echo ">>> Waiting for the instance to reach running state"
aws ec2 wait instance-running --instance-ids $ID
IP=$(aws ec2 describe-instances --instance-ids $ID \
     --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
save IP "$IP"

echo ""
echo "=============================================="
echo " PROVISIONING COMPLETE"
echo " Environment : $NAME"
echo " VPC         : $VPC"
echo " Subnet      : $SUBNET"
echo " Gateway     : $IGW"
echo " Route table : $RT"
echo " Sec. group  : $SG"
echo " Instance    : $ID"
echo " Bucket      : $BUCKET"
echo " URL         : http://$IP"
echo " State file  : $STATE"
echo "=============================================="
echo "Allow 60-90 seconds for nginx to finish installing."
SCRIPT

chmod +x provision.sh
```

## 1.3 Run it

```bash
./provision.sh
```

Takes about 90 seconds to the summary block, then another 60–90 for nginx.

> **SCREENSHOT 1** — the terminal showing the full run ending in the PROVISIONING COMPLETE block.

## 1.4 Capture the network

VPC console → **Your VPCs** → select `cli-lab-<timestamp>-vpc` → **Resource map** tab.

> **SCREENSHOT 2** — the resource map showing VPC, subnet, route table and gateway.

## 1.5 Capture the site

Open the URL printed by the script.

```bash
curl -I http://<IP>          # confirm 200 before opening the browser
```

> **SCREENSHOT 3** — browser showing "Provisioned entirely by AWS CLI" with the URL bar visible.

---

# Step 2 — VLE 1: Backup evidence (~2 min, while VLE 3 boots)

Both are pure capture. Nothing to build.

## 2.1 S3 backup folders

S3 → `pbl1-lab-ra2311003011574` → click into `site-backup/`.

Cron has been running every 5 minutes since ~07:20 UTC, so expect a long list of `2026-09-06-XXXX` folders.

> **SCREENSHOT 4** — the folder listing.

Optional CLI equivalent, from the instance:

```bash
aws s3 ls s3://pbl1-lab-ra2311003011574/site-backup/
cat /var/log/backup.log | tail -20
```

## 2.2 Custom metric

CloudWatch → **Metrics** → All metrics → scroll to **Custom namespaces** → `PBL1/Backup` → **Metric name** → tick `BackupSuccess`.

Set the range to **3h** and the statistic to **Sum** so the repeated executions are visible as a series rather than a flat line at 1.

> **SCREENSHOT 5** — the BackupSuccess graph.

If `PBL1/Backup` is missing, the `put-metric-data` call is failing. Check:

```bash
tail -20 /var/log/backup.log
aws cloudwatch put-metric-data --namespace PBL1/Backup \
  --metric-name BackupSuccess --value 1
```

An AccessDenied means `EC2-Lab-Role` needs `CloudWatchAgentServerPolicy` attached.

---

# Step 3 — VLE 2: Metric filter and alarm (~15 min)

This is the only remaining piece that needs building.

## 3.1 Confirm logs are arriving

On the instance:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

If it isn't running, re-apply the config:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

## 3.2 Generate fresh 404 traffic

```bash
for i in {1..40}; do
  curl -s localhost/ >/dev/null
  curl -s localhost/missing-$i >/dev/null
done
```

Wait about 60 seconds for the agent to ship the batch.

## 3.3 Create the metric filter

CloudWatch → **Log groups** → `/pbl1/nginx/access` → **Metric filters** tab → **Create metric filter**.

| Field | Value |
|---|---|
| Filter pattern | `[ip, id, user, timestamp, request, status=404, size]` |
| Filter name | `nginx-404-filter` |
| Metric namespace | `PBL1/Logs` |
| Metric name | `Nginx404Count` |
| Metric value | `1` |
| Default value | `0` |

Use **Test pattern** against the log stream before saving — it shows which lines match, which proves the pattern works.

> **SCREENSHOT 6** — the metric filter configuration page, ideally with test results showing matched lines.

## 3.4 Create the alarm

From the metric filter page → **Create alarm**.

| Field | Value |
|---|---|
| Statistic | `Sum` |
| Period | 1 minute |
| Condition | Greater `> 5` |
| Datapoints to alarm | 1 out of 1 |
| Notification | SNS topic `lab-ra2311003011574-alerts` |
| Alarm name | `lab-ra2311003011574-404-high` |

Re-run the curl loop from 3.2 to push it into ALARM.

> **SCREENSHOT 7** — the alarm, ideally in the In alarm state.

## 3.5 Logs Insights

CloudWatch → **Logs Insights** → select log group `/pbl1/nginx/access` → set range to 1h → paste:

```
fields @timestamp, @message
| filter @message like /404/
| sort @timestamp desc
| limit 20
```

Click **Run query**.

> **SCREENSHOT 8** — the query and its results.

A second query worth running, since your logs contain real scanner traffic:

```
fields @timestamp, @message
| filter @message like /wp-config|\.env|\.git/
| sort @timestamp desc
| limit 20
```

That surfaces the automated vulnerability probes hitting your public IP — a strong point to raise in the viva.

---

# Step 4 — Security cleanup (do before the demo)

## 4.1 Delete the exposed access key

IAM → Users → `lab-user` → Security credentials → find `AKIA4WNTZ2RN4YOZCOFW` → Deactivate → Delete.

Create a fresh one if you still need CLI access.

## 4.2 Restrict SSH

EC2 → Security Groups → `launch-wizard-1` (`sg-00096e674d7443619`) → Edit inbound rules → change the port 22 source from `0.0.0.0/0` to **My IP**.

Your VLE 2 error log already shows external scanners probing `/.env.local` and `/wp-config.php.bak`, so this is a fix you have evidence for.

## 4.3 Remove the failed-run orphans

The first `provision.sh` attempt created resources before it died on the SSM error:

```bash
aws ec2 delete-security-group --group-id sg-03dbac3415fad0d52
aws ec2 delete-key-pair --key-name cli-lab-key
```

Then find the orphaned VPC (CIDR `10.0.0.0/16`, no Name tag) and delete its subnet, route table and gateway before the VPC itself. Or use the console's **Delete VPC** action, which handles the dependency order for you.

---

# Step 5 — Teardown (after grading)

## 5.1 Script-created environments

```bash
cat > teardown.sh <<'SCRIPT'
#!/bin/bash
# Usage: ./teardown.sh cli-lab-1757148000.state
set -uo pipefail

STATE=${1:?Usage: ./teardown.sh <state-file>}
source "$STATE"
export AWS_DEFAULT_REGION=$REGION

echo ">>> Tearing down $NAME"

echo "    terminating instance $INSTANCE"
aws ec2 terminate-instances --instance-ids $INSTANCE >/dev/null
aws ec2 wait instance-terminated --instance-ids $INSTANCE

echo "    emptying and removing bucket $BUCKET"
aws s3 rb s3://$BUCKET --force

echo "    deleting key pair $KEY"
aws ec2 delete-key-pair --key-name $KEY
rm -f ${KEY}.pem

echo "    deleting security group $SG"
aws ec2 delete-security-group --group-id $SG

echo "    deleting subnet $SUBNET"
aws ec2 delete-subnet --subnet-id $SUBNET

echo "    deleting route table $RT"
aws ec2 delete-route-table --route-table-id $RT

echo "    detaching and deleting internet gateway $IGW"
aws ec2 detach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
aws ec2 delete-internet-gateway --internet-gateway-id $IGW

echo "    deleting VPC $VPC"
aws ec2 delete-vpc --vpc-id $VPC

mv "$STATE" "${STATE}.destroyed"
echo ">>> Teardown complete"
SCRIPT

chmod +x teardown.sh
./teardown.sh cli-lab-<timestamp>.state
```

## 5.2 Console-created resources

These bill continuously and are not covered by the teardown script:

| Resource | Why it matters |
|---|---|
| Instance `i-0016a9e06a7e80388` | **t3.medium**, not free tier — roughly ₹3.5/hour |
| EBS volume `vol-...` at `/data` | **100 GiB**, well over the 30 GiB free tier — roughly ₹700/month |
| EBS snapshot from Practice 6 | Charged per GB-month |
| Elastic IP `65.2.47.225` | Billed whenever allocated but not attached to a running instance |
| Bucket `pbl1-lab-ra2311003011574` | Delete all object versions first, or the bucket won't delete |
| CloudWatch alarms and dashboard | Small but nonzero |
| Log groups `/pbl1/nginx/*` | Ingestion and storage charges |
| SNS topic and subscription | Negligible, but tidy up |

Order: terminate the instance first (this releases the root volume), then delete the data volume, release the EIP, empty and delete the bucket, then the CloudWatch and SNS resources.

```bash
aws ec2 terminate-instances --instance-ids i-0016a9e06a7e80388
aws ec2 release-address --allocation-id eipalloc-0bcd76d51590aafa9
aws s3 rm s3://pbl1-lab-ra2311003011574 --recursive
aws s3api delete-objects --bucket pbl1-lab-ra2311003011574 \
  --delete "$(aws s3api list-object-versions \
    --bucket pbl1-lab-ra2311003011574 \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
    --output json)"
aws s3 rb s3://pbl1-lab-ra2311003011574
```

Set a billing alarm afterwards if you plan to keep using the account.

---

# Screenshot checklist

| # | What | Where |
|---|---|---|
| 1 | provision.sh terminal output | Mac terminal |
| 2 | VPC resource map | VPC → Your VPCs → Resource map |
| 3 | CLI-provisioned site in browser | http://\<new IP\> |
| 4 | S3 site-backup folders | S3 → bucket → site-backup/ |
| 5 | PBL1/Backup custom metric | CloudWatch → Metrics → Custom namespaces |
| 6 | Metric filter config | CloudWatch → Log groups → Metric filters |
| 7 | Nginx404Count alarm | CloudWatch → Alarms |
| 8 | Logs Insights results | CloudWatch → Logs Insights |

Optional: the SNS email for Practice 4, if it arrived.
