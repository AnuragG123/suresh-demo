import boto3
import os

def lambda_handler(event, context):
    source_snapshot_id = event.get('source_snapshot_id')
    target_snapshot_id = event.get('target_snapshot_id', f"copy-of-{source_snapshot_id}")
    source_region = os.environ['SOURCE_REGION']
    target_region = os.environ['TARGET_REGION']
    
    rds_client = boto3.client('rds', region_name=source_region)
    
    try:
        rds_client.copy_db_snapshot(
            SourceDBSnapshotIdentifier=source_snapshot_id,
            TargetDBSnapshotIdentifier=target_snapshot_id,
            SourceRegion=source_region
        )
        return {
            'statusCode': 200,
            'body': f"Snapshot {source_snapshot_id} copied to {target_snapshot_id} in {target_region}"
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': str(e)
        }
