#

## Running the server

The easiest way to start an iRODS 5 server is by running the following command using the service account.

```bash
$ irodsServer -d
```

The server is launched as a daemon process and writes log messages to syslog.

iRODS 5 supports various launch options and is fully compatible with systemd. See the help text to learn more, i.e. `irodsServer -h`.

## Test Mode

This mode instructs the server to write all log messages to an additional log file. Messages will still be written to stdout or syslog depending on how the server was launched. This secondary log file is located at the following location:

```bash
/var/lib/irods/log/test_mode_output.log
```

This mode was introduced to guarantee that log messages appear in the log file immediately. This mode is only meant to be used for testing. The log file produced by this mode is never rotated.

To enable this mode, you must first define the following environment variable in the iRODS service account:

```bash
export IRODS_ENABLE_TEST_MODE=1
```

This environment variable instructs the `IrodsController` python class to start the server in test mode.

**If you are using the run_tests.py script to run tests, then you can ignore the rest of this section because run_tests.py restarts the server in test mode automatically.**

Next, you'll want to start (or restart) the server in test mode.

```bash
$ irodsServer -t
```
