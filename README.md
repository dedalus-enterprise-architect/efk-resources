# Openshift Logging Resources

This project collects some procedures on how to Set Up a custom EFK instance having the following minimum requirements:

 * Cluster Logging Operator - community edition starting from version 5.7.0

 * Elasticsearch Operator - community edition starting from version 5.7.0
 
 * Openshift 4.12 or higher

References:
  - https://github.com/openshift/cluster-logging-operator
  - https://github.com/openshift/elasticsearch-operator

## Openshift Cluster Logging: Overview

This project focus on the following topics:

    * logging persistent storage
    * custom indexes template in order to avoiding the field's map explosion
    * improving the indexes retention
    * kibana custom structured field view

Explore the files used by this project:

* ```deploy/clusterlogging/cl-forwarder.yml``` : this is the __cluster forwarder instance__ definition

* ```deploy/clusterlogging/cl-instance.template.yml``` : this is the __cluster logging instance__ definition

* ```deploy/clusterlogging/cl-operator.yml``` : this template aims is installing of the Openshift Cluster Logging Operator stack

* ```deploy/elasticsearch/es-operator.yml``` : this is the subscription object which instanciate the Redhat Elasticsearch Operator

* ```deploy/elasticsearch/index_explicit_mapping_template.sh``` : this script creates a custom index template on ElasticSearch

* ```deploy/kibana/kibana-externallink.template.yml``` : this template creates a new kibana link aimed to have a custom fields view available as default

### Project minimium requirements

* The Openshift client utility: ```oc```

* A cluster admin roles rights

* Git project local clone

### RedHat Elasticsearch Operator: setup

> WARNING: an Admin Cluster Role is required to proceed on this section.

Run the following command to install the RedHat Elasticsearch Operator:

```
   oc apply -f deploy/elasticsearch/es-operator.yml
```

> Check Objects

Get a list of the objects created:

```
   oc get all,ConfigMap,Secret,Elasticsearch,OperatorGroup,Subscription -l app=es-logging-dedalus --no-headers -n openshift-operators-redhat |cut -d' ' -f1
```

### RedHat Cluster Logging Operator: setup

> WARNING: an Admin Cluster Role is required to proceed on this section.

Run the following command to install the RedHat Openshift Logging Operator:

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

> Check Objects

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

> Check Objects

Get a list of the objects created:

```
   oc get ConsoleExternalLogLink -l app=es-logging-dedalus --no-headers -n openshift-logging |cut -d' ' -f1
```

## Elasticsearch: Create the index template

Create the default index template:

```bash
. deploy/elasticsearch/index_explicit_mapping_template.sh
```

If the command is successful, the output will be:

```json
{"acknowledged":true}
```

### Useful ES commands

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