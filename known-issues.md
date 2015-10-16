# Known Issues

### Elastic Runtime HTTPS-only feature
Support for the Experimental HTTPS-only feature is broken in p-mysql versions 1.6.X and earlier. The HTTPS-only feature will work as expected in p-mysql 1.7.0 and later.

### Changing service plan definition
In p-mysql versions 1.6.3 and earlier, there's only one service plan. Changing the definition of that plan, either the number of megabytes, number of connections, or both, will make it so that any new service instances will have those characteristics.

There is a bug in p-mysql versions 1.6.3 and earlier. Changing the plan does not change existing service instances - they'll continue to be governed by the plan constraints effective at the time they were created. This is true, regardless if the an operator runs `cf update-service`.

There is a workaround for this bug, which will be resolved in future releases of p-mysql. In order for the change to be effective for existing plans, you'll need to trigger this by interacting directly with the service broker:

```
curl -v -k -X PATCH https://BROKER_CREDS_USERNAME:BROKER_CREDS_PASSWORD@p-mysql.SYSTEM.DOMAIN/v2/service_instances/SERVICE_INSTANCE_ID?plan_id=17d793e6-6da6-4f0e-b58d-364a407166a0
```

- SYSTEM.DOMAIN is defined in OpsManager, under the Elastic Runtime tile's `Settings Tab`, in the `Cloud Controller` entry.
- BROKER_CREDS_USERNAME and BROKER_CREDS_PASSWORD are defined in OpsManager, under the p-mysql tile's `Credentials` tab, in the entry `Broker Auth Credentials`.
- To get each SERVICE_INSTANCE_ID, run `cf service INSTANCE --guid`. You should see output like this example:

  ```
  $ cf service acceptDB --guid
  4cae3a5e-66b1-4c9a-8536-feaff25237bf
  ```

Run this for each service instance to be updated. We apologise for the inconvenience.

### Proxies may write to different MySQL masters
All proxy instances use the same method to determine cluster health. However, certain conditions may cause the proxy instances to route to different nodes, for example after brief cluster node failures.

This could be an issue for tables that receive many concurrent writes. Multiple clients writing to the same table could obtain locks on the same row, resulting in a deadlock. One commit will succeed and all others will fail and must be retried. This can be prevented by configuring your load balancer to route connections to **only one proxy instance at a time**.

### Number of proxy instances cannot be reduced.
Once the product is deployed with operator-configured proxy IPs, the number of proxy instances can not be reduced, nor can the configured IPs be removed from the **Proxy IPs** field. If instead the product is initially deployed without proxy IPs, IPs added to the **Proxy IPs** field will only be used when adding additional proxy instances, scaling down is unpredictably permitted, and the first proxy instance can never be assigned an operator-configured IP.

### MyISAM tables
The clustering plugin used in this release (Galera) does not support replication of MyISAM Tables. However, the service does not prevent the creation of MyISAM tables. When MyISAM tables are created, the tables will be created on every node (DDL statements are replicated), but data written to a node won't be replicated. If the persistent disk is lost on the node data is written to (for MyISAM tables only), data will be lost. To change a table from MyISAM to InnoDB, please follow this [guide](http://dev.mysql.com/doc/refman/5.5/en/converting-tables-to-innodb.html).

### Max user connections
When updating the max_user_connections property for an existing plan, the connections currently open will not be affected (ie if you have decreased from 20 to 40, users with 40 open connections will keep them open). To force the changes upon users with open connections, an operator can restart the proxy job as this will cause the connections to reconnect and stay within the limit.  Otherwise, if any connection above the limit is reset it won't be able to reconnect, so the number of connections will eventually converge on the new limit.

### Long SST transfers
We provide a `database_startup_timeout` in our manifest which specifies how long to wait for the initial [SST](proxy.md#state-snapshot-transfer-sst) to complete (default is 150 seconds). If the SST takes longer than this amount of time, the job will report as failing. Versions before `cf-mysql-release v23` have a flaw in our startup script where it does not kill the mysqld process in this case. When monit restarts this process, it sees that mysql is still running and exits without writing a new pidfile. This means the job will continue to report as failing. The only way to fix this is to SSH onto the failing node, kill the mysqld process, and re-run `monit start mariadb_ctrl`.