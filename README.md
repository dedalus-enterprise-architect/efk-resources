# OpenShift Logging Resources

This project collects some procedures on how to setup a custom EFK instance based on the following minimum requirements:

 * RedHat OpenShift Cluster Logging Operator - version 5.7.5

 * RedHat ElasticSearch Operator - version 5.7.5
 
 * OpenShift 4.12 or higher

References:
  - https://github.com/openshift/cluster-logging-operator
  - https://github.com/openshift/elasticsearch-operator

## OpenShift Cluster Logging: Overview

This project focus on the following topics:

    * logging persistent storage
    * custom indexes template in order to avoiding the field's map explosion
    * improving the indexes retention
    * kibana custom structured field view

Explore the files used by this project:

* ```deploy/clusterlogging/cl-forwarder.yml``` : __cluster forwarder instance__ definition

* ```deploy/clusterlogging/cl-instance.template.yml``` : __cluster logging instance__ definition for a single pod deployment

* ```deploy/clusterlogging/cl-instance-ha.template.yml``` : __cluster logging instance__ definition for a high-availability deployment; _tolerations_ directives have been implemented for Infra node roles, uncomment them when needed

* ```deploy/clusterlogging/cl-operator.yml``` : template to install the RedHat Openshift Cluster Logging Operator stack

* ```deploy/elasticsearch/es-operator.yml``` : template to install the RedHat ElasticSearch Operator

* ```deploy/elasticsearch/index_explicit_mapping_template.sh``` : script to create a custom index template on ElasticSearch

* ```deploy/kibana/kibana-externallink.template.yml``` : template to create a Route to publish Kibana link aimed to have a custom fields view available as default

### Project minimum requirements

* The OpenShift client utility: ```oc```

* A cluster admin roles rights

* RedHat Operators catalog available on cluster

* Git project local clone

### RedHat ElasticSearch Operator: setup

> WARNING: an Admin Cluster Role is required to proceed on this section.

Run the following command to install the RedHat ElasticSearch Operator:

```
   oc apply -f deploy/elasticsearch/es-operator.yml
```

Approve the InstallPlan:

```
oc patch installplan $(oc get ip -n openshift-operators-redhat -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n openshift-operators-redhat --type merge --patch '{"spec":{"approved":true}}'
```

> Check objects

Get a list of the objects created:

```
   oc get all,ConfigMap,Secret,Elasticsearch,OperatorGroup,Subscription -l app=es-logging-dedalus --no-headers -n openshift-operators-redhat |cut -d' ' -f1
```

### RedHat OpenShift Cluster Logging Operator: setup

> WARNING: an Admin Cluster Role is required to proceed on this section.

Run the following command to install the RedHat OpenShift Cluster Logging Operator:

1. Instanciate the _Cluster Logging Operator_:

```
   oc apply -f deploy/clusterlogging/cl-operator.yml -n openshift-logging
```

2. Instanciate the _ClusterLogging_ instance with inline parameters:

```
   oc process -f deploy/clusterlogging/cl-instance.template.yml \
     -p STORAGECLASS=@type_here_the_custom_storageclass@ \
     | oc -n openshift-logging apply -f -
```

3. Instanciate the _Cluster Forwarder_:

```
   oc apply -f deploy/clusterlogging/cl-forwarder.yml -n openshift-logging
```

> Check objects

Get a list of the objects created:

```
   oc get all,ConfigMap,Secret,OperatorGroup,Subscription,ClusterLogging,ClusterLogForwarder \
     -l app=cl-logging-dedalus --no-headers -n openshift-logging |cut -d' ' -f1
```

### Kibana: create the External Console Link

> WARNING: an Admin Cluster Role is required to proceed on this section.

Run the following command to create the External Console Link for Kibana default View:

```
   oc process -f deploy/kibana/kibana-externallink.template.yml \
     -p KIBANA_ROUTE=$(oc get route kibana -n openshift-logging -o jsonpath='{.spec.host}') \
     | oc -n openshift-logging apply -f -
```

> Check objects

Get a list of the objects created:

```
   oc get ConsoleExternalLogLink -l app=es-logging-dedalus --no-headers -n openshift-logging |cut -d' ' -f1
```

## ElasticSearch: create the index template

Create the default index template:

```bash
   . deploy/elasticsearch/index_explicit_mapping_template.sh
```

If the command is successful, will be returned the output:

```json
   {"acknowledged":true}
```

### Useful ElasticSearch commands

Getting the ES pod name as pre-requirements for the nexts commands:

```bash
   es_pod=$(oc -n openshift-logging get pods -l component=elasticsearch --no-headers | head -1 | cut -d" " -f1)
```

* Getting a specific Template:

```bash
   oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template/dedalus_es_template
```

* Delete Template:

```bash
   oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template/dedalus_es_template -XDELETE
```

* Getting All Templates:

```bash
   oc exec -n openshift-logging -c elasticsearch ${es_pod} -- es_util --query=_template | jq "[.]"
```
