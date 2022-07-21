# Simulating User Quotas

This example demonstrates how to simulate user quotas using group quotas.

As of iRODS 4.3.0, support for user quotas has been partially disabled. Please consult `iadmin suq` and the release notes for more information.

## How to do it ...

### Step 1: Enable Quota Enforcement

First, we have to instruct iRODS to enforce quotas. Open `/etc/irods/core.re` and change the argument passed to `msiSetRescQuotaPolicy()` from `"off"` to `"on"`. The line you modified should look very similar to the one below.
```python
acRescQuotaPolicy { msiSetRescQuotaPolicy("on"); }
```
We recommend applying this change to all servers in the local zone. That guarantees that all servers enforce the quotas. Keep in mind that this is only a recommendation. You should use a testing environment to verify behavior if you decide not to follow this recommendation.

iRODS does not update the quota information following user interaction. To do that, you're going to need to periodically tell iRODS to update the quota information. One way to do that is by running `iadmin cu` periodically. How you do that is up to you. You can use **cron** or any other tool you find convenient to use. The important thing is that the command runs.

With that out the way, we can focus on applying the actual quota.

### Step 2: Setup the user quota

Given a user named **alice**, all we have to do is create a new group, add the user to it, and set a quota limit on the group.
```bash
$ iadmin mkgroup quota_group_for_alice           # Create the group.
$ iadmin atg quota_group_for_alice alice         # Add user to the group.
$ iadmin sgq quota_group_for_alice total 10000   # Set the group quota.
```

The important thing to remember is that only one user should be in the group. That user being the user you wish to apply the quota to. In a production environment, that may mean each user gets their own group. For that reason, it is recommended that you develop a good naming scheme for your groups. Having a good naming scheme will help other users understand the purpose of that group and allow you to write policy that protects those groups from modification.

## Let's see it in action!

Below are some examples of what you may see when a quota is violated.

Here is an `iput` example.
```bash
$ echo "This violates the quota!" > foo
$ iput foo
remote addresses: 127.0.0.1 ERROR: putUtil: put error for /tempZone/home/alice/foo, status = -110000 status = -110000 SYS_RESC_QUOTA_EXCEEDED
```

Depending on the client, the error information can change. For example, this is what happens when `istream` violates a quota.
```bash
$ echo "This violates the quota!" | istream write other_foo
Error: Cannot open data object.
```

Regardless of the client, the server's log file will always report the error. Notice the value of **log_message**.
```json
{
  "log_category": "legacy",
  "log_facility": "local0",
  "log_level": "error",
  "log_message": "[rsDataObjPut_impl:608] - [SYS_RESC_QUOTA_EXCEEDED: resource quota exceeded\n\n]",
  "request_api_name": "DATA_OBJ_PUT_AN",
  "request_api_number": 606,
  "request_api_version": "d",
  "request_client_user": "alice",
  "request_host": "127.0.0.1",
  "request_proxy_user": "alice",
  "request_release_version": "rods4.3.0",
  "server_host": "38832fb94fff",
  "server_pid": 2178,
  "server_timestamp": "2022-07-21T16:34:35.119Z",
  "server_type": "agent"
}
```
