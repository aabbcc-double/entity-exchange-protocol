# Entity Exchange Protocol
### 1. Meta information

| label        | value                  | description |
| ---          | ---                    |  ----       |
| Version      | v0.1                   | Current version of the protocol |
| Status       | Draft                  | Current status of paper | 
| Author       | Shakhzod Ikromov       |             |
| Contributors |                        |             |

### 2. Abstract
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this 
document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

### 3. Connection
It's required to have exactly two instances of peers implementing Entity Exchange Protocol (master and slave) in order to work properly. 
Connecting more than two peers is not described here but workarounds are suggessted. Slave always makes a connection to master, master 
accepts incoming connections.

Messages beetween master and slave are sent over a single TCP connection. **Recommended port** for master is 9033, but not strictly 
required. TCP connection may be closed at any time. When the connection is closed, any write or read operations including pending ones 
should be aborted and/or discarded. If connection isn't closed by `CLOSE` command, attempts to reestablish the connection should be 
invoked regularly.

### 4. Messaging
##### 4.1 Introduction
Messages are sent simultaneously by writing and reading data from the TCP socket. Each message represents a command. Peers may discard, 
ignore or respond to a command.

##### 4.2 Message structure
Each message is terminated by `NULL` (or simply `\0`, `0x00`) and in document it appears as `\END`. Peers **MUST** read a command up to 
`\END` before reacting or parsing the input. Maximum length of each command including `\END` isn't determined by now.

Message structure can be represented like this: `COMMAND JSONDATA\END` where `COMMAND` and `JSONDATA` are delimitered by space (` ` or 
`0x20`). `JSONDATA` is a JSON structure is encoded as specified in [json.org](https://www.json.org). 

##### 4.3 Commands and their JSON structures
 * META - Metadata of the peer. This command is sent immediately by slave when connection is established
   
   | JSON field     | Type    | Mandatory   | Description |
   | ---------      | ------- | ----------- | ------------ |
   | **version**    | string  | required    | Version of the used protocol |
   | **impl**       | string  | required    | Name and version of the implementation | 
   | _custom fields_| _..._   | optional    | User defined custom fields |

 * CLOSE - Command for closing a connection. 
   
   | JSON field     | Type    | Mandatory   | Description |
   |  ------------  | ---     | ---         | ---         |
   | reason         | string  | optional    |             |
 
 * REFRESH - Slave asks for any changes starting the date

   | JSON field     | Type    | Mandatory   | Description |
   | ---            | ---     | ---         | ---         |
   | **from**       | date and time in UTC [(ISO 8601)](https://en.wikipedia.org/wiki/ISO_8601) | Start sending changes starting from the date |

 * EADD - An entity added to the store
  
   | JSON field     | Type    | Mandatory   | Description |
   | ---            | ---     | ---         | ---         |
   | **collection** | string  | required    | collection of the entity. |
   | **entity**     | object  | required    | Object to insert to       |
  
 * EDELETE - Marks an entity as deleted

   | JSON field     | Type    | Mandatory   | Description |
   | ---            | ---     | ---         | ---         |
   | **filter**     | object  | required    | Delete all entities look like passed filter  |

 * EREPLACE - Updates or creates enitity 

   | JSON field     | Type    | Mandatory   | Description |
   | ---            | ---     | ---         | ---         |
   | **filter**     | object  | required    | Filter to use |
   | **collection** | string  | required    | Collection to insert into |
   | **entity**     | object  | required    | Object to store |

### 5. Data exchange
##### 5.1 Local storage
Both peers should store entities as array items where array is a named collection. When inserting new entity, peers must create a named 
collection when necessary and append the entity. It's illegal to modify entity upon saving or reading without explicit request (by 
consumer). Peers must save meta information within the entity. Required metadata fields at least: `lastLocalEditDate`, `lastRemoteEditDate`,
`isDeleted`. Implementations are free to add additional fields to the metadata and update them at any time.

##### 5.2 Data transfer
First slave gets maximum of `lastRemoteEditDate` (next `RMAX`) from local entities from each collection, and collects all entities where 
`lastLocalEditDate` exists and is greater than `RMAX` and starts to send to the master:
 - if there is a `lastRemoteEditDate` and `isDeleted` true than the entity should be sent as `EDELETE`. Filtering and identifying the 
entity is defined by consumer
 - if there is a `lastRemoteEditDate`then entity should be sent as `EREPLACE` command. Filtering and identifying the entity is defined by 
the consumer. 
 - if there is not `lastRemoteEditDate` and `isDeleted` is false then entity should be sent as `EADD` command.
 - if there is not `lastRemoteEditDate` and `isDeleted` true then then the entity should be deleted locally.

Writing a command to stream of the socket can be treated as successful transfer. After each successful transfer the entity's 
`lastRemoteEditDate` should be set to the entity's `lastLocalEditDate`.
 
