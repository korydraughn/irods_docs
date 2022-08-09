# The iRODS Policy Cookbook

This section is all about presenting techniques to various policy-based situations encountered in the iRODS ecosystem.

Over time, we hope this cookbook will grow and become a valuable resource for new and existing users of iRODS.

If you have suggestions on how to improve the cookbook or you've developed solutions that you feel could help others, please reach out to us by creating an issue at [https://github.com/irods/irods_docs](https://github.com/irods/irods_docs).

## Metadata-based Synchronization

These examples demonstrate how to synchronize rules using metadata.

- live systems will have multiple rules being executed
- these techniques help to sync any number of rules
- \*one technique relies on the fact that iRODS doesn't allow duplicate AVUs

### Example 1: Controlling execution order

#### How to do it ...

```
is_metadata_attached_to_collection(*collection, *attribute_name)
{
    *attached = false;

    foreach (*row in select COLL_NAME where COLL_NAME = '*collection' and META_COLL_ATTR_NAME = '*attribute_name') {
        *attached = true;
    }

    *attached;
}

wait_for_metadata_signal(*collection, *attribute_name)
{
    while (!is_metadata_attached_to_collection(*collection, *attribute_name)) {
        msiSleep('1', '0');
    }
}

synchronized_delay_rules_example()
{
    *number_of_delay_rules = 5;

    # Schedule multiple delay rules.
    for (*i = 0; *i < *number_of_delay_rules; *i = *i + 1) {
        *signum = *i;

        delay("<INST_NAME>irods_rule_engine_plugin-irods_rule_language-instance</INST_NAME>") {
            wait_for_metadata_signal('/tempZone/home/rods', 'irods::signal_*signum');

            # Log a message to let the admin know the metadata signal was received.
            writeLine("serverLog", "Received signal! (irods::signal_*signum)");

            # Set metadata for the next delay rule.
            *next_signum = *signum + 1;
            writeLine("serverLog", "Signaling next delay rule (irods::signal_*next_signum) ...");
            msiModAVUMetadata('-C', '/tempZone/home/rods', 'add', 'irods::lock_*next_signum', 'unused', '');
        }
    }
}
```

## Naming Schemes and Conventions 

- TODOs
    - rule name prefix for NREP libraries
    - temporaryStorage key name scoping (e.g. logical quotas prototype, \*key = "irods::logical_quotas::" ++ name)
    - metadata prefix / scoping

## Sharing data across PEPs

This example demonstrates how to share information across multiple PEPs within the Native Rule Engine Plugin.

### How to do it ...

Let's say we have the following rules defined in `/etc/irods/core.re`.

Pay close attention to the use of `temporaryStorage`. It is the bridge between PEPs within the Native Rule Engine Plugin.

```python
pep_api_data_obj_put_pre(*INSTANCE_NAME, *COMM, *DATA_OBJ_INPUT, *BYTES_BUFFER, *PORTAL_OPR_OUTPUT)
{
    # Capture the logical path of the PUT operation.
    # We can use it later when writing messages to the log file.
    temporaryStorage."logical_path" = *DATA_OBJ_INPUT.objPath;
}

pep_api_data_obj_put_post(*INSTANCE_NAME, *COMM, *DATA_OBJ_INPUT, *BYTES_BUFFER, *PORTAL_OPR_OUTPUT)
{
    # Fetch the logical path we captured during the call to open.
    *logical_path = temporaryStorage."logical_path";

    # Write a message to the log file containing the full logical path.
    writeLine("serverLog", "This string, *logical_path, was captured by the pre-PEP and logged by the post-PEP!");
}
```

If you run the following commands:
```bash
$ touch foo
$ iput foo
```

And inspect the log file at `/var/log/irods/irods.log`, you'd see a log message similar to the following:
```json
{
  "log_category": "legacy",
  "log_facility": "local0",
  "log_level": "info",
  "log_message": "writeLine: inString = This string, /tempZone/home/rods/foo, was captured by the pre-PEP and logged by the post-PEP!\n",
  "request_api_name": "DATA_OBJ_PUT_AN",
  "request_api_number": 606,
  "request_api_version": "d",
  "request_client_user": "rods",
  "request_host": "127.0.0.1",
  "request_proxy_user": "rods",
  "request_release_version": "rods4.3.0",
  "server_host": "38832fb94fff",
  "server_pid": 24234,
  "server_timestamp": "2022-07-24T16:39:24.348Z",
  "server_type": "agent"
}
```

The prototype implementation of the Logical Quotas Rule Engine Plugin uses this technique. You can find the implementation at [https://github.com/irods/irods_policy_examples/blob/main/irods_policy_logical_quotas.re](https://github.com/irods/irods_policy_examples/blob/main/irods_policy_logical_quotas.re)

Additional documentation can be found at [Using temporaryStorage in the iRODS Rule Language](system_overview/tips_and_tricks/#using-temporarystorage-in-the-irods-rule-language).

## Simulating User Quotas

This example demonstrates how to simulate user quotas using group quotas.

As of iRODS 4.3.0, support for user quotas has been partially disabled. Please consult `iadmin suq` and the release notes for more information.

### How to do it ...

#### Step 1: Enable Quota Enforcement

First, we have to instruct iRODS to enforce quotas. Open `/etc/irods/core.re` and change the argument passed to `msiSetRescQuotaPolicy()` from `"off"` to `"on"`. The line you modified should look very similar to the one below.
```python
acRescQuotaPolicy { msiSetRescQuotaPolicy("on"); }
```
We recommend applying this change to all servers in the local zone. That guarantees that all servers enforce the quotas. Keep in mind that this is only a recommendation. You should use a testing environment to verify behavior if you decide not to follow this recommendation.

iRODS does not update the quota information following user interaction. To do that, you're going to need to periodically tell iRODS to update the quota information. One way to do that is by running `iadmin cu` periodically. How you do that is up to you. You can use **cron** or any other tool you find convenient to use. The important thing is that the command runs.

With that out the way, we can focus on applying the actual quota.

#### Step 2: Setup the user quota

Given a user named **alice**, all we have to do is create a new group, add the user to it, and set a quota limit on the group.
```bash
$ iadmin mkgroup quota_group_for_alice           # Create the group.
$ iadmin atg quota_group_for_alice alice         # Add user to the group.
$ iadmin sgq quota_group_for_alice total 10000   # Set the group quota.
```

The important thing to remember is that only one user should be in the group. That user being the user you wish to apply the quota to. In a production environment, that may mean each user gets their own group. For that reason, it is recommended that you develop a good naming scheme for your groups. Having a good naming scheme will help other users understand the purpose of that group and allow you to write policy that protects those groups from modification.

### Let's see it in action!

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

