# Deploying CloudForms On OpenShift

This article assumes you have a POC OpenShift cluster that has NFS for persistent storage.
You are logged into OpenShift as administrator 

**Switch over to `management-infra` project and run the following commands
```
oc project management-infra
oc adm policy add-scc-to-user anyuid system:serviceaccount:management-infra:cfme-anyuid
oc describe scc anyuid | grep Users
oc adm policy add-scc-to-user privileged system:serviceaccount:management-infra:default
oc describe scc privileged | grep Users
```
*Not sure if `cfme-anyuid` is used as I could not find that service account later.  

**Download the three PV templates from here** 
[https://github.com/openshift/openshift-ansible/tree/master/roles/openshift_examples/files/examples/v1.5/cfme-templates](https://github.com/openshift/openshift-ansible/tree/master/roles/openshift_examples/files/examples/v1.5/cfme-templates)
```
cfme-pv-db-example.yaml
cfme-pv-region-example.yaml
cfme-pv-server-example.yaml
```
Edit the three yaml files to add your own NFS server.

**Create NFS exports on your local NFS Server**

```
mkdir /exports/cfme-pv01
mkdir /exports/cfme-pv02
mkdir /exports/cfme-pv03
chown nfsnobody:nfsnobody /exports/cfme-pv01
chown nfsnobody:nfsnobody /exports/cfme-pv02
chown nfsnobody:nfsnobody /exports/cfme-pv03
chmod 777 /exports/cfme-pv01
chmod 777 /exports/cfme-pv02
chmod 777 /exports/cfme-pv03
ls -l /exports
vi /etc/exports.d/pv.exports ==> add entries for /exports/cfme-pv01, 02 and 03 in this file 
exportfs -ra
cat /var/lib/nfs/etab
systemctl restart nfs
```
Now your NFS server should have the exports ready

**Create Persistent Volumes**
```
oc create -f cfme-pv-db-example.yaml 
oc create -f cfme-pv-server-example.yaml
oc create -f cfme-pv-region-example.yaml
```

**Create CloudForms template**
```
oc create -f https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v1.5/cfme-templates/cfme-template.yaml
```

**Deploy CloudForms**
```
oc new-app --template=cloudforms -p APPLICATION_DOMAIN=cloudforms.apps.devday.ocpcloud.com
```
After a little while cloudforms-0 pod should be running

**Turn off auto triggers**
```
oc set triggers dc --manual -l app=cloudforms
oc set triggers dc --from-config --auto -l app=cloudforms
```

**Find the cloudforms URL**
```
oc get route 
```

**Access Cloudforms from the browser**

default credentials `admin/smartvm`

**Update password** to change default password 

**Configuring OpenShift inside CloudForms**

Left Menu `Containers` -> `Providers`

Top Menu `Configuration`-> `New Provider`

**Default tab:**

Security Protocol: `SSL without validation`  	

Hostname:  `your master hostname without prefixing with protocol` (eg: master.devday.ocpcloud.com no https)	

API Port: `443`or `8443`	

Token: 	

Use the following command to get the token	
```
	oc sa get-token management-admin -n management-infra
```		

You must use management-admin token. Otherwise you will have issues with hawkular	

**Hawkular tab:**

Security Protocol: `SSL without validation`	

Hostname: `your hawkular metrics url without https and without /hawkular/metrics`  (eg: hawkular-metrics.apps.devday.ocpcloud.com)	

API Port: `443`	

Under the `Server` tab, in the `Server Control` section, under `Server Roles`	

Turn on the following 	
`Capacity & Utilization Coordinator`	
`Capacity & Utilization Data Collector`	
`Capacity & Utilization Data Processor`	
`Capacity and Utilization` 	
