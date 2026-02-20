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

Before you upgrade IBM Software Hub, check for the whether the following pods are running in this instance of IBM Software Hub

Check whether the global search pods are running
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep elasticsea-0ac3
```

If the command returns an empty response, proceed to the next step

If the command returns a list of pods, review Upgrades fail when global search is configured incorrectly to determine whether you have any configurations that could cause issues during upgrade

Check whether the catalog-api pods are running
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep catalog-api
```

If the command returns an empty response, you are ready to upgrade IBM Software Hub

If the command returns a list of pods, review the following guidance to determine how long the catalog-api service will be down during upgrade

When you upgrade the common core services to IBM Software Hub Version 5.2, the underlying storage for the catalog-api service is migrated to PostgreSQL.

During the final stages of the migration, the catalog-api service is offline, and services that are dependent on the service are not available

The duration of the migration depends on the number of assets and relationships that are stored in the instance

The duration of the outage depends on the number of databases (projects, catalogs, and spaces) in the instance

In a typical upgrade scenario, the outage should be significantly shorter than the overall migration

To determine how many databases will be migrated

Set the INSTANCE_URL environment variable to the URL of IBM Software Hub
```
export INSTANCE_URL=https://cpd-cpd-instance.apps.desjardins.cp.fyre.ibm.com
```

Get the credentials for the wdp-service
```
TOKEN=$(oc get -n ${PROJECT_CPD_INST_OPERANDS} secrets wdp-service-id -o yaml | grep service-id-credentials | cut -d':' -f2- | sed -e 's/ //g' | base64 -d)
```

Get the number of catalogs in the instance
```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=catalogs&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Get the number of projects in the instance
```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=projects&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Get the number of spaces in the instance
```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=spaces&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Add up the number of catalogs, projects, and spaces returned by the previous commands

Then, use the following table to determine approximately how long the service will be offline during the migration
```
Databases
Downtime for migration (approximate)
Up to 1,000 databases	6 minutes
1,001 - 10,000 databases	20 minutes
10,001 - 70,000 databases	60 minutes
```

Save the following script on the client workstation as a file named precheck_migration.sh
```
#!/bin/bash

# Default ranges for couchdb size
SMALL=50
MEDIUM=100
LARGE=200

echo "Performing pre-migration checks"

patch_for_small()
{
  echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 4,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "2", "ephemeral-storage": "10Mi", "memory": "2Gi"},
    "limits": {"cpu": "6", "ephemeral-storage": "1Gi", "memory": "6Gi"}}
}}'
EOF
    echo
    echo "The system is ready for migration. Upgrade your cluster as usual."
}

patch_for_medium()
{
    echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 6,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "3", "ephemeral-storage": "10Mi", "memory": "4Gi"},
    "limits": {"cpu": "8", "ephemeral-storage": "4Gi", "memory": "8Gi"}}
}}'
EOF
    echo
    echo "The system is ready for migration. Upgrade your cluster as usual."
}

patch_for_large()
{
    echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 8,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "6", "ephemeral-storage": "10Mi", "memory": "6Gi"},
    "limits": {"cpu": "10", "ephemeral-storage": "6Gi", "memory": "10Gi"}}
}}'
EOF
    echo
    echo "Before you can start the upgrade, you must prepare the system for migration."
}

check_resources(){
        scale_config=$1
        pvc_size=$(oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} database-storage-wdp-couchdb-0 --no-headers | awk '{print $4}')
        size=$(awk '{print substr($0, 1, length($0)-2)}' <<< "$pvc_size")

        if [[ $scale_config == "small" ]];then
          if [[ "$size" -le "$SMALL" ]];then
            echo "The system is ready for migration. Upgrade your cluster as usual."
          elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$MEDIUM" ];then
            patch_for_medium
          elif [ "$size" -ge "$MEDIUM" ] && [ "$size" -le "$LARGE" ];then
            patch_for_large
          else
            patch_for_large
          fi
        elif [[ $scale_config == "medium" ]];then
          if [[ "$size" -le "$SMALL" ]];then
            patch_for_small
          elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$MEDIUM" ];then
            echo "The system is ready for migration. Upgrade your cluster as usual."
          elif [ "$size" -ge "$MEDIUM" ] && [ "$size" -le "$LARGE" ];then
            patch_for_large
          else
            patch_for_large
          fi
        elif [[ $scale_config == "large" ]];then
          if [[ "$size" -le "$SMALL" ]];then
            patch_for_small
          elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$MEDIUM" ];then
            patch_for_medium
          elif [ "$size" -ge "$MEDIUM" ] && [ "$size" -le "$LARGE" ];then
            echo "The system is ready for migration. Upgrade your cluster as usual."
          else
            patch_for_large
          fi
        fi
}

