---
title: "Disaster Recovery and Cloning"
date:
draft: false
weight: 85
---

Perhaps someone accidentally dropped the `users` table. Perhaps you want to clone your production database to a step-down environment. Perhaps you want to exercise your disaster recovery system (and it is important that you do!).

Regardless of scenario, it's important to know how you can perform a "restore" operation with PGO to be able to recovery your data from a particular point in time, or clone a database for other purposes.

Let's look at how we can perform different types of restore operations. First, let's understand the core restore properties on the custom resource.

## Restore Properties

There are several attributes on the custom resource that are important to understand as part of the restore process. All of these attributes are grouped together in the `spec.dataSource.postgresCluster` section of the custom resource.

Please review the table below to understand how each of these attributes work in the context of setting up a restore operation.

- `spec.dataSource.postgresCluster.clusterName`: The name of the cluster that you are restoring from. This corresponds to the `metadata.name` attribute on a different `postgrescluster` custom resource.
- `spec.dataSource.postgresCluster.clusterNamespace`: The namespace of the cluster that you are restoring from. Used when the cluster exists in a different namespace.
- `spec.dataSource.postgresCluster.repoName`: The name of the pgBackRest repository from the `spec.dataSource.postgresCluster.clusterName` to use for the restore. Can be one of `repo1`, `repo2`, `repo3`, or `repo4`. The repository must exist in the other cluster.
- `spec.dataSource.postgresCluster.options`: Any additional [pgBackRest restore options](https://pgbackrest.org/command.html#command-restore) or general options you would like to pass in. For example, you may want to set `--process-max` to help improve performance on larger databases.
- `spec.dataSource.postgresCluster.resources`: Setting [resource limits and requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits) of the restore job can ensure that it runs efficiently.
- `spec.dataSource.postgresCluster.affinity`: Custom [Kubernetes affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) rules constrain the restore job so that it only runs on certain nodes.
- `spec.dataSource.postgresCluster.tolerations`: Custom [Kubernetes tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) allow the restore job to run on [tainted](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) nodes.

Let's walk through some examples for how we can clone and restore our databases.

## Clone a Postgres Cluster

Let's create a clone of our [`hippo`]({{< relref "./create-cluster.md" >}}) cluster that we created previously. We know that our cluster is named `hippo` (based on its `metadata.name`) and that we only have a single backup repository called `repo1`.

Let's call our new cluster `elephant`. We can create a clone of the `hippo` cluster using a manifest like this:

```
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: elephant
spec:
  dataSource:
    postgresCluster:
      clusterName: hippo
      repoName: repo1
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:centos8-13.4-1
  postgresVersion: 13
  instances:
    - dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 1Gi
  backups:
    pgbackrest:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:centos8-2.35-0
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 1Gi
```

Note this section of the spec:

```
spec:
  dataSource:
    postgresCluster:
      clusterName: hippo
      repoName: repo1
```

This is the part that tells PGO to create the `elephant` cluster as an independent copy of the `hippo` cluster.

The above is all you need to do to clone a Postgres cluster! PGO will work on creating a copy of your data on a new persistent volume claim (PVC) and work on initializing your cluster to spec. Easy!

## Perform a Point-in-time-Recovery (PITR)

Did someone drop the user table? You may want to perform a point-in-time-recovery (PITR) to revert your database back to a state before a change occurred. Fortunately, PGO can help you do that.

You can set up a PITR using the [restore](https://pgbackrest.org/command.html#command-restore) command of [pgBackRest](https://www.pgbackrest.org), the backup management tool that powers the disaster recovery capabilities of PGO. You will need to set a few options on `spec.dataSource.postgresCluster.options` to perform a PITR. These options include:

- `--type=time`: This tells pgBackRest to perform a PITR.
- `--target`: Where to perform the PITR to. Any example recovery target is `2021-06-09 14:15:11-04`.  The timezone specified here as -04 for EDT.  Please see the [pgBackRest documentation for other timezone options](https://pgbackrest.org/user-guide.html#pitr).
- `--set` (optional): Choose which backup to start the PITR from.

A few quick notes before we begin:

- To perform a PITR, you must have a backup that is older than your PITR time. In other words, you can't perform a PITR back to a time where you do not have a backup!
- All relevant WAL files must be successfully pushed for the restore to complete correctly.
- Be sure to select the correct repository name containing the desired backup!


With that in mind, let's use the `elephant` example above. Let's say we want to perform a point-in-time-recovery (PITR) to `2021-06-09 14:15:11-04`, we can use the following manifest:

```
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: elephant
spec:
  dataSource:
    postgresCluster:
      clusterName: hippo
      repoName: repo1
      options:
      - --type=time
      - --target="2021-06-09 14:15:11-04"
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:centos8-13.4-1
  postgresVersion: 13
  instances:
    - dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 1Gi
  backups:
    pgbackrest:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:centos8-2.35-0
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 1Gi
```

The section to pay attention to is this:

```
spec:
  dataSource:
    postgresCluster:
      clusterName: hippo
      repoName: repo1
      options: --type=time --target="2021-06-09 14:15:11-04"
```

Notice how we put in the options to specify where to make the PITR.

Using the above manifest, PGO will go ahead and create a new Postgres cluster that recovers its data up until `2021-06-09 14:15:11-04`. At that point, the cluster is promoted and you can start accessing your database from that specific point in time!

## Perform an In-Place Point-in-time-Recovery (PITR)

Similar to the PITR restore described above, you may want to perform a similar reversion back to a state before a change occurred, but without creating another PostgreSQL cluster. Fortunately, PGO can help you do this as well.

You can set up a PITR using the [restore](https://pgbackrest.org/command.html#command-restore) command of [pgBackRest](https://www.pgbackrest.org), the backup management tool that powers the disaster recovery capabilities of PGO. You will need to set a few options on `spec.dataSource.postgresCluster.options` to perform a PITR. These options include:

- `--type=time`: This tells pgBackRest to perform a PITR.
- `--target`: Where to perform the PITR to. Any example recovery target is `2021-06-09 14:15:11-04`.
- `--set` (optional): Choose which backup to start the PITR from.

A few quick notes before we begin:

- To perform a PITR, you must have a backup that is older than your PITR time. In other words, you can't perform a PITR back to a time where you do not have a backup!
- All relevant WAL files must be successfully pushed for the restore to complete correctly.
- Be sure to select the correct repository name containing the desired backup!

To perform an in-place restore, users will first fill out the restore section of the spec as follows:

```
spec:
  backups:
    pgbackrest:
      restore:
        enabled: true
        repoName: repo1
        options:
        - --type=time
        - --target="2021-06-09 14:15:11-04"
```

And to trigger the restore, you will then annotate the PostgresCluster as follows:

```
kubectl annotate postgrescluster hippo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-restore=id1
```

And once the restore is complete, in-place restores can be disabled:

```
spec:
  backups:
    pgbackrest:
      restore:
        enabled: false
```

Notice how we put in the options to specify where to make the PITR.

Using the above manifest, PGO will go ahead and re-create your Postgres cluster that will recover its data up until `2021-06-09 14:15:11-04`. At that point, the cluster is promoted and you can start accessing your database from that specific point in time!


## Standby Cluster

Advanced high-availability and disaster recovery strategies involve spreading
your database clusters across multiple data centers to help maximize uptime.
In Kubernetes, this technique is known as "[federation](https://en.wikipedia.org/wiki/Federation_(information_technology))".
Federated Kubernetes clusters are able to communicate with each other,
coordinate changes, and provide resiliency for applications that have high
uptime requirements.

As of this writing, federation in Kubernetes is still in ongoing development.
In the meantime, PGO provides a way to deploy Postgres clusters that can span
multiple Kubernetes clusters using an external storage system:

- Amazon S3, or a system that uses the S3 protocol,
- Azure Blob Storage, or
- Google Cloud Storage

Standby Postgres clusters are managed just like any other Postgres cluster in PGO.
For example, adding replicas to a standby cluster is a matter of increasing the
`spec.instances.replicas` value. The main difference is that PostgreSQL data in
the cluster is read-only: one PostgreSQL instance is reading in the database
changes from an external repository while the other instances are replicas of it.
This is known as [cascading replication](https://www.postgresql.org/docs/current/warm-standby.html#CASCADING-REPLICATION).

The following manifest defines a Postgres cluster that recovers from WAL files
stored in an S3 bucket:

```
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: hippo-standby
spec:
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:centos8-13.4-1
  postgresVersion: 13
  instances:
    - dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 1Gi
  backups:
    pgbackrest:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:centos8-2.35-0
      repos:
      - name: repo1
        s3:
          bucket: "my-bucket"
          endpoint: "s3.ca-central-1.amazonaws.com"
          region: "ca-central-1"
  standby:
    enabled: true
    repoName: repo1
```

There comes a time where a standby cluster needs to be promoted to an active
cluster. Promoting a standby cluster means that a PostgreSQL instance within
it will start accepting both reads and writes. This has the net effect of
pushing WAL (transaction archives) to the pgBackRest repository, so we need to
take a few steps first to ensure we don't accidentally create a split-brain scenario.

First, if this is not a disaster scenario, you will want to "shutdown" the
active PostgreSQL cluster. This can be done with the `spec.shutdown` attribute:

```
spec:
  shutdown: true
```

The effect of this is that all the Kubernetes workloads for this cluster are
scaled to 0. You can verify this with the following command:

```
kubectl get deploy,sts,cronjob --selector=postgres-operator.crunchydata.com/cluster=hippo

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hippo-pgbouncer   0/0     0            0           1h

NAME                             READY   AGE
statefulset.apps/hippo-00-lwgx   0/0     1h

NAME                                        SCHEDULE   SUSPEND   ACTIVE
cronjob.batch/hippo-pgbackrest-repo1-full   @daily     True      0
```

We can then promote the standby cluster by removing or disabling its
`spec.standby` section:

```
spec:
  standby:
    enabled: false
```

This change triggers the promotion of the standby leader to a primary PostgreSQL
instance, and the cluster begins accepting writes.


## Next Steps

Now we've seen how to clone a cluster and perform a point-in-time-recovery, let's see how we can [monitor]({{< relref "./monitoring.md" >}}) our Postgres cluster to detect and prevent issues from occurring.
