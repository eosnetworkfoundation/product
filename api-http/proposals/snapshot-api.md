# Snapshot Scheduling API Design specs

:bulb: Following document embodies API proposal in relation to the following github issue https://github.com/AntelopeIO/leap/issues/396.

## :beginner: Design Info

### Preface
It was requested that snapshot functionality and, in particular, snapshot API needs to be extended to support additional functionality. In particular, ability to schedule snapshot(s), edit schedule and query snapshot status.

Following goals for an API were defined in mentioned above github issue:
- Scenario 1: Create a snapshot for a specific block num
- Scenario 2: Create a snapshot every X number of blocks

Following considerations/design decisions were also expressed:
- Ability to query existing requests and cancel them
- Do pending requests persist across nodeos restarts?

---


:small_blue_diamond:Status:
- [X] In progress
- [ ] In review. Reviewer:
- [ ] Finished
---
## :wrench: Proposal API additions


| **API URL** 	| **Purpose** 	| **Parameters** 	|
|----------	|----------	|----------	|
| /producer/schedule_snapshot         	| Adds a task to perform snapshot         	| start_block, end_block, block_spacing, snapshot_description     	|
| /producer/get_snapshot_requests         	| Returns scheduled snapshot requests         	| start_snapshot_request_id, limit (number of snapshots to return)         	|
| /producer/get_snapshot_request_status         	| Queries status of a snapshot request        	| Snapshot ID         	|
| /producer/unschedule_snapshot         	| Removes previously scheduled snapshot task         	| Snapshot ID         	|

### Scheduling Snapshot
- request takes 4 optional parameters - start_block, end_block, block_spacing, snapshot_description:
    - start_block: first block after which snapshot will be taken, if not set, will be taken immediately similar to an old create_snapshot call
    - end_block: block number after which schedule for the snapshot will be no longer active and snapshot task will be automatically removed. If end_block is not specified, snapshot task will never be auto removed and needs to be unscheduled with another api call to stop
    - block_spacing - number of blocks after which a recurring snapshot will be made. If not set, snapshot will be done only once and scheduled task will be removed
    - snapshot_description - optional description of the snapshot
- in case of success api call should return json describing scheduled snapshot task which should contain unique snapshot request id
- in case of error a descriptive message and error code should be returned

As an additional consideration when scheduling a snapshot should be it's alignment with previously scheduled snapshots. Some examples of possible cases:
- Duplicate snapshot requests should be rejected
- Requests that has different parameters but will result in executing snapshots at the same block num as previously scheduled snapshots should result in a single snapshot for block id

Please note that single snapshot request can trigger and correspond to multiple pending snapshots due to waiting for irreversibility or in the case where snapshots happenning on different forks

:bulb: Implementation note - It would be great to encapsulate this logic into a separate class, so we can tweak/alter it without touching other code

