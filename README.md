# Workflow Process Service deployment manual installation ✍️<!-- omit in toc -->

For version 23.0.2 iFix 1

Installs WfPS environment.

- [Disclaimer ✋](#disclaimer-)
- [Preparing your cluster](#preparing-your-cluster)
- [Preparing a client to connect to the cluster](#preparing-a-client-to-connect-to-the-cluster)
  - [Creating an install client directly in OCP](#creating-an-install-client-directly-in-ocp)
  - [Command line preparation in install Pod](#command-line-preparation-in-install-pod)
- [Prerequisite software](#prerequisite-software)
  - [OpenLDAP](#openldap)
    - [Access info after deployment](#access-info-after-deployment)
  - [PostgreSQL](#postgresql)
    - [Access info after deployment](#access-info-after-deployment-1)
- [Workflo Process Service Development Environment](#workflo-process-service-development-environment)
  - [Preparing a client to connect to the cluster](#preparing-a-client-to-connect-to-the-cluster-1)
  - [Setting up the cluster by running a script](#setting-up-the-cluster-by-running-a-script)
  - [Prepare property files](#prepare-property-files)
  - [Generate, update and apply SQL and Secret files](#generate-update-and-apply-sql-and-secret-files)
  - [Validate connectivity](#validate-connectivity)
  - [Generating the custom resource with the deployment script](#generating-the-custom-resource-with-the-deployment-script)
  - [Checking and completing your custom resource](#checking-and-completing-your-custom-resource)
  - [Deploying the custom resource you created with the deployment script](#deploying-the-custom-resource-you-created-with-the-deployment-script)
  - [Access info after deployment](#access-info-after-deployment-2)
  - [Completing post-installation tasks](#completing-post-installation-tasks)
  - [Validating your production deployment](#validating-your-production-deployment)
  - [Next steps](#next-steps)
- [Workflow Process Service Test Environment](#workflow-process-service-test-environment)
  - [Preparing a client to connect to the cluster](#preparing-a-client-to-connect-to-the-cluster-2)
  - [Setting up the cluster by running a script](#setting-up-the-cluster-by-running-a-script-1)
  - [Deploy minmal CP4BA](#deploy-minmal-cp4ba)
  - [Add WfPS deplyoment](#add-wfps-deplyoment)
  - [Access info after deployment](#access-info-after-deployment-3)
  - [Completing post-installation tasks](#completing-post-installation-tasks-1)
  - [Next steps](#next-steps-1)
    - [Connect to Authoring](#connect-to-authoring)
- [Contacts](#contacts)
- [Notice](#notice)

## Disclaimer ✋

This is **not** an official IBM documentation.  
Absolutely no warranties, no support, no responsibility for anything.  
Use it on your own risk and always follow the official IBM documentations.  
It is always your responsibility to make sure you are license compliant when using this repository to install IBM Cloud Pak for Business Automation.  
Used OpenLDAP and PostgreSQL cannot run on Restricted OCP from CP4BA.

Please do not hesitate to create an issue here if needed. Your feedback is appreciated.

Not for production use. Suitable for Demo and PoC environments - but with Production deployment.

OpenLDAP is not a supported LDAP provider for CP4BA.

## Preparing your cluster

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-preparing-your-cluster and  
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-getting-access-images-from-public-entitled-registry

- Empty OpenShift cluster of a supported version
- With direct internet connection
- File RWX StorageClass - in this case ocs-storagecluster-cephfs is used, feel free to find and replace
- Block RWO StorageClass - in this case ocs-storagecluster-ceph-rbd is used, feel free to find and replace. File Based RWX is also usable but is not supported.
- Cluster admin user
- IBM entitlement key from https://myibm.ibm.com/products-services/containerlibrary

## Preparing a client to connect to the cluster

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-preparing-client-connect-cluster
and additional utilities for script execution

Requested tooling provided in the install Pod.

- kubectl
- oc
- podman
- IBM Semeru JDK 17
- yq
- jq

### Creating an install client directly in OCP

In you OCP cluster create Project *wfps-install* using OpenShift console.
```yaml
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: wfps-install
  labels:
    app: wfps-install
```

Create PVC where all files used for installation would be kept if you would need to re-instantiate the install Pod.
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: install
  namespace: wfps-install
  labels:
    app: wfps-install
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
```

Assign cluster admin permissions to *wfps-install* default ServiceAccount under which the installation is performed by applying the following yaml.

This requires the logged in OpenShift user to be cluster admin.

The ServiceAccount needs to have cluster admin to be able to create all resources needed to deploy the platform.
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-admin-wfps-install
  labels:
    app: wfps-install
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: "system:serviceaccount:wfps-install:default"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

Create a pod from which the install will be performed with the following definition.  
It is ready when the message *Install pod - Ready* is in its Log.
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: install
  namespace: wfps-install
  labels:
    app: wfps-install
spec:
  containers:
    - name: install
      securityContext:
        privileged: true
      image: ubi9/ubi:9.3
      command: ["/bin/bash"]
      args:
        ["-c","cd /usr;
          yum install podman -y;
          yum install ncurses -y;
          yum install jq -y;
          curl -LO https://github.com/ibmruntimes/semeru17-binaries/releases/download/jdk-17.0.9%2B9_openj9-0.41.0/ibm-semeru-open-jdk_x64_linux_17.0.9_9_openj9-0.41.0.tar.gz;
          tar -xvf ibm-semeru-open-jdk_x64_linux_17.0.9_9_openj9-0.41.0.tar.gz;
          ln -fs /usr/jdk-17.0.9+9/bin/java /usr/bin/java;
          ln -fs /usr/jdk-17.0.9+9/bin/keytool /usr/bin/keytool;
          curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz --output oc.tar;
          tar -xvf oc.tar oc;
          chmod u+x oc;
          ln -fs /usr/oc /usr/bin/oc;
          curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl;
          chmod u+x kubectl;
          ln -fs /usr/kubectl /usr/bin/kubectl;
          curl -LO https://github.com/mikefarah/yq/releases/download/v4.30.5/yq_linux_amd64.tar.gz;
          tar -xzvf yq_linux_amd64.tar.gz;
          ln -fs /usr/yq_linux_amd64 /usr/bin/yq;
          podman -v;
          java -version;
          keytool;
          oc version;
          kubectl version;
          yq --version;
          jq --version;
          while true;
          do echo 'Install pod - Ready - Enter it via Terminal and \"bash\"';
          sleep 300; done"]
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - name: install
          mountPath: /usr/install
  volumes:
    - name: install
      persistentVolumeClaim:
        claimName: install
```

### Command line preparation in install Pod

This needs to be repeated if you don't perform this in one go and you come back to resume.

Open Terminal window of the *install* pod.

Enter bash.
```sh
bash
```

## Prerequisite software

CP4BA needs at least LDAP and Database. In this deployment OpenLDAP and PostgreSQL are used.

You would normally use your own production ready instances.

Please note that OpenLDAP is working but is not officially supported by CP4BA.

### OpenLDAP

Create wfps-openldap Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: wfps-openldap
  labels:
    app: wfps-openldap
" | oc apply -f -
```

Add required privileged permission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openldap-privileged
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
subjects:
  - kind: ServiceAccount
    name: default
    namespace: wfps-openldap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
" | oc apply -f -
```

Create ConfigMap for env
```bash
echo "
kind: ConfigMap
apiVersion: v1
metadata:
  name: openldap-env
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
data:
  BITNAMI_DEBUG: 'true'
  LDAP_ORGANISATION: cp.internal
  LDAP_ROOT: 'dc=cp,dc=internal'
  LDAP_DOMAIN: cp.internal
" | oc apply -f -
```

Create ConfigMap for users and groups
```bash
echo "
kind: ConfigMap
apiVersion: v1
metadata:
  name: openldap-customldif
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
data:
  01-default-users.ldif: |-
    # cp.internal
    dn: dc=cp,dc=internal
    objectClass: top
    objectClass: dcObject
    objectClass: organization
    o: cp.internal
    dc: cp

    # Units
    dn: ou=Users,dc=cp,dc=internal
    objectClass: organizationalUnit
    ou: Users

    dn: ou=Groups,dc=cp,dc=internal
    objectClass: organizationalUnit
    ou: Groups

    # Users
    dn: uid=cpadmin,ou=Users,dc=cp,dc=internal
    objectClass: inetOrgPerson
    objectClass: top
    cn: cpadmin
    sn: cpadmin
    uid: cpadmin
    mail: cpadmin@cp.internal
    userpassword: Password
    employeeType: admin

    dn: uid=cpuser,ou=Users,dc=cp,dc=internal
    objectClass: inetOrgPerson
    objectClass: top
    cn: cpuser
    sn: cpuser
    uid: cpuser
    mail: cpuser@cp.internal
    userpassword: Password

    # Groups
    dn: cn=cpadmins,ou=Groups,dc=cp,dc=internal
    objectClass: groupOfNames
    objectClass: top
    cn: cpadmins
    member: uid=cpadmin,ou=Users,dc=cp,dc=internal

    dn: cn=cpusers,ou=Groups,dc=cp,dc=internal
    objectClass: groupOfNames
    objectClass: top
    cn: cpusers
    member: uid=cpadmin,ou=Users,dc=cp,dc=internal
    member: uid=cpuser,ou=Users,dc=cp,dc=internal
" | oc apply -f -
```

Create Secret for password
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: openldap
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
stringData:
  LDAP_ADMIN_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: openldap-data
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create Deployment
```bash
echo "
kind: Deployment
apiVersion: apps/v1
metadata:
  name: openldap
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wfps-openldap
  template:
    metadata:
      labels:
        app: wfps-openldap
    spec:
      containers:
        - name: openldap
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            tcpSocket:
              port: ldap-port
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 30
          readinessProbe:
            tcpSocket:
              port: ldap-port
            initialDelaySeconds: 60
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
          livenessProbe:
            tcpSocket:
              port: ldap-port
            initialDelaySeconds: 60
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
          terminationMessagePath: /dev/termination-log
          ports:
            - name: ldap-port
              containerPort: 1389
              protocol: TCP
          image: 'bitnami/openldap:2.6.5'
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: data
              mountPath: /bitnami/openldap/
            - name: custom-ldif-files
              mountPath: /ldifs/
          terminationMessagePolicy: File
          envFrom:
            - configMapRef:
                name: openldap-env
            - secretRef:
                name: openldap
          securityContext:
            privileged: true
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: openldap-data
        - name: custom-ldif-files
          configMap:
            name: openldap-customldif
            defaultMode: 420
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: openldap
  namespace: wfps-openldap
  labels:
    app: wfps-openldap
spec:
  ports:
    - name: ldap-port
      protocol: TCP
      port: 389
      targetPort: ldap-port
  type: NodePort
  selector:
    app: wfps-openldap
" | oc apply -f -
```

Wait for pod in *wfps-openldap* Project to become Ready 1/1.
```bash
oc get pod -n wfps-openldap -w
```

#### Access info after deployment

OpenLDAP
- See Project wfps-openldap, Service openldap for assigned NodePort 
- cn=admin,dc=cp,dc=internal / Password
- In OpenLDAP Pod terminal `ldapsearch -x -b "dc=cp,dc=internal" -H ldap://localhost:1389 -D 'cn=admin,dc=cp,dc=internal' -W "objectclass=*"`

### PostgreSQL

Create wfps-postgresql Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: wfps-postgresql
  labels:
    app: wfps-postgresql
" | oc apply -f -
```

Add required privileged permission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: postgresql-privileged
  namespace: wfps-postgresql
  labels:
    app: wfps-postgresql
subjects:
  - kind: ServiceAccount
    name: default
    namespace: wfps-postgresql
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
" | oc apply -f -
```

Create Secret for config
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: postgresql-config
  namespace: wfps-postgresql
  labels:
    app: wfps-postgresql
stringData:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: cpadmin
  POSTGRES_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-data
  namespace: wfps-postgresql
  labels:
    app: wfps-postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create PVC for table spaces as they should not be in PGDATA
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-tablespaces
  namespace: wfps-postgresql
  labels:
    app: wfps-postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create StatefulSet
```bash
echo "
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: wfps-postgresql
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: wfps-postgresql
  template:
    metadata:
      labels:
        app: wfps-postgresql
    spec:
      containers:
        - name: postgresql
          args:
            - '-c'
            - max_prepared_transactions=500
            - '-c'
            - max_connections=500
          resources:
            limits:
              memory: 4Gi
            requests:
              memory: 4Gi
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 6
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 6
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          startupProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 18
            periodSeconds: 10            
          securityContext:
            privileged: true
          image: postgres:14.7-alpine3.17
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgresql-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresql-data
            - mountPath: /pgsqldata
              name: postgresql-tablespaces
      volumes:
        - name: postgresql-data
          persistentVolumeClaim:
            claimName: postgresql-data
        - name: postgresql-tablespaces
          persistentVolumeClaim:
            claimName: postgresql-tablespaces
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: postgresql
  namespace: wfps-postgresql    
  labels:
    app: wfps-postgresql
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: wfps-postgresql
" | oc apply -f -
```

Wait for pod in *wfps-postgresql* Project to become Ready 1/1.
```bash
oc get pod -n wfps-postgresql -w
```

#### Access info after deployment

PostgreSQL
- See Project wfps-postgresql, Service postgresql for assigned NodePort 
- cpadmin / Password
- In PostgreSQL Pod terminal `psql postgresql://cpadmin@localhost:5432/postgresdb`

## Workflo Process Service Development Environment

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-multi-pattern-production-deployment

### Preparing a client to connect to the cluster

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-preparing-client-connect-cluster

CASE packages repository https://github.com/IBM/cloud-pak/tree/master/repo/case/ibm-cp-automation

```bash
# Create directory for wfps-dev
mkdir /usr/install/wfps-dev

# Download the package
curl https://raw.githubusercontent.com/IBM/cloud-pak/master/repo/case/\
ibm-cp-automation/5.1.1/ibm-cp-automation-5.1.1.tgz \
--output /usr/install/wfps-dev/ibm-cp-automation-5.1.1.tgz

# Extract the package
tar xzvf /usr/install/wfps-dev/ibm-cp-automation-5.1.1.tgz -C /usr/install/wfps-dev/

# Extract cert-kubernetes
tar xvf /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-k8s-23.0.2.tar \
-C /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/
```

### Setting up the cluster by running a script

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=cluster-setting-up-by-running-script

Initiate cluster admin setup
```bash
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-clusteradmin-setup.sh
```

```text
[INFO] Applying the latest IBM CP4BA Operator catalog source...

[✔] IBM CP4BA Operator catalog source Updated!

Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 2 # Based on your platform

What type of deployment is being performed?
ATTENTION: The BAI standalone only supports "Production" deployment type.
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

[NOTES] If you are planning to enable FIPS for CP4BA deployment, this script can perform a check on the OCP cluster to ensure the compute nodes have FIPS enabled.
Do you want to proceed with this check? (Yes/No, default: No): No

[NOTES] If your cluster is not connected to the internet, you can install Cloud Pak for Business Automation in an air gap environment with the IBM Catalog Management Plug-in. Use either a bastion host, or a portable compute/storage device to transfer the images to your air gap environment.

Do you want to deploy CP4BA using private catalog? (Yes/No, default: No): Yes

Where do you want to deploy Cloud Pak for Business Automation?
Enter the name for a new project or an existing project (namespace): wfps-dev
Using project wfps-dev...

The Cloud Pak for Business Automation Operator (Pod, CSV, Subscription) not found in cluster
Continue....

Using project wfps-dev...

Here are the existing users on this cluster: 
1) Cluster Admin
2) Your User
Enter an existing username in your cluster, valid option [1 to 3], non-admin is suggested: 2
ATTENTION: When you run cp4a-deployment.sh script, please use cluster admin user.

[INFO] Creating cp4ba-fips-status configMap in the project "wfps-dev"

[✔] Created cp4ba-fips-status configMap in the project "wfps-dev".

Follow the instructions on how to get your Entitlement Key: 
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=images-getting-access-from-public-entitled-registry

Do you have a Cloud Pak for Business Automation Entitlement Registry key (Yes/No, default: No): Yes

Enter your Entitlement Registry key: # it is OK that when you paste nothing is seen - it is for security.
Verifying the Entitlement Registry key...
Login Succeeded!
Entitlement Registry key is valid.

Please use available storage classes.

The existing storage classes in the cluster: 
...omitted

Creating docker-registry secret for Entitlement Registry key in project wfps-dev...
secret/ibm-entitlement-key created
Done


[INFO] Starting to install IBM Cert Manager and IBM Licensing Operator ...
......omitted Lots of content for IBM Cert Manager and IBM Licensing installation

Waiting for the Cloud Pak for Business Automation operator to be ready. This might take a few minutes...

ibm-cp4a-operator-catalog         ibm-cp4a-operator                 grpc   IBM         35m
Found existing ibm operator catalog source, updating it
catalogsource.operators.coreos.com/ibm-cp4a-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cs-flink-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cs-elastic-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cert-manager-catalog unchanged
catalogsource.operators.coreos.com/ibm-licensing-catalog unchanged
catalogsource.operators.coreos.com/opencloud-operators-v4-0 unchanged
catalogsource.operators.coreos.com/bts-operator unchanged
catalogsource.operators.coreos.com/cloud-native-postgresql-catalog unchanged
catalogsource.operators.coreos.com/ibm-fncm-operator-catalog unchanged
IBM Operator Catalog source updated!

[INFO] Waiting for CP4BA Operator Catalog pod initialization
...
[INFO] CP4BA Operator Catalog is running...
ibm-cp4a-operator-catalog-nqv59                                   1/1   Running     0             31s
operatorgroup.operators.coreos.com/ibm-cp4a-operator-catalog-group created
CP4BA Operator Group Created!
subscription.operators.coreos.com/ibm-cp4a-operator-catalog-subscription created
CP4BA Operator Subscription Created!


[INFO] Waiting for CP4BA operator pod initialization
..................CP4BA operator is running...
ibm-cp4a-operator-6d8b5ff754-nfk8n


[INFO] Waiting for CP4BA Content operator pod initialization
CP4BA Content operator is running...
ibm-content-operator-6c95fbb959-mn5t7


Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default not labeled
Done

Storage classes are needed to run the deployment script. For the Starter deployment scenario, you may use one (1) storage class.  For an Production deployment, the deployment script will ask for three (3) storage classes to meet the slow, medium, and fast storage for the configuration of CP4BA components.  If you don't have three (3) storage classes, you can use the same one for slow, medium, or fast.  Note that you can get the existing storage class(es) in the environment by running the following command: oc get storageclass. Take note of the storage classes that you want to use for deployment.
...omitted list of storage classes
```

Wait until the script finishes.

Wait until all Operators in Project wfps-dev are in *Succeeded* state.
```bash
oc get csv -n wfps-dev -w
```

### Prepare property files

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=pycc-recommended-preparing-databases-secrets-your-chosen-capabilities-by-running-script

Generate properties for components that you would like to deploy.
```bash
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-prerequisites.sh -m property
```

Answer as follows.
```text
Tips:Press [ENTER] to accept the default (None of the patterns is selected)
Enter a valid option [1 to 4, 5a, 5b, 6, 7a, 7b, 8]: 8

Hit Enter to continue.

Pattern "Workflow Process Service Authoring": Select optional components: 
1) Business Automation Insights 
2) Data Collector and Data Indexer 
3) Exposed Kafka Services 

Tips: Press [ENTER] if you do not want any optional components or when you are finished selecting your optional components
Enter a valid option [1 to 3 or ENTER]: 

Hit Enter to continue.

[INFO] LDAP configuration is not required for the IBM Workflow Process Service Authoring, but if you want to login with LDAP user, please select Yes. If you select No, you can do post actions with https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=cpbaf-business-automation-studio to add the LDAP connection manually after install.
Do you want use the LDAP for the IBM Workflow Process Service Authoring? (Yes/No): Yes

What is the LDAP type used for this deployment? 
1) Microsoft Active Directory
2) Tivoli Directory Server / Security Directory Server
Enter a valid option [1 to 2]: 2
[*] You can change the parameter "LDAP_SSL_ENABLED" in the property file "/usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property" later. "LDAP_SSL_ENABLED" is "TRUE" by default.

To provision the persistent volumes and volume claims
please enter the file storage classname for slow storage(RWX): ocs-storagecluster-cephfs
please enter the file storage classname for medium storage(RWX): ocs-storagecluster-cephfs
please enter the file storage classname for fast storage(RWX): ocs-storagecluster-cephfs
please enter the block storage classname for Zen(RWO): ocs-storagecluster-ceph-rbd

[INFO] By default the IBM Workflow Process Service Authoring Operator can provision an EDB postgreSQL database to use, if you want to use your own Postgreql, please select Yes. If you select No, and want to continue to use EDB Postgresql when adding more patterns together with WfPS in the future , you need to add EDB Postgresql info to database section under BAStudio configuration in your CR manually.
Do you want use the external Postgreql for the IBM Workflow Process Service Authoring? (Yes/No): Yes

Please select the deployment profile (default: small).  Refer to the documentation in CP4BA Knowledge Center for details on profile.
1) small
2) medium
3) large
Enter a valid option [1 to 3]: 1

What is the Database type used for this deployment? 
1) PostgreSQL
Enter a valid option [1 to 1]: 1

[*] You can change the parameter "DATABASE_SSL_ENABLE" in the property file "/usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property" later. "DATABASE_SSL_ENABLE" is "TRUE" by default.
[*] You can change the parameter "POSTGRESQL_SSL_CLIENT_SERVER" in the property file "/usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property" later. "POSTGRESQL_SSL_CLIENT_SERVER" is "TRUE" by default
[*] - POSTGRESQL_SSL_CLIENT_SERVER="True": For a PostgreSQL database with both server and client authentication
[*] - POSTGRESQL_SSL_CLIENT_SERVER="False": For a PostgreSQL database with server-only authentication

Enter the alias name(s) for the database server(s)/instance(s) to be used by the CP4BA deployment.
(NOTE: NOT the host name of the database server, and CANNOT include a dot[.] character)
(NOTE: This key supports comma-separated lists (for example: dbserver1,dbserver2,dbserver3)
The alias name(s): postgresql

Where do you want to deploy Cloud Pak for Business Automation?
Enter the name for an existing project (namespace): wfps-dev

Do you want to restrict network egress to unknown external destination for this CP4BA deployment? (Notes: CP4BA 23.0.2 prevents all network egress to unknown destinations by default. You can either (1) enable all egress or (2) accept the new default and create network policies to allow your specific communication targets as documented in the knowledge center.) (Yes/No, default: Yes): Yes

How many object stores will be deployed for the content pattern? 1


============== Creating database and LDAP property files for CP4BA ==============
Creating DB Server property file for CP4BA
[✔] Created the DB Server property file for CP4BA

Creating LDAP Server property file for CP4BA
[✔] Created the LDAP Server property file for CP4BA


============== Creating property file for database name and user required by CP4BA ==============
Creating Property file for IBM FileNet Content Manager GCD
[✔] Created Property file for IBM FileNet Content Manager GCD

Creating Property file for IBM FileNet Content Manager Object Store required by BAW authoring or BAW Runtime
[✔] Created Property file for IBM FileNet Content Manager Object Store required by BAW authoring or BAW Runtime

Creating Property file for IBM FileNet Content Manager Object Store required by AE Data Persistent
[✔] Created Property file for IBM FileNet Content Manager Object Store required by AE Data Persistent
Creating Property file for IBM Business Automation Navigator
[✔] Created Property file for IBM Business Automation Navigator

Creating Property file for Application Engine
[✔] Created Property file for Application Engine

Creating Property file for IBM Business Automation Studio
[✔] Created Property file for IBM Business Automation Studio


============== Created all property files for CP4BA ==============
[NEXT ACTIONS]
Enter the <Required> values in the property files under /usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile
[*] The key name in the property file is created by the cp4a-prerequisites.sh and is NOT EDITABLE.
[*] The value in the property file must be within double quotes.
[*] The value for User/Password in [cp4ba_db_name_user.property] [cp4ba_user_profile.property] file should NOT include special characters "=" "." "\"
[*] The value in [cp4ba_LDAP.property] or [cp4ba_External_LDAP.property] [cp4ba_user_profile.property] file should NOT include special character '"'
* [cp4ba_db_server.property]:
  - Properties for database server used by CP4BA deployment, such as DATABASE_SERVERNAME/DATABASE_PORT/DATABASE_SSL_ENABLE.

  - The value of "<DB_SERVER_LIST>" is an alias for the database servers. The key supports comma-separated lists.

* [cp4ba_db_name_user.property]:
  - Properties for database name and user name required by each component of the CP4BA deployment, such as GCD_DB_NAME/GCD_DB_USER_NAME/GCD_DB_USER_PASSWORD.

  - Change the prefix "<DB_SERVER_NAME>" to assign which database is used by the component.

  - The value of "<DB_SERVER_NAME>" must match the value of <DB_SERVER_LIST> that is defined in "<DB_SERVER_LIST>" of "cp4ba_db_server.property".

* [cp4ba_LDAP.property]:
  - Properties for the LDAP server that is used by the CP4BA deployment, such as LDAP_SERVER/LDAP_PORT/LDAP_BASE_DN/LDAP_BIND_DN/LDAP_BIND_DN_PASSWORD.

* [cp4ba_user_profile.property]:
  - Properties for the global value used by the CP4BA deployment, such as "sc_deployment_license".

  - properties for the value used by each component of CP4BA, such as <APPLOGIN_USER>/<APPLOGIN_PASSWORD>
```

Update values for cp4ba_db_server.property

```bash
# Backup generated file
cp /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/\
\cp4ba-prerequisites/propertyfile/cp4ba_db_server.property \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/\
cp4ba-prerequisites/propertyfile/cp4ba_db_server.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.DATABASE_SERVERNAME="<Required>"/'\
'postgresql.DATABASE_SERVERNAME="postgresql.wfps-postgresql.svc.cluster.local"/g' \
-e 's/postgresql.DATABASE_PORT="<Required>"/postgresql.DATABASE_PORT="5432"/g' \
-e 's/postgresql.DATABASE_SSL_ENABLE="True"/postgresql.DATABASE_SSL_ENABLE="False"/g' \
-e 's/postgresql.POSTGRESQL_SSL_CLIENT_SERVER="True"/postgresql.POSTGRESQL_SSL_CLIENT_SERVER="False"/g' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property
```

Update values for cp4ba_db_name_user.property
```bash
# Backup generated file
cp /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/\
cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/\
cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.STUDIO_DB_NAME="BASDB"/postgresql.STUDIO_DB_NAME="DEVBAS"/g' \
-e 's/postgresql.STUDIO_DB_USER_NAME="<youruser1>"/postgresql.STUDIO_DB_USER_NAME="devbas"/g' \
-e 's/postgresql.STUDIO_DB_USER_PASSWORD="{Base64}<yourpassword>"/'\
'postgresql.STUDIO_DB_USER_PASSWORD="Password"/g' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/\
cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property
```

Update values for cp4ba_LDAP.property
```bash
# Backup generated file
cp /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property.bak

# Update generated file with real values
sed -i \
-e 's/LDAP_TYPE="IBM Security Directory Server"/LDAP_TYPE="Custom"/g' \
-e 's/LDAP_SERVER="<Required>"/LDAP_SERVER="openldap.wfps-openldap.svc.cluster.local"/g' \
-e 's/LDAP_PORT="<Required>"/LDAP_PORT="389"/g' \
-e 's/LDAP_BASE_DN="<Required>"/LDAP_BASE_DN="dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN="<Required>"/LDAP_BIND_DN="cn=admin,dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN_PASSWORD="{Base64}<Required>"/LDAP_BIND_DN_PASSWORD="Password"/g' \
-e 's/LDAP_SSL_ENABLED="True"/LDAP_SSL_ENABLED="False"/g' \
-e 's/LDAP_USER_NAME_ATTRIBUTE="\*:uid"/LDAP_USER_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_GROUP_BASE_DN="<Required>"/LDAP_GROUP_BASE_DN="ou=Groups,dc=cp,dc=internal"/g' \
-e 's/LC_USER_FILTER="(\&(cn=%v)(objectclass=person))"/'\
'LC_USER_FILTER="(\&(uid=%v)(objectclass=inetOrgPerson))"/g' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property
```

Update values for cp4ba_user_profile.property
```bash
# Backup generated file
cp /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property.bak

# Update generated file with real values
sed -i \
-e 's/CP4BA.CP4BA_LICENSE="<Required>"/CP4BA.CP4BA_LICENSE="non-production"/g' \
-e 's/CP4BA.BAW_LICENSE="<Required>"/CP4BA.BAW_LICENSE="non-production"/g' \
-e 's/BASTUDIO.ADMIN_USER="<Required>"/BASTUDIO.ADMIN_USER="cpadmin"/g' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/\
scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property
```

### Generate, update and apply SQL and Secret files

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=pycc-recommended-preparing-databases-secrets-your-chosen-capabilities-by-running-script

Generate SQL and Secrets
```bash
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-prerequisites.sh -m generate
```

Copy create scripts to PostgreSQL instance
```bash
oc cp /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/dbscript \
wfps-postgresql/$(oc get pods --namespace wfps-postgresql -o name | cut -d"/" -f2):/usr/dbscript-dev
```

Execute create scripts with table space directory creation
```bash
# Studio
oc --namespace wfps-postgresql exec statefulset/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin@localhost:5432/postgresdb \
--file=/usr/dbscript-dev/bas/postgresql/postgresql/create_bas_studio_db.sql'
```

Apply secrets which are in /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/.
```bash
# Switch to Project
oc project wfps-dev

# Apply secrets
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4ba-prerequisites/create_secret.sh
```

### Validate connectivity

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=pycc-recommended-preparing-databases-secrets-your-chosen-capabilities-by-running-script

```bash
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-prerequisites.sh -m validate
```
Look for all green checkmarks.

### Generating the custom resource with the deployment script

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-option-2a-generating-custom-resource-script

Run deployment script
```bash
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-deployment.sh
```

```text
IMPORTANT: Review the IBM Cloud Pak for Business Automation license information here: 

http://www14.software.ibm.com/cgi-bin/weblap/lap.pl?li_formnum=L-WJWD-9SAFJA

Press any key to continue

Do you accept the IBM Cloud Pak for Business Automation license (Yes/No, default: No): Yes

Did you deploy Content CR (CRD: contents.icp4a.ibm.com) in current cluster? (Yes/No, default: No): No

Starting to Install the Cloud Pak for Business Automation Operator...


What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

*******************************************************
            Summary of CP4BA capabilities              
*******************************************************
1. Cloud Pak capability to deploy: 
   * Workflow Process Service Authoring
2. Optional components to deploy: 
   * None
*******************************************************

[INFO] Above CP4BA capabilities is already selected in the cp4a-prerequisites.sh script
Press any key to continue

Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 2 # Based on your platform

ATTENTION: If you are unable to use [cpadmin] as the default IAM admin user due to it having the same user name in your LDAP Directory, you need to change the Cloud Pak administrator username. See: "https://www.ibm.com/docs/en/cpfs?topic=configurations-changing-cloud-pak-administrator-access-credentials#user-name"
Do you want to use the default IAM admin user: [cpadmin] (Yes/No, default: Yes): No

What is the non default IAM admin user you renamed?
Enter the admin user name: cpfsadmin

Provide a URL to zip file that contains JDBC and/or ICCSAP drivers.
(optional - if not provided, the Operator will configure using the default shipped JDBC driver): # Hit Enter - omit

The Operator will configure using the default shipped JDBC driver.

*******************************************************
                    Summary of input                   
*******************************************************
1. Cloud Pak capability to deploy: 
   * Workflow Process Service Authoring
2. Optional components to deploy: 
   * None
3. File storage classname(RWX):
   * Slow: ocs-storagecluster-cephfs
   * Medium: ocs-storagecluster-cephfs
   * Fast: ocs-storagecluster-cephfs
4. Block storage classname(RWO): ocs-storagecluster-ceph-rbd
5. URL to zip file for JDBC and/or ICCSAP drivers: 
*******************************************************

Verify that the information above is correct.
To proceed with the deployment, enter "Yes".
To make changes, enter "No" (default: No): Yes

Creating the Custom Resource of the Cloud Pak for Business Automation operator...

Applying value in property file into final CR
[✔] Applied value in property file into final CR under /usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr
Please confirm final custom resource under /usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr

The custom resource file used is: "/usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml"

ATTENTION: If the cluster is running a Linux on Z (s390x)/Power architecture, remove the baml_configuration section from "/usr/install/wfps-dev/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml" before you apply the custom resource. Business Automation Machine Learning Server (BAML) is not supported on this architecture.


To monitor the deployment status, follow the Operator logs.
For details, refer to the troubleshooting section in Knowledge Center here: 
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=automation-troubleshooting
```

### Checking and completing your custom resource

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-checking-completing-your-custom-resource

Change LDAP sub settings from tds to custom
```bash
sed -i \
-e 's/tds:/custom:/g' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

If you want to see all debug output in operator logs add the following properties (Not for production use, can leak sensitive info!)
```bash
yq -i '.spec.shared_configuration.no_log = false' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.shared_configuration.show_sensitive_log = true' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

### Deploying the custom resource you created with the deployment script

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=cpd-option-2b-deploying-custom-resource-you-created-deployment-script

Add permissive network policy to enable the deployment to reach to LDAP and DB (TODO - workaround, last needed 23.0.2.1)
```bash
echo "
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: custom-permit-db-egress
  namespace: wfps-dev
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector: {}
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: wfps-postgresql
" | oc apply -f -
echo "
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: custom-permit-ldap-egress
  namespace: wfps-dev
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector: {}
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: wfps-openldap
" | oc apply -f -
```

Apply CR  
```bash
oc apply -n wfps-dev -f /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Wait for the deployment to be completed. This can be determined by looking in Project wfps-dev in Kind ICP4ACluster, instance named icp4adeploy to have the following conditions:
```bash
oc get -n wfps-dev icp4acluster icp4adeploy -o=jsonpath="{.status.conditions}" | jq
```
```json
[
  {
    "message": "Running reconciliation",
    "reason": "Running",
    "status": "True",
    "type": "Running"
  },
  {
    "message": "",
    "reason": "Successful",
    "status": "True",
    "type": "Ready"
  },
  {
    "message": "Prerequisites execution done.",
    "reason": "Successful",
    "status": "True",
    "type": "PrereqReady"
  }
]
```

### Access info after deployment

- https://cpd-wfps-dev.<ocp_apps_domain>/
- cpadmin / Password
- Additional capabilities based info in Project wfps-dev in ConfigMap icp4adeploy-cp4ba-access-info
- To get cpfsadmin password ```oc get secret platform-auth-idp-credentials -n wfps-dev -o jsonpath='{.data.admin_password}' | base64 -d```

### Completing post-installation tasks

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-completing-post-installation-tasks  
and https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=tasks-workflow-process-service-authoring

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=cpbaf-business-automation-studio  
You need to setup permissions for your users.  
```bash
# Get password of zen initial admin
zen_admin_password=`oc get secret admin-user-details -n wfps-dev \
-o jsonpath='{.data.initial_admin_password}' | base64 -d`
echo $zen_admin_password

# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n wfps-dev \
-o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get zen token
# Based on https://cloud.ibm.com/apidocs/cloud-pak-data/cloud-pak-data-4.5.0#getauthorizationtoken
zen_token=`curl --silent -k -H "Content-Type: application/json" \
-d '{"username":"admin","password":"'$zen_admin_password'"}' \
https://cpd-wfps-dev.${apps_endpoint}/icp4d-api/v1/authorize | jq -r ".token"`
echo $zen_token

# Add all roles to cpadmin user to be usable for all tasks
curl -kv -X PUT -H "Content-Type: application/json" -H "Authorization: Bearer $zen_token" \
-d '{"username":"cpadmin", '\
'"user_roles":["zen_administrator_role","iaf-automation-admin","iaf-automation-analyst",'\
'"iaf-automation-developer","iaf-automation-operator","zen_user_role"]}' \
https://cpd-wfps-dev.${apps_endpoint}/usermgmt/v1/user/cpadmin?add_roles=true
```

### Validating your production deployment

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-recommended-validating-your-production

You can further verify the environemnts and get important information. But before running anything else then --help follow additional steps for script configuration.
```bash
oc project wfps-dev
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-post-install.sh --help
```

Perform setup for cp4a-post-install.sh script
```bash
# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n wfps-dev -o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get CPFS token
cpfs_token=`curl --silent -k --header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'username=cpadmin' \
--data-urlencode 'password=Password' \
--data-urlencode 'scope=openid' \
https://cp-console-wfps-dev.${apps_endpoint}/idprovider/v1/auth/identitytoken | jq -r ".access_token"`
echo $cpfs_token

# Exchange CPFS token for Zen token
zen_token=`curl --silent -k -H "Content-Type: application/json" \
--header "iam-token: ${cpfs_token}" \
--header 'username: cpadmin' \
https://cpd-wfps-dev.${apps_endpoint}/v1/preauth/validateAuth | jq -r ".accessToken"`
echo $zen_token

# Generate ZenApiKey
zen_api_key=`curl --silent -k -H "Content-Type: application/json" \
--header "Authorization: Bearer ${zen_token}" \
--header 'username: cpadmin' \
https://cpd-wfps-dev.${apps_endpoint}/usermgmt/v1/user/apiKey | jq -r ".apiKey"`
echo $zen_api_key

# Change CPFS namespace for cp4a-post-install.sh script and add password and zen api key
sed -i \
-e 's/CP4BA_COMMON_SERVICES_NAMESPACE="ibm-common-services"/'\
'CP4BA_COMMON_SERVICES_NAMESPACE="wfps-dev"/g' \
-e 's/PROBE_USER_API_KEY=/'\
'PROBE_USER_API_KEY="'${zen_api_key}'"/g' \
-e 's/PROBE_USER_NAME=/'\
'PROBE_USER_NAME="cpadmin"/g' \
-e 's/PROBE_USER_PASSWORD=/'\
'PROBE_USER_PASSWORD="Password"/g' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/helper/post-install/env.sh
```

Now you can run cp4a-post-install.sh with other parameters.
```bash
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-post-install.sh --precheck
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-post-install.sh --status
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-post-install.sh --console
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-post-install.sh --probe # Y to continue
```

### Next steps

Follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-completing-post-installation-tasks as needed.

Custom CPFS console TLS - follow https://www.ibm.com/docs/en/SSYHZ8_23.0.2/com.ibm.dba.managing/op_topics/tsk_custom_cp4ba_iam.html

Custom CPFS admin password - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=tasks-cloud-pak-foundational-services

Custom Zen certificates - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=security-customizing-cloud-pak-entry-point

## Workflow Process Service Test Environment

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-multi-pattern-production-deployment

### Preparing a client to connect to the cluster

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-preparing-client-connect-cluster

```bash
# Create directory for wfps-test
mkdir /usr/install/wfps-test

# Download the package
curl https://raw.githubusercontent.com/IBM/cloud-pak/master/repo/case/\
ibm-cp-automation/5.1.1/ibm-cp-automation-5.1.1.tgz \
--output /usr/install/wfps-test/ibm-cp-automation-5.1.1.tgz

# Extract the package
tar xzvf /usr/install/wfps-test/ibm-cp-automation-5.1.1.tgz -C /usr/install/wfps-test/

# Extract cert-kubernetes
tar xvf /usr/install/wfps-test/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-k8s-23.0.2.tar \
-C /usr/install/wfps-test/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/
```

### Setting up the cluster by running a script

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=cluster-setting-up-by-running-script

Initiate cluster admin setup
```bash
/usr/install/wfps-test/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/cp4a-clusteradmin-setup.sh
```

```text
Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 2 # Based on your platform

What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

[NOTES] If you are planning to enable FIPS for CP4BA deployment, this script can perform a check on the OCP cluster to ensure the compute nodes have FIPS enabled.
Do you want to proceed with this check? (Yes/No, default: No): No

[NOTES] If your cluster is not connected to the internet, you can install Cloud Pak for Business Automation in an air gap environment with the IBM Catalog Management Plug-in. Use either a bastion host, or a portable compute/storage device to transfer the images to your air gap environment.

Do you want to deploy CP4BA using private catalog? (Yes/No, default: No): Yes

Where do you want to deploy Cloud Pak for Business Automation?
Enter the name for a new project or an existing project (namespace): wfps-test

Here are the existing users on this cluster: 
1) Cluster Admin
2) Your User
Enter an existing username in your cluster, valid option [1 to 3], non-admin is suggested: 2
ATTENTION: When you run cp4a-deployment.sh script, please use cluster admin user.

[INFO] Creating cp4ba-fips-status configMap in the project "wfps-test"

[✔] Created cp4ba-fips-status configMap in the project "wfps-test".

Follow the instructions on how to get your Entitlement Key: 
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=images-getting-access-from-public-entitled-registry

Do you have a Cloud Pak for Business Automation Entitlement Registry key (Yes/No, default: No): Yes

Enter your Entitlement Registry key: # it is OK that when you paste nothing is seen - it is for security.
Verifying the Entitlement Registry key...
Login Succeeded!
Entitlement Registry key is valid.

Please use available storage classes.

The existing storage classes in the cluster: 
...omitted

Creating docker-registry secret for Entitlement Registry key in project wfps-test...
secret/ibm-entitlement-key created
Done

[INFO] Starting to install IBM Cert Manager and IBM Licensing Operator ...
......omitted Lots of content for IBM Cert Manager and IBM Licensing installation

Waiting for the Cloud Pak for Business Automation operator to be ready. This might take a few minutes... 

ibm-cp4a-operator-catalog         ibm-cp4a-operator                 grpc   IBM         17h
Found existing ibm operator catalog source, updating it
catalogsource.operators.coreos.com/ibm-cp4a-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cs-flink-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cs-elastic-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cert-manager-catalog unchanged
catalogsource.operators.coreos.com/ibm-licensing-catalog unchanged
catalogsource.operators.coreos.com/opencloud-operators-v4-0 unchanged
catalogsource.operators.coreos.com/bts-operator unchanged
catalogsource.operators.coreos.com/cloud-native-postgresql-catalog unchanged
catalogsource.operators.coreos.com/ibm-fncm-operator-catalog unchanged
IBM Operator Catalog source updated!

[INFO] Waiting for CP4BA Operator Catalog pod initialization

[INFO] CP4BA Operator Catalog is running...
ibm-cp4a-operator-catalog-nqv59                                   1/1   Running     0             17h
operatorgroup.operators.coreos.com/ibm-cp4a-operator-catalog-group created
CP4BA Operator Group Created!
subscription.operators.coreos.com/ibm-cp4a-operator-catalog-subscription created
CP4BA Operator Subscription Created!

[INFO] Waiting for CP4BA operator pod initialization
............CP4BA operator is running...
ibm-cp4a-operator-6495dcd4b8-4bvcf


[INFO] Waiting for CP4BA Content operator pod initialization
CP4BA Content operator is running...
ibm-content-operator-59c69795f7-lpk5p


Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default not labeled
Done

Storage classes are needed to run the deployment script. For the Starter deployment scenario, you may use one (1) storage class.  For an Production deployment, the deployment script will ask for three (3) storage classes to meet the slow, medium, and fast storage for the configuration of CP4BA components.  If you don't have three (3) storage classes, you can use the same one for slow, medium, or fast.  Note that you can get the existing storage class(es) in the environment by running the following command: oc get storageclass. Take note of the storage classes that you want to use for deployment. 
...omitted list of storage classes
```

Wait until the script finishes.

Wait until all Operators in Project wfps-test are in *Succeeded* state.
```bash
oc get csv -n wfps-test -w
```

### Deploy minmal CP4BA

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-workflow-process-service-production-deployment


Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=parameters-ldap-configuration

Add ldap bind secret
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: ldap-bind-secret
  namespace: wfps-test
stringData:
  ldapPassword: Password
  ldapUsername: cn=admin,dc=cp,dc=internal
type: Opaque
" | oc apply -f -
```

Create CP4BA Deployment yaml file
```bash
cat >/usr/install/wfps-test/cp4ba.yaml <<EOF
apiVersion: icp4a.ibm.com/v1
kind: ICP4ACluster
metadata:
   name: icp4adeploy
   namespace: wfps-test
   labels:
     app.kubernetes.io/instance: ibm-dba
     app.kubernetes.io/managed-by: ibm-dba
     app.kubernetes.io/name: ibm-dba
     release: 23.0.2
spec:
  appVersion: 23.0.2
  ibm_license: accept
  shared_configuration:
    show_sensitive_log: true
    ## Use this parameter to specify the license for the IBM Cloud Pak for Business Automation deployment for the rest of the Cloud Pak for Business Automation components.
    ## This value could differ from the rest of the licenses.
    sc_deployment_license: production
    sc_deployment_type: production
    ## On OCP 3.x and 4.x, the user script will populate these three (3) parameters based on your input for "enterprise" deployment.
    ## If you are manually deploying without using the user script, then you would provide the different storage classes for the slow, medium
    ## and fast storage parameters below. If you only have 1 storage class defined, then you can use that 1 storage class for all 3 parameters.
    storage_configuration:
      sc_slow_file_storage_classname: ocs-storagecluster-cephfs
      sc_medium_file_storage_classname: ocs-storagecluster-cephfs
      sc_fast_file_storage_classname: ocs-storagecluster-cephfs
      sc_block_storage_classname: ocs-storagecluster-ceph-rbd # Added manually for Zen
    sc_deployment_platform: OCP
    no_log: false  # Added manually for better logging
    sc_iam:
      default_admin_username: cpfsadmin # Added manually to customize CPFS admin username
    sc_egress_configuration:
      sc_api_namespace: null
      sc_api_port: null
      sc_dns_namespace: null
      sc_dns_port: null
      sc_restricted_internet_access: true
  ldap_configuration: # Added manually for LDAP connection
    lc_ldap_user_display_name_attr: cn
    lc_ldap_group_base_dn: 'ou=Groups,dc=cp,dc=internal'
    lc_ldap_base_dn: 'dc=cp,dc=internal'
    lc_bind_secret: ldap-bind-secret
    lc_ldap_user_name_attribute: '*:cn'
    lc_ldap_group_member_id_map: 'groupofnames:member'
    lc_ldap_port: '389'
    custom:
      lc_group_filter: >-
        (&(cn=%v)(|(objectclass=groupofnames)(objectclass=groupofuniquenames)(objectclass=groupofurls)))
      lc_user_filter: (&(uid=%v)(objectclass=inetOrgPerson))
    lc_ldap_server: openldap.wfps-openldap.svc.cluster.local
    lc_ldap_group_membership_search_filter: >-
      (|(&(objectclass=groupofnames)(member={0}))(&(objectclass=groupofuniquenames)(uniquemember={0})))
    lc_selected_ldap_type: Custom
    lc_ldap_ssl_secret_name: ibm-cp4ba-ldap-ssl-secret
    lc_ldap_group_name_attribute: '*:cn'
    lc_ldap_group_display_name_attr: cn
    lc_ldap_ssl_enabled: false     
  ## this field is required to deploy Resource Registry (RR)
  resource_registry_configuration:
    replica_size: 1
EOF
```

Add minimal CP4BA CR
```bash
oc apply -f /usr/install/wfps-test/cp4ba.yaml
```

Wait for the deployment to be completed. This can be determined by looking in Project wfps-test in Kind ICP4ACluster, instance named icp4adeploy to have the following conditions:
```bash
oc get -n wfps-test icp4acluster icp4adeploy -o=jsonpath="{.status.conditions}" | jq
```
```json
[
  {
    "message": "Running reconciliation",
    "reason": "Running",
    "status": "True",
    "type": "Running"
  },
  {
    "message": "",
    "reason": "Successful",
    "status": "True",
    "type": "Ready"
  },
  {
    "message": "Prerequisites execution done.",
    "reason": "Successful",
    "status": "True",
    "type": "PrereqReady"
  }
]
```

### Add WfPS deplyoment

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-workflow-process-service-production-deployment#tsk_prepare_wfps__deploy__title__1

Create WfPS DB in PostgreSQL

```bash
# Create SQL file
cat >/usr/install/wfps-test/wfps-test.sql <<EOF
-- create the user
CREATE ROLE testwfps WITH INHERIT LOGIN ENCRYPTED PASSWORD 'Password';

-- create the database:
CREATE DATABASE testwfps WITH OWNER testwfps ENCODING 'UTF8';

-- Connect to your database and create schema
\c testwfps;
CREATE SCHEMA IF NOT EXISTS testwfps AUTHORIZATION testwfps;
GRANT ALL ON schema testwfps to testwfps;
EOF

# Copy the SQL to postgresql
oc cp /usr/install/wfps-test/wfps-test.sql \
wfps-postgresql/$(oc get pods --namespace wfps-postgresql -o name | cut -d"/" -f2):/usr/wfps-test.sql

# Execute the SQL
oc --namespace wfps-postgresql exec statefulset/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin@localhost:5432/postgresdb \
--file=/usr/wfps-test.sql'
```

Create WfPS DB secret

```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: wfps-db-secret
  namespace: wfps-test
type: Opaque
stringData:
  username: "testwfps"
  password: "Password"
" | oc apply -f -
```

Create WfPS Deployment yaml file
```bash
cat > /usr/install/wfps-test/wfps.yaml << EOF
kind: WfPSRuntime
apiVersion: icp4a.ibm.com/v1
metadata:
  name: wfps-test-instance1
  namespace: wfps-test
spec:
  appVersion: 23.0.2
  deploymentLicense: non-production
  admin:
    username: cpadmin
  license:
    accept: true
  persistent:
    storageClassName: ocs-storagecluster-cephfs
    data:
      enable: true
      size: 1Gi
    dump:
      enable: true
      size: 1Gi
    logs:
      enable: true
      size: 1Gi
  database:
    external:
      type: postgresql
      serverName: postgresql.wfps-postgresql.svc.cluster.local
      port: 5432
      enableSSL: false
      databaseName: testwfps
      dbCredentialSecret: wfps-db-secret
  node:
    serverType: Test
EOF
```

Apply WFPS into project
```bash
oc apply -f /usr/install/wfps-test/wfps.yaml
```

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-workflow-process-service-production-deployment#tsk_prepare_wfps__verify__title__1  
Wait for the deployment to be completed. This can be determined by looking in Project wfps-test in Kind WfPSRuntime, instance named wfps-test-instance1 to have the following conditions:
```bash
oc get -n wfps-test WfPSRuntime wfps-test-instance1 -o=jsonpath="{.status.conditions}" | jq
```
```json
[
  {
    "lastTransitionTime": "2024-02-13T07:50:19Z",
    "message": "IWfPS installation is completed successfully",
    "reason": "Ready",
    "status": "True",
    "type": "Ready"
  },
  {
    "lastTransitionTime": "2024-02-06T18:30:59Z",
    "message": "License checking passed",
    "reason": "Passed",
    "status": "True",
    "type": "LicenseChecked"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:02Z",
    "message": "Deploy Zen successfully",
    "reason": "Deployed",
    "status": "True",
    "type": "ZenProgressReady"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:02Z",
    "message": "ICP4A prerequisition checking passed",
    "reason": "Validated",
    "status": "True",
    "type": "ICP4AClusterReady"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:02Z",
    "message": "IAM clients created successfully",
    "reason": "Registered",
    "status": "True",
    "type": "IAMClientRegistered"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:03Z",
    "message": "IWfPS server network policy is available",
    "reason": "Created",
    "status": "True",
    "type": "IWfPSNetworkPolicyReady"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:04Z",
    "message": "ConfigMap for IWfPS configuration is available",
    "reason": "Created",
    "status": "True",
    "type": "ConfigMapReady"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:04Z",
    "message": "IWfPS server service is available",
    "reason": "Created",
    "status": "True",
    "type": "IWfPSServiceReady"
  },
  {
    "lastTransitionTime": "2024-02-06T18:31:04Z",
    "message": "IWfPS server route is available",
    "reason": "Created",
    "status": "True",
    "type": "IWfPSRouteReady"
  },
  {
    "lastTransitionTime": "2024-02-13T07:50:19Z",
    "message": "IWfPS server instance is deployed and available",
    "reason": "Available",
    "status": "True",
    "type": "IWfPSServerReady"
  },
  {
    "lastTransitionTime": "2024-02-06T18:34:17Z",
    "message": "Deploy ZenExtension successfully",
    "reason": "Completed",
    "status": "True",
    "type": "ZenExtensionReady"
  }
]
```

### Access info after deployment

- https://cpd-wfps-test.<ocp_apps_domain>/
- cpadmin / Password
- Additional info about endpoints is in WfPS CR ```oc get -n wfps-test WfPSRuntime wfps-test-instance1 -o=jsonpath="{.status.endpoints}" | jq```
- To get cpfsadmin password ```oc get secret platform-auth-idp-credentials -n wfps-test -o jsonpath='{.data.admin_password}' | base64 -d```

### Completing post-installation tasks

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-workflow-process-service-production-deployment#tsk_prepare_wfps__post_deploy__title__1

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=cpbaf-business-automation-studio  

You need to setup permissions for your users.  
```bash
# Get password of zen initial admin
zen_admin_password=`oc get secret admin-user-details -n wfps-test \
-o jsonpath='{.data.initial_admin_password}' | base64 -d`
echo $zen_admin_password

# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n wfps-test \
-o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get zen token
# Based on https://cloud.ibm.com/apidocs/cloud-pak-data/cloud-pak-data-4.5.0#getauthorizationtoken
zen_token=`curl --silent -k -H "Content-Type: application/json" \
-d '{"username":"admin","password":"'$zen_admin_password'"}' \
https://cpd-wfps-test.${apps_endpoint}/icp4d-api/v1/authorize | jq -r ".token"`
echo $zen_token

# Also add all roles to cpadmin user to be usable for all tasks
curl -kv -X PUT -H "Content-Type: application/json" -H "Authorization: Bearer $zen_token" \
-d '{"username":"cpadmin", '\
'"user_roles":["zen_administrator_role","iaf-automation-admin","iaf-automation-analyst",'\
'"iaf-automation-developer","iaf-automation-operator","zen_user_role"]}' \
https://cpd-wfps-test.${apps_endpoint}/usermgmt/v1/user/cpadmin?add_roles=true
```

### Next steps

Optional custom CPFS console TLS - follow https://www.ibm.com/docs/en/SSYHZ8_23.0.2/com.ibm.dba.managing/op_topics/tsk_custom_cp4ba_iam.html

Optional custom Zen certificates - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=security-customizing-cloud-pak-entry-point

Optional custom CPFS admin password - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=tasks-cloud-pak-foundational-services

#### Connect to Authoring

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=icwpspd-optional-configuring-workflow-process-service-runtime-work-workflow-process-service-authoring  
The exchange needs to happen periodically based on the validity of the certificates.

```bash
# Get Zen CA from dev environment
oc get secret iaf-system-automationui-aui-zen-ca -n wfps-dev \
-o template --template='{{ index .data "tls.crt" }}' \
| base64 --decode > /usr/install/zenRootCAPC.cert

# Create secret on Test with Zen CA from Dev
oc create secret generic wfps-tls-zen-secret -n wfps-test \
--from-file=tls.crt=/usr/install/zenRootCAPC.cert

# Get Zen CA from test environment
oc get secret iaf-system-automationui-aui-zen-ca -n wfps-test \
-o template --template='{{ index .data "tls.crt" }}' \
| base64 --decode > /usr/install/zenRootCAPS.cert

# Get CPFS CA from test environment
oc get secret cs-ca-certificate-secret -n wfps-test \
-o template --template='{{ index .data "tls.crt" }}' \
| base64 --decode > /usr/install/csRootCAPS.cert

# Create secret on Dev with Zen CA and CPFS CA from Test
oc create secret generic wfpsaut-tls-zen-secret -n wfps-dev \
--from-file=tls.crt=/usr/install/zenRootCAPS.cert
oc create secret generic wfpsaut-tls-cs-secret -n wfps-dev \
--from-file=tls.crt=/usr/install/csRootCAPS.cert

# Get Router CA from Test environment (We have same cluster for both Dev and Test in our scenario)
oc get secret router-ca -n openshift-ingress-operator \
-o template --template='{{ index .data "tls.crt" }}' \
| base64 --decode > /usr/install/routercaPS.cert

# Create secret on Dev with Router CA from Test
oc create secret generic wfpsaut-routerca-secret -n wfps-dev \
--from-file=tls.crt=/usr/install/routercaPS.cert

# Get Router CA from Dev environment (We have same cluster for both Dev and Test in our scenario)
oc get secret router-ca -n openshift-ingress-operator \
-o template --template='{{ index .data "tls.crt" }}' \
| base64 --decode > /usr/install/routercaPC.cert

# Create secret on Test with Router CA from Dev
oc create secret generic wfps-routerca-secret -n wfps-test \
--from-file=tls.crt=/usr/install/routercaPC.cert
```

```bash
# Create secret with WfPS Authoring credentials in Test 
echo "
kind: Secret
apiVersion: v1
metadata:
  name: ibm-wfps-wc-secret
  namespace: wfps-test
type: Opaque
stringData:
  username: cpadmin
  password: Password
" | oc apply -f -

# Configure WfPS Runtime to trust WfPS Authoring TLS
yq -i '.spec.tls = {"serverTrustCertificateList": ["wfps-tls-zen-secret", "wfps-routerca-secret"]}' \
/usr/install/wfps-test/wfps.yaml

# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n wfps-dev -o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Configure WfPS Runtime to be able to connect to WfPS Authoring
yq -i '.spec.authoringServer = '\
'{"url": "https://cpd-wfps-dev.'$apps_endpoint'/bas/ProcessCenter", '\
'"secretName": "ibm-wfps-wc-secret", "heartbeatInterval": 30, '\
'"webPDUrl": "https://cpd-wfps-dev.'$apps_endpoint'/bas/WebPD"}' \
/usr/install/wfps-test/wfps.yaml
```

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=security-configuring-cluster#task_egd_lwt_wlb__external
```bash
# Add permissive Network Policy for WfPS Runtime to be able to reach WfPS Authoring
# This policy should be further limited to a specific IP of either internal router 
# if both Runtime and Authoring are running in the same cluster 
# or to a specific external IP of a different cluster if Runtime and Authroing are running in separate clusters
echo "
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: icp4adeploy-cp4a-egress-external-app-wfps
  namespace: wfps-test
spec:
  podSelector:
    matchLabels:
      com.ibm.cp4a.networking/egress-external-app-component: WfPS
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: TCP
          port: 443
" | oc apply -f -
```

```bash
# Configure WfPS Authoring to trust WfPS Runtime TLS
yq -i '.spec.bastudio_configuration.tls = {"tlsTrustList": '\
'["wfpsaut-tls-zen-secret", "wfpsaut-tls-cs-secret", "wfpsaut-routerca-secret"]}' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml

# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n wfps-test -o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Configure WfPS Authoring to trust WfPS Runtime URLs
yq -i '.spec.workflow_authoring_configuration.environment_config = '\
'{"content_security_policy_additional_all":["https://cpd-wfps-test.'$apps_endpoint'/wfps-test-instance1-wfps/"],'\
'"csrf":{"origin_allowlist":'\
'"https://cpd-wfps-test.'$apps_endpoint',https://cp-console-wfps-test.'$apps_endpoint'",'\
'"referer_allowlist":'\
'"cpd-wfps-test.'$apps_endpoint',cp-console-wfps-test.'$apps_endpoint'"}}' \
/usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=security-configuring-cluster#task_egd_lwt_wlb__external
```bash
# Add permissive Network Policy for WfPS Authoring to be able to reach WfPS Runtime
# This policy should be further limited to a specific IP of either internal router 
# if both Runtime and Authoring are running in the same cluster 
# or to a specific external IP of a different cluster if Runtime and Authroing are running in separate clusters
echo "
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: icp4adeploy-cp4a-egress-external-app-bas
  namespace: wfps-dev
spec:
  podSelector:
    matchLabels:
      com.ibm.cp4a.networking/egress-external-app-component: BAS
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: TCP
          port: 443
" | oc apply -f -
```

```bash
# Apply updated wfps-dev CR
oc apply -n wfps-dev -f /usr/install/wfps-dev/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml

# Apply updated wfps-test CR
oc apply -n wfps-test -f /usr/install/wfps-test/wfps.yaml
```

Now you need to wait for next Operator cycle to propagate the changes from CR to WfPS pods in both dev and test environment.

Good indicator of the change for wfps-test is existence of AUTHORING_SERVER_URL environment parameter in WfPS Runtime
```bash
oc get statefulset wfps-test-instance1-wfps-runtime-server -n wfps-test -o yaml | grep AUTHORING_SERVER_URL
```

Good indicator of the change for wfps-dev is existence of wfpsaut-tls-zen-secret secret mapping in WfPS Authoring
```bash
oc get statefulset icp4adeploy-bastudio-deployment -n wfps-dev -o yaml | grep wfpsaut-tls-zen-secret
```

## Contacts

Jan Dušek  
jdusek@cz.ibm.com  
Business Automation Partner Technical Specialist  
IBM Czech Republic

## Notice

© Copyright IBM Corporation 2024.
