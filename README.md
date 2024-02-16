import boto3

# Initialize Glue client
glue_client = boto3.client('glue')

# Specify the name of the database you want to delete
database_name = 'your_database_name'

try:
    # Delete the specified database
    response = glue_client.delete_database(
        Name=database_name
    )
    print(f"Database '{database_name}' deleted successfully.")
except glue_client.exceptions.EntityNotFoundException:
    print(f"Database '{database_name}' not found.")
except Exception as e:
    print(f"An error occurred: {str(e)}")
