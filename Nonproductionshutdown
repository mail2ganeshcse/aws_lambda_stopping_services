import boto3
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

def send_email(subject, body, to_email, from_email, smtp_server, smtp_port, smtp_user, smtp_password):
    # Create the email
    msg = MIMEMultipart()
    msg['From'] = from_email
    msg['To'] = to_email
    msg['Subject'] = subject

    # Attach the body with the msg instance
    msg.attach(MIMEText(body, 'plain'))

    # Create server object with SSL option
    server = smtplib.SMTP(smtp_server, smtp_port)
    
    # Perform operations via server
    server.starttls()
    server.login(smtp_user, smtp_password)
    text = msg.as_string()
    server.sendmail(from_email, to_email, text)
    server.quit()

def start_rds_cluster(rds_client, cluster_id):
    try:
        response = rds_client.start_db_cluster(DBClusterIdentifier=cluster_id)
        return f"RDS cluster {cluster_id} started successfully."
    except rds_client.exceptions.InvalidDBClusterStateFault as e:
        if "available state but expected it to be one of stopped,inaccessible-encryption-credentials-recoverable" in str(e):
            return f"RDS cluster {cluster_id} is already in the 'available' state."
        else:
            raise e

def format_ec2_response(response):
    instances = response['StartingInstances']
    statuses = []
    for instance in instances:
        instance_id = instance['InstanceId']
        current_state = instance['CurrentState']['Name']
        previous_state = instance['PreviousState']['Name']
        statuses.append(f"Instance {instance_id} changed from {previous_state} to {current_state}.")
    return "\n".join(statuses)

def lambda_handler(event, context):
    # Define regions
    region_1 = 'ap-south-1'
    region_2 = 'ap-south-2'
    
    # EKS Auto Scaling Groups
    UAT1 = 'EKS-ASG-1'
    UAT2 = 'EKS-ASG-2'
    
    # EC2 Instances
    ec2_instance_ids_region_1 = ['instance-id-0']
    ec2_instance_ids_region_2 = [
        'instance-id-1',
        'instance-id-2'
    ]
    
    # RDS Clusters
    rds_cluster_ids = [
        'RDS-cluster-1',
        'RDS-cluster-2'
    ]

    response_message = ""
    subject = "infra service started status"
    
    try:
        # Create clients for Auto Scaling, EC2, and RDS for both regions
        autoscaling_2 = boto3.client('autoscaling', region_name=region_2)
        ec2_1 = boto3.client('ec2', region_name=region_1)
        ec2_2 = boto3.client('ec2', region_name=region_2)
        rds_2 = boto3.client('rds', region_name=region_2)

        # Update capacity settings for UAT1 in region_2
        response_UAT1 = autoscaling_2.update_auto_scaling_group(
            AutoScalingGroupName=UAT1,
            MinSize=1,
            MaxSize=3,
            DesiredCapacity=1
        )
        response_message += f"Updated Auto Scaling Group {UAT1} to MinSize=1, MaxSize=3, DesiredCapacity=1\n"
        
        # Update capacity settings for UAT2 in region_2
        response_UAT2 = autoscaling_2.update_auto_scaling_group(
            AutoScalingGroupName=UAT2,
            MinSize=1,
            MaxSize=3,
            DesiredCapacity=1
        )
        response_message += f"Updated Auto Scaling Group {UAT2} to MinSize=1, MaxSize=3, DesiredCapacity=1\n"

        # Start EC2 instances in region_1
        response_ec2_region_1 = ec2_1.start_instances(
            InstanceIds=ec2_instance_ids_region_1
        )
        response_message += f"Started EC2 instances in {region_1}:\n{format_ec2_response(response_ec2_region_1)}\n"

        # Start EC2 instances in region_2
        response_ec2_region_2 = ec2_2.start_instances(
            InstanceIds=ec2_instance_ids_region_2
        )
        response_message += f"Started EC2 instances in {region_2}:\n{format_ec2_response(response_ec2_region_2)}\n"

        # Start RDS clusters in region_2
        for cluster_id in rds_cluster_ids:
            try:
                response_message += start_rds_cluster(rds_2, cluster_id) + "\n"
            except Exception as e:
                response_message += f"Error starting RDS cluster {cluster_id}: {str(e)}\n"

    except Exception as e:
        response_message += f"An error occurred: {str(e)}\n"
        subject = "GB UAT infra start failed"

    print(response_message)

    # Email details
    to_email = "test@gmail.com"
    from_email = "test@gmail.com,"
    smtp_server = "smtp.google.com"
    smtp_port = 587
    smtp_user = "test"
    smtp_password = "test123"

    # Send email
    send_email(subject, response_message, to_email, from_email, smtp_server, smtp_port, smtp_user, smtp_password)

    return {
        'statusCode': 200,
        'body': {
            'message': 'Process completed with some potential errors. Check email for details.',
            'details': response_message
        }
    }