check_upgrade_case(){     
        echo -e "Checking if automatic upgrade or semi-automatic upgrade is needed"
        scale_config=$(oc get ccs -n ${PROJECT_CPD_INST_OPERANDS} ccs-cr -o json | jq -r '.spec.scaleConfig')

        # Default case, scale config is set to small
        if [[ -z "${scale_config}" ]];then
          scale_config=small
        fi

        check_resources $scale_config
}

check_upgrade_case
```

Run the precheck_migration.sh to determine whether you can run an automatic migration of the common core services or whether you need to configure common core services to run a semi-automatic migration
```
./precheck_migration.sh 
```

Example output
```
Performing pre-migration checks
Checking if automatic upgrade or semi-automatic upgrade is needed
The system is ready for migration. Upgrade your cluster as usual
```

Take the appropriate action based on the message returned by the script
```
The system is ready for migration
Upgrade your cluster as usual
Automatic
You are ready to upgrade IBM Software Hub
After you upgrade the services in your environment, ensure that you complete the catalog-api service migration
```

```
Run the following command to increase the CPU and memory.
Automatic 
Run the patch command returned by the script.
Upgrade IBM Software Hub
After you upgrade the services in your environment, ensure that you complete the catalog-api service migration
```

```
The script returns both of the following messages
Run the following command to increase the CPU and memory
Before you can start the upgrade, you must prepare the system for migration
Semi-automatic
Run the patch command returned by the script
Run the following command to enable semi-automatic migration

oc patch ccs ccs-cr \
-n ${PROJECT_CPD_INST_OPERANDS} \
--type merge \
--patch '{"spec": {"use_semi_auto_catalog_api_migration": true}}'

Upgrade IBM Software Hub
After you upgrade the services in your environment, ensure that you complete the catalog-api service migration
```

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

## 7.2 Upgrade Operators - Est. 25-30 minutes

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true
```

## 7.3 Upgrade Services - ws,wml,analyticsengine,ws_pipelines

Upgrade Watson Studio and CCS - Est. 65-70 minutes
```
cpd-cli manage apply-cr \
--components=ws \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Upgrade Watson Machine Learning - Est. 43 minutes
```
cpd-cli manage apply-cr \
--components=wml \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Upgrade Analytics Engine - Est. 17-20 minutes
```
cpd-cli manage apply-cr \
--components=analyticsengine \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true \
--upgrade=true
```

Upgrade Pipelines - Est. 12-13 minutes
```
cpd-cli manage apply-cr \
--components=ws_pipelines \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

---

# 8. [Completing the catalog-api service migration](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=services-completing-catalog-api-migration)

After you upgrade the common core services to IBM® Software Hub Version 5.2, the back-end database for the catalog-api service is migrated from CouchDB to PostgreSQL

If you ran an automatic migration, the common core services waits for the migration jobs to complete before upgrading the components associated with the common core services

If you ran a semi-automatic migration, the common core services runs the migration jobs while upgrading the components associated with the common core services

Run the following command to determine which migration method was used
```
oc describe ccs ccs-cr \
--namespace ${PROJECT_CPD_INST_OPERANDS} \
| grep use_semi_auto_catalog_api_migration
```

The command returns an empty response
```
Automatic -> Proceed to 4. Collecting statistics about the migration
```

The command returns true
```
Semi-automatic -> 2. Checking the status of the migration jobs
```

4. Collecting statistis about the migration

Save the following script on the client workstation as a file named migration_status.sh
```
#!/bin/bash

