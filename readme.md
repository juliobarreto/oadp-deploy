# Configuring OADP with Noobaa ODF
First let's create an app to perform the deployment, this app has the objective of showing a home page to validate the access.

    oc new-project fastapi
    oc new-app --name fastapi --labels app=fastapi https://github.com/laurobmb/fastapi.git#master --context-dir app --strategy=source
    oc expose service fastapi

![InitialPage](/photos/initialPage.png)

## Criando ObjectBucketClaim
    oc apply -f obc.yml 

## Extraindo secret from bucket
    oc extract secret/backup-noobaa --confirm 

## create file credential 
    cat << EOF > ./credentials-velero
    [default]
    aws_access_key_id=$(cat AWS_ACCESS_KEY_ID)
    aws_secret_access_key=$(cat AWS_SECRET_ACCESS_KEY)
    EOF

## Create secret for ADP
    oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=credentials-velero

## Get route s3
    oc -n openshift-storage get route s3 -o jsonpath='{.spec.host}{"\n"}'

## Get name from OBC
    oc -n openshift-adp get obc backup-noobaa  -o jsonpath='{.spec.bucketName}{"\n"}'
    oc -n openshift-adp get obc backup-noobaa  -o jsonpath='{.spec.generateBucketName}{"\n"}'

## Installing the OADP Operator

You install the OpenShift API for Data Protection (OADP) Operator on OpenShift Container Platform 4.9 by using Operator Lifecycle Manager (OLM). The OADP Operator installs Velero 1.7.

### Prerequisites
You must be logged in as a user with cluster-admin privileges.
### Procedure
1. In the OpenShift Container Platform web console, click **Operators → OperatorHub**.
2. Use the **Filter by keyword** field to find the **OADP Operator**.
3. Select the **OADP Operator** and click **Install**.
4. Click **Install** to install the Operator in the openshift-adp project.
5. Click **Operators → Installed Operators** to verify the installation.

![Install Operator](/photos/oadp-operator.png)

## Edite ADP
With the information collected before, edit the file

    vim dataProtectionApplication.yml

## Verifify
    oc -n openshift-adp get pods
    oc -n openshift-adp get events
    oc -n openshift-adp get all

### Result 

    NAME                                                    READY   STATUS    RESTARTS   AGE
    pod/oadp-my-config-1-aws-registry-7f98fc598f-bwwks      1/1     Running   0          34m
    pod/openshift-adp-controller-manager-5cdb6b7bd6-sdjs7   1/1     Running   0          50m
    pod/velero-677db77949-f4rdf                             1/1     Running   0          34m

    NAME                                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    service/oadp-my-config-1-aws-registry-svc                  ClusterIP   172.50.68.59    <none>        5000/TCP   34m
    service/openshift-adp-controller-manager-metrics-service   ClusterIP   172.50.105.21   <none>        8443/TCP   50m
    service/openshift-adp-velero-metrics-svc                   ClusterIP   172.50.53.108   <none>        8085/TCP   34m

    NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/oadp-my-config-1-aws-registry      1/1     1            1           34m
    deployment.apps/openshift-adp-controller-manager   1/1     1            1           50m
    deployment.apps/velero                             1/1     1            1           34m

    NAME                                                          DESIRED   CURRENT   READY   AGE
    replicaset.apps/oadp-my-config-1-aws-registry-7f98fc598f      1         1         1       34m
    replicaset.apps/openshift-adp-controller-manager-5cdb6b7bd6   1         1         1       50m
    replicaset.apps/velero-677db77949                             1         1         1       34m

    NAME                                                           HOST/PORT                                                                     PATH   SERVICES                            PORT    TERMINATION   WILDCARD
    route.route.openshift.io/oadp-my-config-1-aws-registry-route   oadp-my-config-1-aws-registry-route-openshift-adp.apps.lagomes.rhbr-lab.com          oadp-my-config-1-aws-registry-svc   <all>                 None


## get name of dataProtectionApplication

Now, before running the backup, the dataProtectionApplication creates a storage location inside the operator and you must edit the backup and put it in the configuration

    oc get BackupStorageLocation -o jsonpath='{.items[].metadata.name}{"\n"}'

Edit the ***storageLocation:*** line in the backup-fastapi.yml file with the collector value in the previous command

## Create backup
    oc apply -f backup-fastapi.yml

## check backup
    oc get Backup fastapi-backup -o jsonpath='{.status.phase}{"\n"}'
    oc get Backup fastapi-backup -o jsonpath='{.status.progress}{"\n"}'

![Result ceph](/photos/bucket-noobaa.png)

# Test of recovery
The test consists of removing the project from the cluster OCP and after destroying it, let's repair the project using the bucket backup

    oc delete project fastapi

## Create restore
The ***backupName:***, in the file restore-fastapi.yaml, should point to the name of the backup created before this step

    oc apply -f restore-fastapi.yaml

    oc get Restore fastapi-restore -o jsonpath='{.status.phase}{"\n"}'
    oc get Restore fastapi-restore -o jsonpath='{.status.progress}{"\n"}'


# Reference links

https://docs.openshift.com/container-platform/4.9/backup_and_restore/application_backup_and_restore/installing/installing-oadp-mcg.html

https://docs.openshift.com/container-platform/4.8/backup_and_restore/application_backup_and_restore/installing/installing-oadp-mcg.html

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10/html-single/backup_and_restore/index#installing-and-configuring-oadp

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.9/html-single/backup_and_restore/index#installing-oadp-mcg

https://github.com/openshift/oadp-operator

https://www.youtube.com/watch?v=OcIZ3SfHyFY&t=2451s&pp=ugMICgJwdBABGAE%3D
