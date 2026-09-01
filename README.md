MQTT Invalid CheckTransactions Test
===================================

Simulates malformed CheckTransactions responses from MQTT assets to verify
that the platform correctly closes sessions and prevents subscription leaks
(GCD-4695).

The script:
  - Connects registered devices to the MQTT broker over WebSockets (WSS)
  - Publishes intentionally malformed CheckTransactions responses
  - Waits for CloseSession commands from the platform
  - Measures response time and validates session cleanup


Requirements
------------
- Python 3.8+
- pip install locust
- pip install paho-mqtt


File Structure
--------------
project/
  locustfile.py               Main Locust test script
  config.json                MQTT broker configuration
  DeviceActivations.json     Device activations / MQTT credentials


Configuration
-------------

config.json

  {
    "mqtt_broker": {
      "host": "messaging-dev.ubiqular.com",
      "port": 8443,
      "websocket_path": "/mqtt",
      "keepalive": 60
    }
  }

Parameters:

mqtt_broker.host             MQTT broker host
mqtt_broker.port             MQTT broker port
mqtt_broker.websocket_path   WebSocket MQTT endpoint
mqtt_broker.keepalive        MQTT keepalive interval


DeviceActivations.json
----------------------

The script loads device activations generated during registration.

Expected structure:

  {
    "device1": {
      "activationData": {
        "typeId": "CI50",
        "assetId": "CI50001",
        "clientId": "...",
        "accessId": "...",
        "accessToken": "..."
      }
    }
  }

Each Locust user consumes one activation entry.


Environment Variables
---------------------

TARGET_TYPE_ID

Optional device type filter.

Example:

  TARGET_TYPE_ID=CI50

Only matching devices are used.


MAX_PENDING_SESSIONS_PER_USER

Maximum number of active sessions allowed per simulated device.

Default:

  10

Example:

  MAX_PENDING_SESSIONS_PER_USER=20


CLOSE_SESSION_TIMEOUT_SECONDS

Maximum time to wait for a CloseSession command.

Default:

  10

Example:

  CLOSE_SESSION_TIMEOUT_SECONDS=30


Malformed Test Cases
--------------------

Current configuration:

  MALFORMED_CASES = (
      "missing_detail",
  )

Supported malformed payload scenarios:

  empty_operation_source
  missing_operation_source
  null_operation_source

  missing_machine_type
  null_machine_type
  invalid_machine_type
  numeric_machine_type

  missing_transaction_numbers
  null_transaction_numbers
  string_transaction_numbers

  missing_current_transaction_number
  string_current_transaction_number

  missing_detail
  null_detail

Only the cases listed in MALFORMED_CASES are executed.

Example:

  MALFORMED_CASES = (
      "missing_detail",
      "null_detail",
      "missing_operation_source"
  )

The script rotates through the configured cases sequentially.


What the Test Does
------------------

1. Loads device activations from DeviceActivations.json.

2. Connects each device using:
   - MQTT over WebSockets
   - TLS
   - Activation credentials

3. Subscribes to:

   /glory/g-connect/{typeId}/{assetId}/action/CloseSession

4. Creates a unique session:

   /glory/g-connect-session/{typeId}/{assetId}/{sessionId}

5. Publishes a malformed CheckTransactions response to:

   .../response/CheckTransactions

6. Waits for the platform to send:

   .../action/CloseSession

7. Removes temporary subscriptions.

8. Clears retained response messages.

9. Records success or failure through Locust metrics.


Expected Flow
-------------

Malformed CheckTransactions Response
                  |
                  v
         Validation Error
                  |
                  v
            CloseSession
                  |
                  v
          Session Cleanup
                  |
                  v
        No Subscription Leak


Success Criteria
----------------

A test is considered successful when:

- Malformed payload is published successfully
- Platform detects the validation error
- CloseSession command is received
- Session subscription is removed
- Session is removed from the pending list

Locust metric example:

  CloseSession: missing_detail

with no exception reported.


Failure Criteria
----------------

MQTT connection failure:

  MQTT connection failed

Session subscription failure:

  Session subscription failed

Publish failure:

  Publish failed

CloseSession timeout:

  CloseSession was not received

If CloseSession is not received within
CLOSE_SESSION_TIMEOUT_SECONDS, the session is marked as failed.

This may indicate a regression of the original GCD-4695 issue.


Running the Test
----------------

Start Locust:

  locust -f locustfile.py

Open:

  http://localhost:8089

Configure:
  - Number of users
  - Spawn rate

Each Locust user uses one device activation from
DeviceActivations.json.


Metrics Reported
----------------

The script generates Locust metrics for:

  Connect

  Subscribe: CloseSession

  Subscribe: test session

  Publish: missing_detail

  CloseSession: missing_detail

These metrics are available through:
  - Locust UI
  - CSV exports
  - Distributed Locust runs


Session Cleanup
---------------

If CloseSession is received:

  - Session is removed from the pending list
  - Command subscription is removed
  - Empty retained response is published

If CloseSession is not received before timeout:

  - Session is forcefully cleaned up
  - Command subscription is removed
  - Empty retained response is published
  - TimeoutError is reported to Locust


Stopping
--------

When the test stops:

  - All pending sessions are cleaned up
  - All session subscriptions are removed
  - Empty retained responses are published
  - MQTT clients disconnect
  - MQTT network loops stop


Expected Result (GCD-4695 Verification)
---------------------------------------

Invalid Payload
      |
      v
Validation Error
      |
      v
CloseSession Sent
      |
      v
Session Removed
      |
      v
No Subscription Leak

The test passes when every malformed CheckTransactions response results in
a CloseSession command and all sessions are eventually cleaned up.
