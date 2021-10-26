# Openshift project creations

## Python PIP dependencies: 

 - openshift
 - pyyaml
 - kubernetes
 - kubernetes-validate

## installation for requirements
```bash
yum install python36  
python3 -m pip openshift pyyaml kubernetes kubernetes-validate
```
## For installing on Ansible Tower that works with Python 2, you should create virtual env

```bash
python3 -m venv /var/lib/awx/venv/ocp_project
source /var/lib/awx/venv/ocp_project/bin/activate
umask 0022
python3 -m pip install pip --upgrade
python3 -m pip install ansible openshift pyyaml kubernetes kubernetes-validate psutil
deactivate
```

After creating virtual env you should set it on Ansible Tower also.

Then create RBAC and permisions like below.
```bash
cat << EOF | oc create -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ansible-create-project-cr
rules:
  - verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
    apiGroups:
      - ''
    resources:
      - namespaces
  - verbs:
      - create
    apiGroups:
      - project.openshift.io
    resources:
      - projectrequests
  - verbs:
      - create
      - get
      - list
    apiGroups:
      - ''
    resources:
      - limitranges
      - resourcequotas
  - verbs:
      - update
      - patch
    apiGroups:
      - network.openshift.io
    resources:
      - netnamespace
  - verbs:
      - update
      - patch
    apiGroups:
      - maistra.io
    resources:
      - servicemeshmemberrolls
EOF
```

```bash
# Either create a service account and give permission to it
oc create serviceaccount ansible-create-project-sa -n openshift
oc adm policy add-cluster-role-to-user ansible-create-project-cr -z ansible-create-project-sa -n openshift

oc get secret -n openshift \
$(oc get serviceaccount -n openshift ansible-create-project-sa -ojsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) \
-o jsonpath='{.data.token}' | base64 --decode

# Or directly give permission to Openshift user (e.g LDAP)
oc adm policy add-cluster-role-to-user ansible-create-project-cr "OPENSHIFTUSER"
```

Finally, add the User info or service account token to Ansible Role to use.
