# s3-to-ssm

import json
import boto3
import re

sg = "sg-3e89f577"
instance_list = ['']
Uniq_IPaddr_temp = ['']

def get_ec2_instances_from_sg(sg_group):

    ec2 = boto3.client('ec2')

    response = ec2.describe_instances(
        Filters=[
            {
                'Name': 'instance.group-id',
                'Values': [
                    sg_group,
                ]
            },
        ],
    )
    instancelist = []
    for reservation in (response["Reservations"]):
        for instance in reservation["Instances"]:
            instancelist.append(instance["InstanceId"])
#    return instancelist
    return instancelist
    print (instancelist)

def lambda_handler(event, context):
    # TODO implement
    S3 = boto3.client("s3")
    if (event!=None):
        print("Event:",event)
        file_obj = event ["Records"][0]
        filename = str(file_obj['s3']['object']['key'])
        print("Filename:",filename)
        fileObj = S3.get_object(Bucket = "xuwenbo-new", Key = filename)
        file_content = fileObj["Body"].read().decode('utf-8')
        print(file_content)
        IPaddr = re.findall(r'[0-9]*[.][0-9]*[.][0-9]*[.][0-9]*',file_content,re.M)
        Uniq_IPaddr_temp = list(set(IPaddr))
     #   Uniq_IPaddr = Uniq_IPaddr_temp.sort(key=IPaddr.index)
        print (Uniq_IPaddr_temp)
     #   instance_id = list(get_ec2_instances_from_sg(sg))
        instance_list = get_ec2_instances_from_sg(sg)
        print instance_list
        ssm_handler(Uniq_IPaddr_temp,instance_list)
        
        

def ssm_handler(ipaddr, instances):
    client = boto3.client('ssm')
    totalcmd = ""
#  a = ['i-0ae3555ba2ab9607b', 'i-0f1fa446a91f78332', 'i-0c58cbf5c4a2302cf', 'i-0e86c859aa291de75'] 
#    ipaddr = ['53.54.1.1','53.54.2.2']
    for i in ipaddr:
        print i
        singlecmd = "echo" + ' ' + i + ">>" + "/tmp/1.txt"
        totalcmd += singlecmd + "\n"
    print totalcmd
    for instanceid in instances:
        #print instanceid
        #print type(instanceid)
        response = client.send_command(
            InstanceIds=[
                instanceid,
            ],
            DocumentName='AWS-RunShellScript',
            DocumentVersion='$LATEST',
            DocumentHash='99749de5e62f71e5ebe9a55c2321e2c394796afe7208cff048696541e6f6771e',
            DocumentHashType='Sha256',
            TimeoutSeconds=123,
            Comment='update WAF rules',
            Parameters={
                'commands': [
                    totalcmd,
                ]
            },
            OutputS3BucketName='nancyxue',
            OutputS3KeyPrefix='ssmoutput',
            MaxErrors='10',
        #    ServiceRoleArn='arn:aws:iam::351955137083:role/shiwang-S3-lambda',
        )

        
    
    

