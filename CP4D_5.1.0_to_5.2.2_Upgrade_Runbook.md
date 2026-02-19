## Des Jardins CP4D 5.1.0 to 5.2.2 Upgrade

## Author: Alex Kuan (alex.kuan@ibm.com)

From:

```
CPD: 5.1.0
OCP: 4.16
Storage: OpenShift Data Foundation
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,wml,analyticsengine,ws_pipelines
```

To:

```
CPD: 5.2.2
OCP: 4.16
Storage: OpenShift Data Foundation
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,wml,analyticsengine,ws_pipelines
```

# Table of Contents

1. [Overview](#1-overview)  
2. [Prerequisites](#2-prerequisites)  
   - 2.1 [Backups](#21-backups)  
   - 2.2 [Upgrade Clients](#22-upgrade-clients)  
   - 2.3 [OLM Utils Proxy Issue (TS020697962)](#23-olm-utils-proxy-issue-ts020697962)  
3. [Collect Required Information](#3-collect-required-information)  
   - 3.1 [List Deployed Components](#31-list-deployed-components)  
   - 3.2 [Check Scheduler Namespace](#32-check-scheduler-namespace)  
   - 3.3 [Update cpd_vars.sh](#33-update-cpd_varssh)  
4. [Upgrade Shared Cluster Components](#4-upgrade-shared-cluster-components)  
   - 4.1 [Login](#41-login)  
   - 4.2 [Upgrade Certificate Manager & License Service](#42-upgrade-certificate-manager--license-service)  
   - 4.3 [Upgrade Scheduler](#43-upgrade-scheduler)  
5. [Apply Entitlements](#5-apply-entitlements)  
6. [Cluster Health Checks](#6-cluster-health-checks)  
7. [Upgrade CPD Instance](#7-upgrade-cpd-instance)  
   - 7.1 [Setup Instance](#71-setup-instance)  
   - 7.2 [Upgrade Operators](#72-upgrade-operators)  
   - 7.3 [Upgrade Services (Batch)](#73-upgrade-services-batch)  
   - 7.4 [Validation](#74-validation)  
8. [RSI & CronJobs](#8-rsi--cronjobs)  
9. [Upgrade RSI Injection Controller](#9-upgrade-rsi-injection-controller)  
10. [Service-Specific Upgrades](#10-service-specific-upgrades)  
    - 10.1 [Spark (Analytics Engine)](#101-spark-analytics-engine)  
    - 10.2 [Data Virtualization (DV)](#102-data-virtualization-dv)  
11. [db2aaservice Removal (Post Migration)](#11-db2aaservice-removal-post-migration)  
12. [Known Issue – DMC (TS020862040)](#12-known-issue--dmc-ts020862040)  
13. [Final Validation](#13-final-validation)  

---

# 1. Overview

This runbook describes the upgrade procedure from IBM Software Hub / Cloud Pak for Data **5.1.0 to 5.2.2**.

---

# 2. Prerequisites

## 2.1 Backups

- CP4D Backup completed  
- DataStage backup is automatic  
- Validate IKC backup with Nicolas  
- For DataStage (WKS), export:
  - Projects  
  - Catalogs  
  - Terms  
  - Additional metadata as required

## 2.2 Upgrade Clients

Upgrade clients on **Edge node** and **Bastion**.

### Upgrade `cpd-cli`

```
cpd-cli version
```

### Restart OLM container (if updated)

```
cpd-cli manage restart-container
```

## 2.3 OLM Utils Proxy Issue (TS020697962)

Check:

```
podman exec -it olm-utils-play-v3 grep -n "self.no_proxy = None" /usr/local/lib/python3.12/site-packages/kubernetes/client/configuration.py
```

Remove line 173:

```
podman exec -it -u root olm-utils-play-v3 sed -i -e '173d' /usr/local/lib/python3.12/site-packages/kubernetes/client/configuration.py
```

Restart:

```
cpd-cli manage restart-container
```

---

# 3. Collect Required Information

## 3.1 List Deployed Components

```
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## 3.2 Check Scheduler Namespace

```
oc get scheduling -A
```

## 3.3 Update cpd_vars.sh

```
cd ~
cp cpd_vars.sh cpd_vars_v522.sh
vi cpd_vars_v522.sh
```

Set:

```
export VERSION=5.2.2
```

Switch symlink:

```
unlink cpd_vars.sh
ln -s cpd_vars_v522.sh cpd_vars.sh
source cpd_vars.sh
```

---

# 4. Upgrade Shared Cluster Components

## 4.1 Login

```
cpd-cli manage login-to-ocp \
  --username=${OCP_USERNAME} \
  --password=${OCP_PASSWORD} \
  --server=${OCP_URL}
```

Test:

```
cpd-cli manage oc get node
```

Check for any unhealthy or problematic pods:

```
oc get pods --all-namespaces -o wide |grep -Eiv '1/1|2/2|3/3|4/4|5/5|6/6|7/7|8/8|9/9|10/10|11/11|12/12|13/13|complete|download'
```

```
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

## 4.2 Upgrade Certificate Manager & License Service - Est. 3 minutes

Upgrade the License Service and the IBM Certificate manager, if it is installed
```
cpd-cli manage apply-cluster-components \
  --release=${VERSION} \
  --license_acceptance=true \
  --cert_manager_ns=${PROJECT_CERT_MANAGER} \
  --licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Environments without the IBM Certificate manager use
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Confirm that the License Service pods are Running or Completed
```
 oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

## 4.3 Upgrade Scheduler - Est. 7-8 minutes

If the scheduling service is installed, upgrade the scheduling service
```
cpd-cli manage apply-scheduler \
  --release=${VERSION} \
  --license_acceptance=true \
  --scheduler_ns=${PROJECT_SCHEDULING_SERVICE}
```

---

# 5. Apply Entitlements - Est. 1-2 minutes

DEV / CERT:

```
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=cpd-enterprise \
  --production=false
```

PROD:

```
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=cpd-enterprise
```

```
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=datastage
```

```
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=ikc-standard
```

---

# 6. Cluster Health Checks - Est. 1-2 minutes

```
cpd-cli health cluster
cpd-cli health nodes
cpd-cli health operands -c ${PROJECT_CPD_INST_OPERANDS}
cpd-cli health network-performance --image-prefix=${ARTIFACTORY_REGISTRY_LOCATION}/docker-icr-remote/cpopen/cpd
cpd-cli health network-connectivity --control_plane_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

# 7. Upgrade CPD Instance

## 7.1 Setup Instance - Est. 55-60 minutes

```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--run_storage_tests=false
```

## 7.2 Upgrade Operators - Est. 30 minutes

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true
```

## 7.3 Upgrade Services (Batch)

```
cpd-cli manage apply-cr \
  --release=${VERSION} \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=${COMPONENTS} \
  --block_storage_class=${STG_CLASS_BLOCK} \
  --file_storage_class=${STG_CLASS_FILE} \
  --license_acceptance=true \
  --upgrade=true
```

---

# 8. RSI & CronJobs

```
oc patch CronJob zen-rsi-evictor-cron-job \
  --namespace=${PROJECT_CPD_INST_OPERANDS} \
  --type=merge \
  --patch='{"spec":{"suspend": false}}'
```

---

# 9. Upgrade RSI Injection Controller

```
podman search --list-tags cp.icr.io/cpopen/cpd/zen-rsi-adm-controller
```

---

# 10. Service-Specific Upgrades

## 10.1 Spark

```
cpd-cli service-instance upgrade \
  --service-type=spark \
  --profile=${CPD_PROFILE_NAME} \
  --all
```

## 10.2 Data Virtualization

```
cpd-cli service-instance list \
  --service-type=dv \
  --profile=${CPD_PROFILE_NAME}
```

---

# 11. db2aaservice Removal

```
cpd-cli manage delete-cr \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=db2aaservice \
  --include_dependency=true
```

---

# 12. Known Issue – DMC

```
oc get csv -n cpd-operators ibm-databases-dmc.v5.9.0 -oyaml | grep -i image
```

---

# 13. Final Validation

```
oc get pods --all-namespaces -o wide |grep -Eiv '1/1|2/2|3/3|4/4|5/5|6/6|7/7|8/8|9/9|10/10|11/11|12/12|13/13|complete|download'
```

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

# Upgrade Complete

- All pods running  
- CR reconciled  
- Services validated  

- GUI confirms 5.2.2