![image](https://user-images.githubusercontent.com/79997103/212993658-af39ac43-ee64-4578-8d8f-4cbe16dba1da.png)

### Retrieving Snapshot Requests
This endpoint will return all scheduled snapshot requests with pagination - starting from snapshot determined by 'start_snapshot_request_id' (optional) and returning 'limit' number of snapshots

![image](https://user-images.githubusercontent.com/79997103/212993725-debd027e-05d0-490b-bbe4-ee95e5d1820e.png)

### Retrieving Snapshot Status
Request takes snapshot_id as parameter and either returns object describing scheduled snapshot or an error

![image](https://user-images.githubusercontent.com/79997103/212993777-69c841d7-01a3-4565-b925-696b24332fdd.png)

### Unscheduling Snapshot
Request takes snapshot_id as parameter and returns either successful confirmation of snapshot task removal or an error. Multiple snapshots require multiple removal requests. In future API revisions this, if needed, can be extended to support an array of snashot ID's per request

![image](https://user-images.githubusercontent.com/79997103/212993890-5e8a0bd1-139a-49a5-9392-ae5134f98e52.png)

## :feet: Implementation

:bulb: Following section on a high level addresses proposed implementation of snapshot scheduling API

Implementation of API URL handlers and execution of the snapshot itself will be similar to a number of existing API endpoints and wont be described here

However, it is important to drill down into implementation of a "snapshot scheduling engine" that will encapsulate all business logic needed for API functionality

In particular following constraints needs to be accounted for:
- Ability to query existing requests and cancel them
- Make snapshot schedule persistent to nodeos restarts
- Handling of multiple scheduling requests and conflicts resolution
- Handling of forks

#### Database and data model
Database, probably in a form of a json file, will be used to hold snapshot schedule. It should contain at least the following data (example):

```json
{
 [
    {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 0,
         "snapshot_description": "Example of recurring snapshot",
         "block_spacing": 100,
         "start_block_num": 5000,
         "end_block_num": 10000,

    },
    {
         "snapshot_request_time": "2020-11-16T00:00:10.000Z",
         "snapshot_request_id": 1,
         "snapshot_description": "Example of one-time snapshot",
         "block_spacing": 0,
         "start_block_num": 5200,
         "end_block_num": 5200,

    }
 ],
 "pending_snapshots": [
     {
         "head_block_id": "ABC",
         "head_block_num": 5100,
         "head_block_time": "2020-11-16T00:00:30.000Z",
         "version": 6,
         "snapshot_name": "/home/me/nodes/node-ame/snapshots/snapshot-5100-ABC.bin.pending",
         "triggering_requests": [0]
     },
     {
         "head_block_id": "DEF",
         "head_block_num": 5100,
         "head_block_time": "2020-11-16T00:00:31.000Z",
         "version": 6,
         "snapshot_name": "/home/me/nodes/node-ame/snapshots/snapshot-5100-DEF.bin.pending",
         "triggering_requests": [0]
     },
     {
         "head_block_id": "GHI",
         "head_block_num": 5200,
         "head_block_time": "2020-11-16T00:01:21.000Z",
         "version": 6,
         "snapshot_name": "/home/me/nodes/node-ame/snapshots/snapshot-5200-GHI.bin.pending",
         "triggering_requests": [0, 1]
     }
 ]
}
```

Database is updated by following events:
- New snapshot request is scheduled
- Snapshot request has been unscheduled
- Pending snapshot has been generated
- Pending snapshot has been finalized due to irreversibility

Snapshot scheduling engine should be able to read this database file on startup, handle all the updates to it and support queries / interfaces to retrieve snapshot data coreresponding to a snapshot request and find which request produced given snapshot.


## :blush: Example of Use / Demo
:bulb: Following examples may need to be revised once API calls would be actually implemented

#### Snapshot "now", one time, asap
```shell
curl http://127.0.0.1:8888/v1/producer/schedule_snapshot | json_pp
```
will return:
```json
 {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 0,
         "snapshot_description": "",
         "block_spacing": 0,
         "start_block_num": 288076281,
         "end_block_num": 288076281,
}
```

#### Schedule recurring snapshot
```shell
curl -X POST http://127.0.0.1:8888/v1/producer/schedule_snapshot
   -H 'Content-Type: application/json'
   -d '{"block_spacing":"1000","start_block_number":"288076281", "end_block_number":"388076281"}'
```
will return:
```json
 {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 0,
         "snapshot_description": "",
         "block_spacing": 1000,
         "start_block_num": 288076281,
         "end_block_num": 388076281,
}
```

#### Schedule recurring snapshot that will never expire
```shell
curl -X POST http://127.0.0.1:8888/v1/producer/schedule_snapshot
   -H 'Content-Type: application/json'
   -d '{"block_spacing":"1000","start_block_number":"288076281","snapshot_description":"Example of recurring snapshot"}'
```
will return:
```json
 {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 0,
         "snapshot_description": "Example of recurring snapshot",
         "block_spacing": 1000,
         "start_block_num": 288076281,
         "end_block_num": 0,
}
```

#### Schedule a single snapshot at specific block height, and delete request after done
```shell
curl -X POST http://127.0.0.1:8888/v1/producer/schedule_snapshot
   -H 'Content-Type: application/json'
   -d '{start_block_number":"288076281"}'
```
will return:
```json
 {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 0,
         "snapshot_description": "",
         "block_spacing": 0,
         "start_block_num": 288076281,
         "end_block_num": 288076281,
}
```

#### Get snapshot requests
```shell
curl -X POST http://127.0.0.1:8888/v1/producer/get_snapshot_requests
   -H 'Content-Type: application/json'
   -d '{"start_snapshot_request_id":"0","limit":"10"}'
```
will return:
```json
{
 "snapshot_requests": [
    {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 0,
         "snapshot_description": "Example of recurring snapshot",
         "block_spacing": 100,
         "start_block_num": 5000,
         "end_block_num": 10000,
    },
    {
         "snapshot_request_time": "2020-11-16T00:00:10.000Z",
         "snapshot_request_id": 1,
         "snapshot_description": "Example of one-time snapshot",
         "block_spacing": 0,
         "start_block_num": 5200,
         "end_block_num": 5200,
    }
 ],
 "next_request_id": null
}

```

#### Get snapshot request status by ID
```shell
curl -X POST http://127.0.0.1:8888/v1/producer/get_snapshot_request_status
   -H 'Content-Type: application/json'
   -d '{"snapshot_request_id":"1"}'
```
will return:
```json
 {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 1,
         "snapshot_description": "Example of recurring snapshot",
         "block_spacing": 0,
         "start_block_num": 5000,
         "end_block_num": 10000,
         "pending_snapshots": [
             {
                 "head_block_id": "ABC",
                 "head_block_num": 5100,
                 "head_block_time": "2020-11-16T00:00:30.000Z",
                 "version": 6,
                 "snapshot_name": "/home/me/nodes/node-ame/snapshots/snapshot-5100-ABC.bin.pending",
                 "triggering_requests": [0]
             },
             {
                 "head_block_id": "DEF",
                 "head_block_num": 5100,
                 "head_block_time": "2020-11-16T00:00:31.000Z",
                 "version": 6,
                 "snapshot_name": "/home/me/nodes/node-ame/snapshots/snapshot-5100-DEF.bin.pending",
                 "triggering_requests": [0]
             }
        ]
}
```

Output includes pending snapshot(s) triggered by that specific snapshot request.

#### Unschedule snapshot by ID
```shell
curl -X POST http://127.0.0.1:8888/v1/producer/unschedule_snapshot
   -H 'Content-Type: application/json'
   -d '{"snapshot_request_id":"1"}'
```
will return:
```json
 {
         "snapshot_request_time": "2020-11-16T00:00:00.000Z",
         "snapshot_request_id": 1,
         "snapshot_description": "Example of recurring snapshot",
         "block_spacing": 0,
         "start_block_num": 288076281,
         "end_block_num": 288076281,
}
```


## :boom: Swagger API definition

:bulb: This can be found in snapshot-api branch, file: plugins/producer_api_plugin/producer.swagger.yaml

```yaml
/producer/schedule_snapshot:
    post:
      summary: schedule_snapshot
      description: Submits a request to generate a schedule for automated snapshot with given parameters. If request body is empty, triest to execute snapshot immediately. Returns error when unable to create snapshot.
      operationId: schedule_snapshot
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                block_spacing:
                  type: integer
                  description: Generate snapshot every block_spacing blocks
                start_block_num:
                  type: integer
                  description: Block number after which schedule starts
                  example: 5102
                end_block_num:
                  type: integer
                  description: Block number after which schedule ends
                  example: 15102
                snapshot_description:
                    type: string
                    description: Description of the snapshot
                    example: Daily snapshot

      responses:
        "201":
          description: OK
          content:
            application/json:
              schema:
                  type: object
                  properties:
                    snapshot_request_id:
                      type: integer
                      description: Unique id identifying current snapshot request
                    block_spacing:
                      type: integer
                      description: Generate snapshot every block_spacing blocks
                    start_block_num:
                      type: integer
                      description: Block number after which schedule starts
                      example: 5102
                    end_block_num:
                      type: integer
                      description: Block number after which schedule ends
                      example: 15102
                    snapshot_description:
                        type: string
                        description: Description of the snapshot
                        example: Daily snapshot
        "400":
          description: client error
          content:
            application/json:
              schema:
                $ref: "#/component/schema/Error"

  /producer/get_snapshot_requests:
    post:
      summary: get_snapshot_requests
      description: Returns a list of scheduled snapshots
      operationId: get_snapshot_status
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                start_snapshot_request_id:
                  type: integer
                  description: Snapshot offset
                limit:
                  type: integer
                  description: Number of snapshots to return

      responses:
        "201":
          description: OK
          content:
            application/json:
             schema:
                type: object
                properties:
                  next_request_id:
                    type: integer
                    description: Next available snapshot ID
                  snapshot_requests:
                    type: array
                    description: An array of scheduled snapshots
                    items:
                      type: object
                      properties:
                          snapshot_request_time:
                            type: string
                            description: Snapshot unix timestamp
                            example: 2020-11-16T00:00:00.000
                          snapshot_request_id:
                            type: integer
                            description: Unique id identifying current snapshot request
                          block_spacing:
                            type: integer
                            description: Generate snapshot every block_spacing blocks
                          start_block_num:
                            type: integer
                            description: Block number after which schedule starts
                            example: 5102
                          end_block_num:
                            type: integer
                            description: Block number after which schedule ends
                            example: 15102
                          snapshot_description:
                              type: string
                              description: Description of the snapshot
                              example: Daily snapshot
        "400":
          description: client error
          content:
            application/json:
              schema:
                $ref: "#/component/schema/Error"

  /producer/get_snapshot_request_status:
    post:
      summary: get_snapshot_request_status
      description: Queries a status of the existing scheduled snapshot. Returns error when unable to find snapshot requested or query its status.
      operationId: get_snapshot_status
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                snapshot_request_id:
                  type: integer
                  description: snapshot id
      responses:
        "201":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  snapshot_request_id:
                    type: integer
                    description: Unique id identifying current snapshot request
                  snapshot_request_time:
                    type: string
                    description: Snapshot unix timestamp
                    example: 2020-11-16T00:00:00.000
                  block_spacing:
                    type: integer
                    description: Generate snapshot every block_spacing blocks
                  start_block_num:
                    type: integer
                    description: Block number after which schedule starts
                    example: 5102
                  end_block_num:
                    type: integer
                    description: Block number after which schedule ends
                    example: 15102
                  snapshot_description:
                      type: string
                      description: Description of the snapshot
                      example: Daily snapshot
                  pending_snapshots:
                      type: array
                      description: List of pending snapshots
                      items:
                        type: object
                        properties:
                          head_block_id:
                            $ref: "https://docs.eosnetwork.com/openapi/v2.0/Sha256.yaml"
                          head_block_num:
                            type: integer
                            description: Highest block number on the chain
                            example: 5102
                          head_block_time:
                            type: string
                            description: Highest block unix timestamp
                            example: 2020-11-16T00:00:00.000
                          version:
                            type: integer
                            description: version number
                            example: 6
                          snapshot_name:
                            type: string
                            description: The path and file name of the snapshot
                            example: /home/me/nodes/node-name/snapshots/snapshot-0000999f99999f9f999f99f99ff9999f999f9fff99ff99ffff9f9f9fff9f9999.bin

        "400":
          description: client error
          content:
            application/json:
              schema:
                $ref: "#/component/schema/Error"

  /producer/unschedule_snapshot:
    post:
      summary: unschedule_snapshot
      description: Submits a request to remove identified by id recurring snapshot from schedule. Returns error when unable to create unschedule.
      operationId: unschedule_snapshot
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                snapshot_request_id:
                  type: integer
                  description: snapshot id
      responses:
        "201":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  snapshot_request_id:
                      type: integer
                      description: Unique id identifying current snapshot request
                  block_spacing:
                    type: integer
                    description: Generate snapshot every block_spacing blocks
                  start_block_num:
                    type: integer
                    description: Block number after which schedule starts
                    example: 5102
                  end_block_num:
                    type: integer
                    description: Block number after which schedule ends
                    example: 15102
                  snapshot_description:
                      type: string
                      description: Description of the snapshot
                      example: Daily snapshot
        "400":
          description: client error
          content:
            application/json:
              schema:
                $ref: "#/component/schema/Error"

```

###### tags: `Proposal` `Snapshot API`