# Set postgres connection parameters
postgres_password=$(oc get secret -n ${PROJECT_CPD_INST_OPERANDS} ccs-cams-postgres-app -o json 2>/dev/null | jq -r '.data."password"' | base64 -d)
postgres_username=cams_user
postgres_db=camsdb
postgres_migrationdb=camsdb_migration

echo -e "======MIGRATION STATUS==========="

# Total migrated database(s)
databases=$(oc -n ${PROJECT_CPD_INST_OPERANDS} -c postgres exec ccs-cams-postgres-1 -- psql -t postgresql://$postgres_username:$postgres_password@localhost:5432/$postgres_migrationdb -c "select count(*) from migration.status where state='complete'" 2>/dev/null)
if [ -n "$databases" ];then
  databases_no_space=$(echo "$databases" | tr -d ' ')
  echo "Total catalog-api databases migrated: $databases_no_space"
else
  echo "Unable to fetch migration information for databases"
fi

# Total migrated assets
assets=$(oc -n ${PROJECT_CPD_INST_OPERANDS} -c postgres exec ccs-cams-postgres-1 -- psql -t postgresql://$postgres_username:$postgres_password@localhost:5432/$postgres_db -c "select count(*) from cams.asset" 2>/dev/null)
if [ -n "$assets" ];then
  assets_no_space=$(echo "$assets" | tr -d ' ')
  echo -e "Total catalog-api assets migrated: $assets_no_space\n"
else
  echo "Unable to fetch migration information for assets"
fi
```

Run the migration_status.sh script
```
./migration_status.sh
```

The following is an example of the output
```
======MIGRATION STATUS===========
Total catalog-api databases migrated: 6
Total catalog-api assets migrated: 0
```

Proceed with 5. Backing up the PostgreSQL database

5. Backing up the PostgreSQL database

Save the following script on the client workstation as a file named backup_postgres.sh
```
#!/bin/bash

# Make sure PROJECT_CPD_INST_OPERANDS is set
if [ -z "$PROJECT_CPD_INST_OPERANDS" ]; then
  echo "Environment variable PROJECT_CPD_INST_OPERANDS is not defined. This environment variable must be set to the project where IBM Software Hub is running."
  exit 1
fi

echo "PROJECT_CPD_INST_OPERANDS namespace is: $PROJECT_CPD_INST_OPERANDS"

# Step 1: Find the replica pod
REPLICA_POD=$(oc get pods -n $PROJECT_CPD_INST_OPERANDS -l app=ccs-cams-postgres -o jsonpath='{range .items[?(@.metadata.labels.role=="replica")]}{.metadata.name}{"\n"}{end}')

if [ -z "$REPLICA_POD" ]; then
  echo "No replica pod found."
  exit 1
fi

echo "Replica pod: $REPLICA_POD"

# Step 2: Extract JDBC URI from a secret
JDBC_URI=$(oc get secret ccs-cams-postgres-app -n $PROJECT_CPD_INST_OPERANDS -o jsonpath="{.data.uri}" | base64 -d)

if [ -z "$JDBC_URI" ]; then
  echo "JDBC URI not found in secret."
  exit 1
fi

#  Set path on the pod to save the dump file 
TARGET_PATH="/var/lib/postgresql/data/forpgdump"

# Step 3: Run pg_dump with nohup inside the pod
oc exec "$REPLICA_POD" -n $PROJECT_CPD_INST_OPERANDS -- bash -c "
  TARGET_PATH=\"$TARGET_PATH\"
  JDBC_URI=\"$JDBC_URI\"
  echo \"TARGET_PATH is $TARGET_PATH\"
  mkdir -p $TARGET_PATH &&
  chmod 777 $TARGET_PATH &&
  nohup bash -c '
    pg_dump $JDBC_URI -Fc -f $TARGET_PATH/cams_backup.dump > $TARGET_PATH/pgdump.log 2>&1 &&
    echo \"Backup succeeded. Please copy $TARGET_PATH/cams_backup.dump file from this pod to a safe place and delete it on this pod to save space.\" >> $TARGET_PATH/pgdump.log
  ' &
  echo \"pg_dump started in background. Logs: $TARGET_PATH/pgdump.log\"
"
```

Run the backup_postgres.sh script
```
./backup_postgres.sh
```

The following is an example of the output
```
PROJECT_CPD_INST_OPERANDS namespace is: cpd-instance
Replica pod: ccs-cams-postgres-2
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)
TARGET_PATH is /var/lib/postgresql/data/forpgdump
pg_dump started in background. Logs: /var/lib/postgresql/data/forpgdump/pgdump.log
```

The script starts the backup in a separate terminal session

Set the REPLICA_POD environment variable
```
REPLICA_POD=$(oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -l app=ccs-cams-postgres -o jsonpath='{range .items[?(@.metadata.labels.role=="replica")]}{.metadata.name}{"\n"}{end}')
```

Confirm the REPLICA_POD variable is set correctly
```
echo $REPLICA_POD
ccs-cams-postgres-2
```

Open a remote shell in the replica pod
```
oc rsh ${REPLICA_POD}
```

Change to the /var/lib/postgresql/data/forpgdump/ directory
```
cd /var/lib/postgresql/data/forpgdump/
```

Run the following command to monitor the list of files in the directory
```
ls -lat
```

```
total 884
-rw-r--r--. 1 1000790000 1000790000    158 Feb 20 02:36 pgdump.log
-rw-r--r--. 1 1000790000 1000790000 890308 Feb 20 02:36 cams_backup.dump
drwxrwsrwx. 2 1000790000 1000790000   4096 Feb 20 02:36 .
drwxrwsr-x. 5 root       1000790000   4096 Feb 20 02:36 ..
```

Wait for the backup to complete. (This process can take several hours if the database is large)
```
Backup Phase	What to Look For
In progress - During the backup, the size of the pgdump.log file increases.

Complete - The backup is complete when the script writes the following message to the pgdump.log file:
“Backup succeeded. Please copy /var/lib/postgresql/data/forpgdump/cams_backup.dump file from this pod to a safe place and delete it on this pod to save space.”

Failed - If the backup fails, the pgdump.log file will include error messages.
If the backup fails, contact IBM Support and append the pgdump.log file to your support case.
```

Confirm the contents of the pgdump.log file
```
cat pgdump.log

Backup succeeded. Please copy /var/lib/postgresql/data/forpgdump/cams_backup.dump file from this pod to a safe place and delete it on this pod to save space.
```

Do not proceed to the next step unless the backup is complete

Set the POSTGRES_BACKUP_STORAGE_LOCATION environment variable to the location where you want to store the backup
```
export POSTGRES_BACKUP_STORAGE_LOCATION=<your-pgbackup-location>
chmod 777 <your-pgbackup-location>
```

Copy the backup to the POSTGRES_BACKUP_STORAGE_LOCATION
```
oc cp ${REPLICA_POD}:/var/lib/postgresql/data/forpgdump/cams_backup.dump $POSTGRES_BACKUP_STORAGE_LOCATION/cams_backup.dump
```

Output from the copy command for reference
```
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)
tar: Removing leading `/' from member names
```

