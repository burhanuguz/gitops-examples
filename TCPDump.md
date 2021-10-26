# Openshift TCP Dump Automation with Ansible

## **Prerequisites**
- Python 3 and Ansible(obviously)
- **openshift pyyaml kubernetes kubernetes-validate** python 3 modules to execute tasks in playbook

```bash
# Install ansible and python 3.6 first
yum install ansible python36 jq -y -q
# Then install modules to execute tasks in playbook
python3 -m pip install openshift pyyaml kubernetes kubernetes-validate
```

## **Requirements on Openshift side**
This automation needs privileged SCC and custom rules to take dump.

- First, create the cluster role. It is a basic cluster role to get info from deployment and pods, and additionaly for creating pods to take TCP dumps from nodes.

```bash
cat << EOF | oc create -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ansible-debugger-cr
rules:
  - verbs:
      - get
      - list
      - create
      - delete
    apiGroups:
      - ''
    resources:
      - pods
  - verbs:
      - get
    apiGroups:
      - apps
    resources:
      - deployments
EOF
```

- Now create the serviceaccount. Then add the privileged SCC and bind the role just created above.

```bash
# Create service account
oc create serviceaccounts -n openshift ansible-debugger-sa
# Add privileged SCC to service account 
oc adm policy add-scc-to-user          privileged          -n openshift -z ansible-debugger-sa
# Bind the role to service account
oc adm policy add-cluster-role-to-user ansible-debugger-cr -n openshift -z ansible-debugger-sa
```

- Get the **ansible-debugger-sa** service account token to use it in playbook.

```bash
oc get secret -n openshift \
$(oc get serviceaccount -n openshift ansible-debugger-sa -ojsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) \
-o jsonpath='{.data.token}' | base64 --decode
```

- After these steps, you should add your apiserver and service account token information to **inventory/inventory.yaml** in this repo.
  - To make changes easily use the commands below.

```bash
# Get Token
TOKEN=$(oc get secret -n openshift \
$(oc get serviceaccount -n openshift ansible-debugger-sa -ojsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) \
-o jsonpath='{.data.token}' | base64 --decode)

# Define API server
APISERVER="https://openshift-api:6443"

# Make changes at inventor.yaml with sed substitution.
sed -i "s|\(.*url: '\)\(.*\)\('\)|\1${APISERVER}\3|"  inventory/inventory.yaml
sed -i "s|\(.*sa_token: '\)\(.*\)\('\)|\1${TOKEN}\3|" inventory/inventory.yaml
```

- You can try it with a deployment name **example** on **openshift** namespace like below. The **start time** command below is useful start the taking dump operation 2 minutes later. You need to specify exact minute and you can not define it lower than the minute at current clock . If the clock is 01:23, you should specify **start_time** >~ **23** to give time cluster to pull images for taking dump.

- One little information also for **tower_job_id**. If you use **Ansible Tower**, you do not have to specify the value, it comes from Ansible Tower itself.

```bash
ansible-playbook oc_take_dump_tasks.yml -i inventory/inventory.yaml \
-e start_time=$(( ($(date +%M) % 60) + 2)) \
-e resource_type=deployment \
-e deployment_name=example \
-e namespace=default \
-e tower_job_id=$RANDOM 
```
