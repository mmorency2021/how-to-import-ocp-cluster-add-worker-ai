# how-to-import-ocp-cluster-add-worker-ai
The idea is to show how to import an OpenShift Running Cluster to new Assisted-Installer then add a worker node to this imported cluster
## Purpose
The  of this repo is to show how to import an Openshift cluster to Offline/Standard-alone Assisted-Installer using REST API (curl) to AI.
Once OCP Cluster is successfully imported to AI, then prepare and add a new worker node to it using aicli.

When the node boots with that ISO, the node automatically reaches out to the existing cluster, the result it causes two CertificateSigningRequests (CSRs) to be sent from the new node to the existing cluster. A CSR is simply a request to obtain the client certificates for the (existing) cluster. You'll need to explicitly approve these requests. Once approved, the existing cluster provides the client certificates to the new node, and the new node is allowed to join the existing cluster. From AI GUI status, it only shows new worker host but not the imported cluster, and green checked wont be possible at current AI version (installed status). With oc cmd all old and new hosts are visible as normally.
  
**Note**: using aicli to prepare and set static IP for new worker node


## Pre-requites
- latest aicli tool  
  https://aicli.readthedocs.io/en/latest/
```bash
python way:
sudo pip3 install -U aicli
sudo pip3 install -U assisted-service-client

Container's way:
alias aicli='sudo podman run --rm -e AI_URL=${AI_URL_EP} -e AI_OFFLINETOKEN=${TOKEN} -v ${HOME}/.ssh/:/root/.ssh/:Z -v $HOME/noknom-aicli/:/root/.aicli:Z -v ${HOME}/noknom-aicli/:/workdir:Z quay.io/karmab/aicli:latest'
```
- An Offline Assisted-Installer
- An Existed/Running OCP Cluster(SNO or Compact Cluster)
- An Offline Token that can get from here  
  https://console.redhat.com/openshift/token/show#
- A Public SSH-Key for core user ssh
- A Pull Secret for Image  
  https://console.redhat.com/openshift/install/pull-secret
  
## Prepare to import Openshift Cluster
### Prepare the deployment by setting the environment variables
#### Setup aicli alias 
```
alias aicli="aicli --offlinetoken $OFFLINE_TOKEN -U $API_URL"
```
#### An API URL to Offline Assisted-Installer
```
export API_URL=192.168.24.80:8080
```
#### Image Pull Secret
```
export PULL_SECRET=$(sed '/^[[:space:]]*$/d' openshift_pull.json| jq -R .)
```
#### Offline Token
```
export OFFLINE_TOKEN='xxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```
#### JWT Token 
```
export TOKEN=$(curl --silent --data-urlencode "grant_type=refresh_token" --data-urlencode "client_id=cloud-services" --data-urlencode "refresh_token=${OFFLINE_TOKEN}" https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token | jq -r .access_token)
```
#### Set Cluster Name, Base-domain and Existed OCP Cluster-ID
```
export CLUSTER_NAME='noknom-aicli'
export CLUSTER_DOMAIN='hubcluster-1.lab.eng.cert.redhat.com'
export OS_CLUSTER_ID=$(oc get clusterversion -o jsonpath='{.items[].spec.clusterID}{"\n"}')
```
### Start Import Openshift Cluster
```bash
curl -X POST  "$API_URL/api/assisted-install/v2/clusters/import" -H "accept: application/json" -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -d "{\"name\":\"$CLUSTER_NAME\",\"openshift_cluster_id\":\"$OS_CLUSTER_ID\",\"api_vip_dnsname\":\"api.$CLUSTER_NAME.$CLUSTER_DOMAIN\"}"
```
### Prepare to create InfraENV CFG
#### Set Environment for New Cluster-ID
```
NEW_CLUSTER_ID=$(aicli list cluster|grep $CLUSTER_NAME |awk '{print $4}')
```
#### Setup/Prepare InfraENV Json for new worker host
```bash
cat << EOF > ./infra-envs-addhost.json
{
  "name": "noknom-aicli",
  "ssh_authorized_key": "$(cat /root/noknom-aicli/.aicli/jumphost-ssh)",
  "pull_secret": $PULL_SECRET,
  "cluster_id": "$NEW_CLUSTER_ID",
  "openshift_version": "4.10",
  "image_type": "minimal-iso",
  "base_dns_domain": "$CLUSTER_DOMAIN"
}
EOF
```
#### Start Create InfraEnv
```
curl -X POST "$API_URL/api/assisted-install/v2/infra-envs" -H "accept: application/json" -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -d @infra-envs-addhost.json
```