Confirm the copy was successful
```
pgbackup]# ll
total 872
-rw-r--r-- 1 root root 890308 Feb 19 18:51 cams_backup.dump
```

Delete the backup from the replica pod
```
oc rsh $REPLICA_POD rm -f /var/lib/postgresql/data/forpgdump/cams_backup.dump
```

Output from the delete command for reference
```
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)
```

Proceed to 6. Consolidating the PostgreSQL database

6. Consolidating the PostgreSQL database

After you back up the new PostgreSQL database, you must consolidate all of the existing copies of identical data across governed catalogs into a single record so that all identical data assets share a set of common properties

Set the INSTANCE_URL environment variable to the URL of IBM Software Hub
```
export INSTANCE_URL=https://cpd-cpd-instance.apps.desjardins.cp.fyre.ibm.com
```

Get the name of a catalog-api-jobs pod
```
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep catalog-api-jobs
```

For example
```
catalog-api-jobs-7949486ffc-bntzv
```

Set the CAT_API_JOBS_POD environment variable to the name a pod returned by the preceding command
```
export CAT_API_JOBS_POD=catalog-api-jobs-7949486ffc-bntzv
```

Open a Bash prompt in the pod
```
oc exec ${CAT_API_JOBS_POD} -n ${PROJECT_CPD_INST_OPERANDS} -it -- bash 
```

