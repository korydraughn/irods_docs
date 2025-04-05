#

## Starting the server

The easiest way to start an iRODS 5 server is by running the following command as the iRODS service account user.

```bash
$ irodsServer -d
```

The server is launched as a daemon process and writes log messages to syslog. The root PID of the active iRODS server is written to `/var/run/irods/irods-server.pid` by default. To specify a different file, use `--pid-file`.

iRODS 5 supports various launch options and is fully compatible with systemd. See the help text to learn more, i.e. `irodsServer -h`.

## Stopping the server

To stop an iRODS server, send SIGTERM or SIGQUIT to the root server process, e.g. `kill $(cat /var/run/irods/irods-server.pid)`.

SIGTERM instructs the server to shut down fast. Agents will complete the active request and terminate.

SIGQUIT instructs the server to shut down gracefully. Agents are allowed to continue servicing requests for a limited amount of time. Once the time limit is reached, agents will perform the normal shutdown sequence as if SIGTERM was received by the server. SIGINT can be used to perform this action as well.

## Reloading Configuration

TODO
