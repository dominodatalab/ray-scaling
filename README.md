# Scaling Ray Clusters for IRSA

The attached notebook has an example of using IRSA with Ray clusters that need to start with 100's of workers which need to be IRSA enabled. Default IRSA behaviour is the inject a side-car which calls the `irsa-svc` to configure the `AWS_CONFIG_FILE`. However when that call is made simultaneously from 100's of containers, handling failures becomes important. Also because the `irsa-svc` updates the AWS Trust Policies on roles to enable IRSA for a specific Service Account, this leads to scaling challenges.

The notebook demonstrates a simple workaround to the scaling challenge. 

1. We do not inject the side-car container. The side-car is convinience container to configure the user workload without any action on part of the user. The updated [mutation](https://github.com/dominodatalab/domino-field-solutions-installations/blob/main/irsa/helm/irsa/templates/mutation.yaml) removes the side-car injection. The changes to the `values.yaml` is to add
the following [section](https://github.com/dominodatalab/domino-field-solutions-installations/blob/main/irsa/values.yaml)
```
irsa_client_sidecar:
  enabled: false
```

2. Next the user now assumed the responsibility for fetching and saving the `AWS_CONFIG_FILE`. Note that the containers are fully enabled with all the necessary mounts of the `AWS_WEB_IDENTITY_TOKEN_FILE` and other environment variables. The only missing part is the actual file `AWS_CONFIG_FILE`. The user makes the following call to create the file
```python
#Emulate Side-Car
import requests
import os


#Get Access Token
access_token_endpoint='http://localhost:8899/access-token'
resp = requests.get(access_token_endpoint)
token = resp.text

# Get AWS Config File
os.environ['SSL_CERT_DIR']='/etc/ssl/certs/irsa'
headers = {
             "Content-Type": "application/json",
             "Authorization": "Bearer " + token,
        }
endpoint='https://irsa-svc.domino-field/map_iam_roles_to_pod_sa'
print(f"Domino Run Id{os.environ['DOMINO_RUN_ID']}")
data = {"run_id": os.environ['DOMINO_RUN_ID'], "irsa_workload_type":"cluster-edge"} ## It fetches this fom the downward api
resp = requests.post(endpoint,headers=headers,json=data,verify=False)

# Write the AWS Config file contents to the AWS_CONFIG_FILE location
aws_config_file_contents=''
if resp.status_code == 200:
    aws_config_file_contents=resp.text
    print('Write to config file')
    config_file = os.environ["AWS_CONFIG_FILE"]
    with open(config_file, "w") as f:
        f.write(resp.content.decode())

```
3. Next pass the contents of this file as a string to the ray worker and make the following calls inside the ray worker to configure the file
```python
import os
import time
import ray
import mlflow
import boto3
@ray.remote
def example_ray_worker_function(worker_id,aws_config_file_contents):
    config_file = os.environ["AWS_CONFIG_FILE"]
    with open(config_file, "w") as f:
        f.write(aws_config_file_contents)
    session = boto3.Session(profile_name="sw-domino-project3-role")
    region_name=session.region_name
    aws_access_key_id=session.get_credentials().access_key
    aws_secret_access_key=session.get_credentials().secret_key
    aws_session_token= session.get_credentials().token
    #os.environ['AWS_ROLE_ARN']='arn:aws:iam::946429944765:role/sw-domino-project-based-mlflow-6526f64938a634604600664a'
    os.environ['AWS_ACCESS_KEY_ID'] = aws_access_key_id
    os.environ['AWS_SECRET_ACCESS_KEY'] = aws_secret_access_key
    os.environ['AWS_SESSION_TOKEN'] = aws_session_token
    .... #DO YOUR STUFF#
```
That's it. Even if you use thousands of ray workers you will only need to call the backend IRSA-SVC only once. 
