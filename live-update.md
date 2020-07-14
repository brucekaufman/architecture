
# Marley Live Update System Architecture

## Introduction

This document provides an overview of the live update system used to provide dynamic updates to enable real time applications built on the Marley backend.  The initial rollout of this system will occur with the mobile (iOS/android) app, but the plan is to backport this system to the web app as time and technical needs allow and require.

The live update system is composed of three levels, which are built on top of each other

* system to send live updates over a network (websockets)
* topic management used to control which clients will receive specific updates
* definition of live update payloads that will occur on specific events

## websockets

The live update system is built ontop of websockets. Websockets provide a persistent bidirectional connection between the client and server. When the client connects to a websocket, it can also subscribe and unsubscribe from topics which serve as channels for incoming events.

On the client side, Websockets can be instantiated using native [javascript websockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket).

Using javascript websockets, the client can connect to the websocket on a single channel and perform the handshake with one line:

```javascript
this.socket = new WebSocket('${host}?topic=all')
```

subscriptions to additional channels can then be created as follows:

```javascript
this.socket.send(JSON.stringify({
    action : 'subscribe',
    topic : 'new_topic'
}))
```

Likewise unsubscribing can be performed as follows:

```javascript
this.socket.send(JSON.stringify({
    action : 'unsubscribe',
    topic : 'some_topic'
}))
```

Connections may only persist for 2 hours after which a new connection must be established per API Gateway [limitations](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html) (there is a TTL of 2 hours on all topic connection pairs in conjunction to that limitation)

If making use of the SocketClient implementation catching topics is accessible via the EventEmitter interface:

```javascript
socket.on("someTopic", payload => {
    //some code
})
```

Current list of Marley environment websocket URLS:

| environment | url |
| ----------- | --- |
| apiuat | wss://7p3nupb05l.execute-api.us-east-1.amazonaws.com/dev|

## Topics in Marley Live update System

The initial implementation of the live update system will broadcast all events associated with a given organization on a single topic.  In the future, this may become more granular.  For now, a client should connect to a topic with the organization ID of the logged in user.  The channel should be constructed in the following format:

`${organizationId}-v${version}`

Where `organizationId` is the organization ID of the live update and `version` is the version of the live update payloads.  The initial release can be hardcoded to version 1 but eventually we may support multiple versions of payloads.

## Live Update Payloads

Various events that occur in the Marley backend will generate a payload which will be sent on the relevant channel(s) associated with that event. For example, suppose a case is modified, we can expect all sockets subscribed to the owner organization topic to receive a packet looking something like this

```json
    {
        payload : {
            ...some payload
        },
        dateEmitted : (some date),
        topic: (string of topic BE emitted to),
        event : (type of event),
        entity: (type of entity)
    }
```

All events will share a common format with the exception of the `payload` which will vary based on the specific event and entity.  Note that over time more entities may be added to the payload as the system functionality expands.  The entities will not be defined in this document.  Please review the API spec [TODO: add link] for details of the entity payloads.  

Note that along with the core entity, other entities may be included in the payload which are needed on the client side in reaction to the entity/event which triggered the live update.  For example, when a case is created the contact entity of the primary customer and the primary operator are included as this information is needed on the client.

### Case

*Note* : Not yet supported by marley backend

A live updated will be generated for a case when it is created, or when one of the case properties associated with it changes.  Examples of this include reassignment of the primary operator.

#### Payload

```json
{
    payload : {
      cases: [
          (case entity)
      ],
      contacts: [
        (contact entity of primary customer),
        (contact entity of primary operator)
      ],
    },
    dateEmitted : (some date),
    topic: "fdd07191-e9f5-437f-aef3-cfce0da1a95a-v1",
    entity : "case",
    event: (event type),

}
```

### Message

A live update will be generated for a message when it is created, or when some property of the message changes after creation. Currently this happens when a new translation of a message is created by the backend.  Note that these updates will be identical in format, regardless of the source of the message (either an operator using the marley app or an end user sending messages over SMS).  The live update will include the message itself, the case, and the contactsa associated with the primaryOperator and the 

