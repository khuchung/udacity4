### Deploy to VMSS

az vmss list-instance-connection-info --resource-group udacity-c4-exercise --name udacity-vmss

<!-- az vmss list-instance-connection-info --resource-group udacity-c4-exercise-autoscale --name udacity-vmss  -->

ssh -p 50003 udacityadmin@20.171.75.121

git clone https://github.com/khuchung/udacity4.git

cd udacity4

git checkout Deploy_to_VMSS

sudo apt update


sudo apt install python3.7
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 2
sudo apt-get install python-pip
sudo apt-get install python3-distutils
sudo apt-get install python3-apt
wget https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py

# sudo -H pip3 install --upgrade pip

wget https://download.redis.io/releases/redis-6.2.4.tar.gz
tar xzf redis-6.2.4.tar.gz
cd redis-6.2.4
make
redis-cli ping

cd ..
pip install -r requirements.txt # python3 -m pip install -r requirements.txt

cd azure-vote/
python3 main.py

### AKS
az aks get-credentials --resource-group udacity-c4-exercise --name udacity-4-cluster --verbose

kubectl apply -f azure-vote-all-in-one-redis.yaml
kubectl set image deployment azure-vote-front azure-vote-front=udacity4952022.azurecr.io/azure-vote-front:v1

kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=1 --max=10

kubectl run -i --tty load-generator --image=busybox /bin/sh

while true; do wget -q -O- http://azure-vote-front.default.svc.cluster.local; done


kubectl get hpa


while true; do wget -q -O- http://azure-vote-front.default.svc.cluster.local; done


### Runbook
```
#!/usr/bin/env python3
import os
from azure.mgmt.compute import ComputeManagementClient
import azure.mgmt.resource
import automationassets

def get_automation_runas_credential(runas_connection):
    from OpenSSL import crypto
    import binascii
    from msrestazure import azure_active_directory
    import adal

    # Get the Azure Automation RunAs service principal certificate
    cert = automationassets.get_automation_certificate("AzureRunAsCertificate")
    pks12_cert = crypto.load_pkcs12(cert)
    pem_pkey = crypto.dump_privatekey(crypto.FILETYPE_PEM,pks12_cert.get_privatekey())

    # Get run as connection information for the Azure Automation service principal
    # NOTE: no need to update your ID here
    application_id = runas_connection["ApplicationId"]
    thumbprint = runas_connection["CertificateThumbprint"]
    tenant_id = runas_connection["TenantId"]

    # Authenticate with service principal certificate
    resource ="https://management.core.windows.net/"
    authority_url = ("https://login.microsoftonline.com/"+tenant_id)
    context = adal.AuthenticationContext(authority_url)
    return azure_active_directory.AdalAuthentication(
    lambda: context.acquire_token_with_client_certificate(
            resource,
            application_id,
            pem_pkey,
            thumbprint)
    )

# Authenticate to Azure using the Azure Automation RunAs service principal
runas_connection = automationassets.get_automation_connection("AzureRunAsConnection")
azure_credential = get_automation_runas_credential(runas_connection)

# Initialize the compute management client with the Run As credential and specify the subscription to work against.
# NOTE: no need to update your ID here
compute_client = ComputeManagementClient(
    azure_credential,
    str(runas_connection["SubscriptionId"])
)

myvmss = compute_client.virtual_machine_scale_sets.get("udacity-c4-exercise-autoscale", "udacity-vmss")
print("Initial myvmss.sku.capacity = ",myvmss.sku.capacity)
myvmss.sku.capacity = 4
print("New myvmss.sku.capacity = ",myvmss.sku.capacity)
# Increase the VMSS SKU Capacity
async_vmss = compute_client.virtual_machine_scale_sets.create_or_update("udacity-c4-exercise-autoscale", "udacity-vmss", myvmss)
async_vmss.wait()
```
