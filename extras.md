# Shoud not be used - not finished

Would start with cp4ba deployment of application engine with data persistence which would bring FNCM, BAN in terms of configurationand SQLs which is needed for PFS Workplace with settings saved

#### Enabling business events with IBM Business Automation Insights

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-customizing-workflow-process-service-runtime#tsk_customize_wfps__bai_events

Prepare custom secret for BAI credentials
```bash
echo "
apiVersion: v1
kind: Secret
metadata:
  name: custom-bai-secret
  namespace: wfps-test
type: Opaque
stringData:
  management-username: cpadmin
  management-password: Password
" | oc apply -f -  
```

Add BAI configuration to CP4BA CR
```bash
yq -i '.spec.bai_configuration = '\
'{
  "bai_secret": "custom-bai-secret",
  "settings": {
    "egress": true
  },
  "flink": {
    "create_route": true
  },
  "management": null,
  "bpmn": {
    "install": true,
    "force_elasticsearch_timeseries": true
  }
}' \
/usr/install/wfps-test/cp4ba.yaml

yq -i '.spec.shared_configuration.sc_optional_components = "bai"' \
/usr/install/wfps-test/cp4ba.yaml
```

Based on https://www.ibm.com/docs/en/cloud-paks/1.0?topic=configuration-operational-datastore#updating-and-preconfiguring-the-superuser-password  
Preconfigure elasticsearch password
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: iaf-system-elasticsearch-es-default-user
  namespace: wfps-test
  labels:
    app.kubernetes.io/component: es
    app.kubernetes.io/instance: iaf-system
    app.kubernetes.io/name: elasticsearch
    elastic.automation.ibm.com/cr-name: iaf-system
stringData:
  username: cpadmin
  password: Password
type: kubernetes.io/basic-auth
" | oc apply -f -
```

Update CP4BA CR in cluster
```bash
oc apply -f /usr/install/wfps-test/cp4ba.yaml
```

TODO verify BAI deployed - maybe by running pods? or cp4ba cr status?


Add events emission configuration to WfPS
```bash
yq -i '.spec.businessEvent = '\
'{
  "enable": true,
  "enableTaskRecord": true,
  "enableTaskApi": true,
  "subscription": [
    {
      "appName": "*",
      "componentName": "*",
      "componentType": "*",
      "elementName": "*",
      "elementType": "*",
      "nature": "*",
      "version": "*"
    }
  ]
}' \
/usr/install/wfps-test/wfps.yaml
```

Update WfPS CR in cluster
```bash
oc apply -f /usr/install/wfps-test/wfps.yaml
```

TODO verify it works

#### Configuring Workflow Process Service Runtime for federation

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployment-customizing-workflow-process-service-runtime#tsk_customize_wfps__pfs__title__1  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/23.0.2?topic=deployments-installing-cp4ba-process-federation-server-production-deployment

```bash
echo "
apiVersion: icp4a.ibm.com/v1
kind: ProcessFederationServer
metadata:
  name: pfsdeploy
spec:
  appVersion: 23.0.2
  license:
    accept: true
  shared_configuration: 
    sc_deployment_license: non-production
    storage_configuration:
      sc_fast_file_storage_classname: ocs-storagecluster-cephfs
      sc_medium_file_storage_classname: ocs-storagecluster-cephfs
      sc_slow_file_storage_classname: ocs-storagecluster-cephfs
  pfs_configuration:
    replicas: 1
" | oc apply -f -
```

TODO verify PFS up and running 

Add federation configuration to WfPS
```bash
yq -i '.spec.capabilities = '\
'{
  "fullTextSearch": {
    "enable": true,
    "adminGroups": [
      "cpadmins"
    ],
    "esStorage": {
      "storageClassName": "ocs-storagecluster-ceph-rbd",
      "size": "50Gi"
    },
    "esSnapshotStorage": {
      "storageClassName": "ocs-storagecluster-ceph-rbd",
      "size": "10Gi"
    }
  },
  "federate": {
    "enable": true
  }
}' \
/usr/install/wfps-test/wfps.yaml
```

Update WfPS CR in cluster
```bash
oc apply -f /usr/install/wfps-test/wfps.yaml
```