Run the following command to set the AUTH_TOKEN environment variable
```
AUTH_TOKEN=$(cat /etc/.secrets/wkc/service_id_credential)
```

Start the consolidation
```
curl -k -X PUT "${INSTANCE_URL}/v2/shared_assets/initialize_content?bss_account_id=999" -H "Authorization: Basic $AUTH_TOKEN"
```

The command returns a transaction ID
```
{"transaction_id":"8itd5ruv4bujzcfna3rnf04zw"}
```

Get the name of a catalog-api pod
```
oc get pods \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep catalog-api \
| grep -v catalog-api-jobs
```

For example
```
catalog-api-67b59996c4-stttn
```

Set the CAT_API_POD environment variable to the name a pod returned by the preceding command
```
export CAT_API_POD=catalog-api-67b59996c4-stttn
```

Check the catalog-api pod logs to determine the status of the consolidation

Check for the following success message
```
oc logs ${CAT_API_POD} \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep "Initial consolidation for bss account 999 complete"
```

If the command returns a response the consolidation was successful, proceed to What to do if the consolidation completed successfully

If the command returns an empty response, proceed to the next step

Check for the following failure message
```
oc logs ${CAT_API_POD} \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep "Error running initial consolidation with resource key"
```

If the command returns a response the consolidation failed, try to consolidate the database again. If the problem persists, contact IBM Support

If the command returns an empty response, proceed to the next step

Check for the following failure message
```
oc logs ${CAT_API_POD} \
-n ${PROJECT_CPD_INST_OPERANDS} \
| grep "Error executing initial consolidation for bss 999"
```

If the command returns a response the consolidation failed, try to consolidate the database again. If the problem persists, contact IBM Support

If the command returns an empty response, proceed to the next step

If the preceding commands returned empty responses, wait 10 minutes before checking the pod logs again

What to do if the consolidation completed successfully

If the PostgreSQL database consolidation was successful, wait several weeks to confirm that the projects, catalogs, and spaces in your environment are working as expected

After you confirm that the projects, catalogs, and spaces are working as expected, run the following commands to clean up the migration resources

Delete the pods associated with the migration
```
oc delete pod \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the jobs associated with the migration
```
oc delete job \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the config maps associated with the migration
```
oc delete cm \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```


Delete the secrets associated with the migration
```
oc delete secret \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l app=cams-postgres-migration-app
```

Delete the persistent volume claim associated with the migration
```
oc delete pvc cams-postgres-migration-pvc \
-n ${PROJECT_CPD_INST_OPERANDS}
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

Before you begin, [create a profile on the workstation from which you will upgrade the service instances](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cli-creating-cpd-profile)

Set profile environment variables
```
export API_KEY=TgVmHJAaeKXz7qAyZ0i2QfqqVDQyaeuxJ9u178Xf
export CPD_USERNAME=cpadmin
export LOCAL_USER=cpadmin
export CPD_PROFILE_NAME=cpadmin
export CPD_PROFILE_URL=https://cpd-cpd-instance.apps.desjardins.cp.fyre.ibm.com
```

Create the local user configuration
```
cpd-cli config users set ${LOCAL_USER} \
--username ${CPD_USERNAME} \
--apikey ${API_KEY}
```

Create the profile
```
cpd-cli config profiles set ${CPD_PROFILE_NAME} \
--user ${LOCAL_USER} \
--url ${CPD_PROFILE_URL}
```

Get a list of all service instances using your profile
```
cpd-cli service-instance list \
--profile=${CPD_PROFILE_NAME}
```

This is an example output for reference
```
Namespace           Service type        Version             ID                  Name                               Provision status    Upgrade version option 
 ---------           ------------        -------             --                  ----                               ----------------    ---------------------- 
 cpd-instance        volumes             -                   1771543053516863    cpd-instance::datarefinerylibvol   PROVISIONED         [] 
 cpd-instance        volumes             -                   1771448989203918    cpd-instance::ws-runtimes-libs     PROVISIONED         [] 
```

## 10.1 Update Watson Studio Runtimes



## 10.2 Update Spark Service Instances
```
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
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

