#

iRODS 5+ launches a long-running server process (`/usr/sbin/irodsServer`), which immediately forks the Agent Factory and spawns the Delay Server (`/usr/sbin/irodsDelayServer`).  All incoming API requests are serviced by the Agent Factory, and for every new connection the Agent Factory spawns a new Agent.  Each Agent will service its request as quickly as possible, and then terminate.

The iRODS Delay Server sleeps most of the time, but spawns an Agent every 30 seconds to check the delay queue for any delayed rules that need to be run. The frequency of the check is configurable. See <INSERT_LINK> for details.

Below is an example showing the process tree for an iRODS server.

    irodsServer
      ├─irodsAgent
      │   ├─irodsAgent
      │   ├─irodsAgent
      │   └─irodsAgent
      └─irodsDelayServer

| iRODS Process         | Expected Duration        | Expected Resident Memory Usage  |
| --------------------- | ------------------------ | ------------------------------- |
| irodsServer           | Long-running             | 34M                             |
| irodsDelayServer      | Long-running             | 33M                             |
| irodsAgent (factory)  | Long-running             | 92M                             |
| irodsAgent            | Depends on client        | 92M (minimum)                   |

TODO: Update or remove graphic.
![iRODS Process Model Diagram](../images/process_model_diagram.jpg)
