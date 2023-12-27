PLEASE NOTE: This work has been moved into a private repo. If I get that functioning well and have time I intend to update this repo to reflect the progress of the private repo. Admittedly, this was so I could be lazy about privacy concerns while working on it. 

# comfy-diffusion-aws
A fork from mikeage@ 's stable-diffusion-aws, with a goal of running comfyui with the same on-demand spot instance setup. 

For now, the focus will be on getting a functional automatic111 webui that can be called on demand, attach to permanent storage with the correct checkpoints and LORAs, and attach to an existing Elastic Network Adapter at eth0 to integrate into 404.exchange's existing services. 


#### Launch the instance
```bash {name=launch-an-instance}
# Create the spot instance request
# Get the latest Debian 11 image
export AMI_ID=$(aws ec2 describe-images --owners 136693071363 --query "sort_by(Images, &CreationDate)[-1].ImageId" --filters "Name=name,Values=debian-12-amd64-*" | jq -r .)

#Default VPC name
export DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)

#Security Group. I personally pre-made these. 
export SG_ID=$(aws ec2 create-security-group --group-name SSH-Only --description "Allow SSH from anywhere" --vpc-id $DEFAULT_VPC_ID --query 'GroupId' --output text)

#Note that in the spot instance creation below, the userdata file is setup.sh, also in this repo. 

aws ec2 run-instances \
    --no-cli-pager \
    --image-id $AMI_ID \
    --instance-type g4dn.2xlarge \
    --key-name stable-diffusion-aws \
    --security-group-ids $SG_ID \
    --block-device-mappings "DeviceName=/dev/xvda,Ebs={VolumeSize=${EBS_SIZE},VolumeType=gp3}" \
    --user-data file://setup.sh \
    --metadata-options "InstanceMetadataTags=enabled" \
    --tag-specifications "ResourceType=spot-instances-request,Tags=[{Key=creator,Value=stable-diffusion-aws}]" "ResourceType=instance,Tags=[{Key=INSTALL_AUTOMATIC1111,Value=$INSTALL_AUTOMATIC1111},{Key=INSTALL_INVOKEAI,Value=$INSTALL_INVOKEAI},{Key=GUI_TO_START,Value=$GUI_TO_START}]" \
    --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.70,SpotInstanceType=persistent,InstanceInterruptionBehavior=stop}'

```


#### Create an Alarm to stop the instance after 15 minutes of idling (optional)

```bash {name=create-cloudwatch-alarm, promptEnv=false}
export INSTANCE_ID="$(aws ec2 describe-spot-instance-requests --spot-instance-request-ids $SPOT_INSTANCE_REQUEST | jq -r '.SpotInstanceRequests[].InstanceId')"
aws cloudwatch put-metric-alarm \
    --alarm-name stable-diffusion-aws-stop-when-idle \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --statistic Maximum \
    --period 300  \
    --evaluation-periods 3 \
    --threshold 5 \
    --comparison-operator LessThanThreshold \
    --unit Percent \
    --dimensions "Name=InstanceId,Value=$INSTANCE_ID" \
    --alarm-actions arn:aws:automate:$AWS_REGION:ec2:stop

```
