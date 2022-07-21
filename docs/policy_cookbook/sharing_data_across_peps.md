# Sharing data across PEPs

This example demonstrates how to share information across multiple PEPs within the Native Rule Engine Plugin.

## How to do it ...

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