#### Sample Payload

```json
{
  "payload": {
    "messages": [
      {
        "id": "fbde667e-f1be-4923-b79b-f2262356605f",
        "chatId": "76402ded-e67b-46fe-b7e0-00656ea57789",
        "body": "Test",
        "type": "text",
        "format": "user",
        "createdAt": "2020-07-12T18:48:12.881Z",
        "updatedAt": "2020-07-12T18:48:12.881Z",
        "authorId": "475ce5b6-0c14-4104-a61c-d8580d0b7b77",
        "sourceLanguage": "en",
        "channelSource": "mobile",
        "sendStatus": {
          "channel": "marley",
          "status": "sent"
        },
        "translations": {
          "en": "Translating..."
        }
      }
    ],
    "cases": [
      {
        "id": "9fc47a6d-d461-48f8-a18a-dd7e7c116be0",
        "caseType": {
          "id": "75dd685b-b04b-4113-83e1-8c23f1149c11",
          "url": "/case-types/75dd685b-b04b-4113-83e1-8c23f1149c11",
          "object": "case-type"
        },
        "referenceId": "f11560d8-0c09-11ea-8d71-362b9e155667",
        "state": "open",
        "updatedAt": "2020-07-12T18:48:26.307Z",
        "createdAt": "2020-07-12T18:39:30.369Z",
        "primaryOperator": {
          "id": "08f03402-a527-4d9f-9908-20988f4daeac",
          "url": "/contacts/08f03402-a527-4d9f-9908-20988f4daeac",
          "object": "contact"
        },
        "primaryCustomer": {
          "id": "475ce5b6-0c14-4104-a61c-d8580d0b7b77",
          "url": "/contacts/475ce5b6-0c14-4104-a61c-d8580d0b7b77",
          "object": "contact"
        },
        "primaryChat": {
          "id": "76402ded-e67b-46fe-b7e0-00656ea57789",
          "url": "/chats/76402ded-e67b-46fe-b7e0-00656ea57789",
          "object": "chat"
        },
        "chats": [
          {
            "id": "76402ded-e67b-46fe-b7e0-00656ea57789",
            "url": "/chats/76402ded-e67b-46fe-b7e0-00656ea57789",
            "object": "chat"
          }
        ],
        "metaData": {
          "f11560d8-0c09-11ea-8d71-362b9e155667": "2081002-1"
        }
      }
    ],
    "contacts": [
      {
        "id": "08f03402-a527-4d9f-9908-20988f4daeac",
        "role": "operator",
        "first": "Bruce",
        "last": "Kaufman",
        "languagePreference": "en",
        "email": {
          "value": "bruce.kaufman@himarley.com",
          "locked": false,
          "blocked": false,
          "verified": false
        },
        "mobile": {
          "value": "+15550002026",
          "locked": false,
          "blocked": false,
          "verified": false
        }
      },
      {
        "id": "475ce5b6-0c14-4104-a61c-d8580d0b7b77",
        "orgId": "fdd07191-e9f5-437f-aef3-cfce0da1a95a",
        "createdAt": "2020-07-12T18:39:25.755Z",
        "updatedAt": "2020-07-12T18:41:06.160Z",
        "role": "enduser",
        "first": "Dave",
        "last": "Wingert",
        "languagePreference": "en",
        "email": {
          "value": "dw@himarley.com",
          "locked": false,
          "blocked": false,
          "verified": false
        },
        "mobile": {
          "value": "+15552081002",
          "locked": false,
          "blocked": false,
          "verified": true
        }
      }
    ]
  },
  "topic": "fdd07191-e9f5-437f-aef3-cfce0da1a95a-v1",
  "eventType": "MESSAGE:CREATED",
  "dateEmitted": "2020-07-12T18:48:26.669Z"
}
 ```

### Valid event types

* created
* updated
