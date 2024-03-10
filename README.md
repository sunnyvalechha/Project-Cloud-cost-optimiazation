# Project-Cloud-cost-optimiazation

Project Link : https://github.com/iam-veeramalla/aws-devops-zero-to-hero/tree/main/day-18

In this example, we'll create a Lambda function that identifies EBS snapshots that are no longer associated with any active EC2 instance and deletes them to save on storage costs.

**Description:**

The Lambda function fetches all EBS snapshots owned by the same account ('self') and also retrieves a list of active EC2 instances (running and stopped). For each snapshot, it checks if the associated volume (if exists) is not associated with any active instance. If it finds a stale snapshot, it deletes it, effectively optimizing storage costs.

**Code**

    import boto3

    def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")

**Practical**

Note: What admin has done, he has configured a script or lambda function that daily this volume of the instance should be backed up (snapshot creation ) as this EC2 instance has data coming daily so backup is required.
So the count of snapshot is increasing or in another scenerio the admin has deleted the instance and volume manually but forget to delete the snapshot.
To identify such cases we can use Lambda function.
This Lambda function does not delete any snapshot which is associated with any volume.

    1. Launch an EC2 instance with default configuration. (1 default volume attached)
    2. Check volume in Instance details and re-name the volume in Volume section
    3. Create Snapshot of Volume not Instance.
    4. Go to Lambda > Create function > Author from Scratch
    5. Name = Cost-optimization-EBS-snapshot
    6. Runtime = Python3.12
    7. Create function
    8. Copy the code and paste into Code Source section
    9. Save = clt+s
    10. Deploy and click on Test
    11. Eventname = testevent, Save

Note: This will fail couple of time due to Permission Issues.

What we are doing here through Lambda, we are describing snapshot and filter out the stale snapshots.

Here, task time out is 3.02 seconds we have to increase the Execution timout value to 10 seconds

    12. Configuration > Edit > Increase seconds from 3 to 10 > Save

We also have to create a role to attached to Lambda function.

![image](https://github.com/sunnyvalechha/Project-Cloud-cost-optimiazation/assets/59471885/a116b2dd-a585-410b-b58a-75abe70a46e5)

![image](https://github.com/sunnyvalechha/Project-Cloud-cost-optimiazation/assets/59471885/fdd9fff9-57de-4246-88d6-ceabff5078df)

    13. Click on the role > open in new tab > Add Permissions

If we search for snapshot, it is not there because snapshot is not part of the policy, required permissions (describe snapshots, delete snapshots)

![image](https://github.com/sunnyvalechha/Project-Cloud-cost-optimiazation/assets/59471885/12432899-0f9b-4bdd-b51e-f155f4854232)

    14. Create Policy
    15. Chosse Service = EC2 > Filter Actions > Snapshots > Describe / Delete, Resources = ALL > Create

![image](https://github.com/sunnyvalechha/Project-Cloud-cost-optimiazation/assets/59471885/767240ee-398b-49c3-abc3-20ea59325a5d)

    16. Go to lambda function and attached newly created policy
    17. Go to code and run the function again, and it failed again

![image](https://github.com/sunnyvalechha/Project-Cloud-cost-optimiazation/assets/59471885/199deb1e-fc67-4040-aa24-8ab870d8e79e)

    18 Edit the policy and attach describevolume and describeinstances
    19. Run the Policy again

Note: This time the function succeed but does not delete the snapshot because this is attached to an Volume.

    20. Now delete the Instance, the instance will delete the volume as well
    21. Run the code again

![image](https://github.com/sunnyvalechha/Project-Cloud-cost-optimiazation/assets/59471885/6fe70174-ad4f-4e04-89d5-582b9e01a2fb)







  


    
