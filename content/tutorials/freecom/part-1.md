---
alias: xuakjj68lp
path: /docs/tutorials/freecom-graphql-server
layout: TUTORIAL
preview: freecom-part-1.png
description: Freecom is the free alternative to Intercom
tags:
  - freecom
related:
  further:
    - koo4eevun4
  more:
    - e8a6ajt8ax
---

# Freecom: Requirements Analysis & Designing the Data Model

<iframe height="315" src="https://www.youtube.com/embed/4q0fFEypacA" frameborder="0" allowfullscreen></iframe>

In this first part of our tutorial series, we're focussing on generating a data model from a set of requirements that we have for the Freecom app.

### Features

We have a few requirements for the app, here's a list of features we want to implement:

- A customer visits a website and can open the support window
- The support window displays a list of the customer's previous conversations with support agents if any, or otherwise directly opens a new conversation
- The customer can either continue one of the existing conversations or initiate a new one
- A support agent interacts through Slack with customers on the website
- Every conversation happens in one dedicated Slack channel
- Every conversation is associated with one customer and one support agent
- The support agent of a conversation can change any time if a different support agent responds in a Slack channel
- One support agent can engage in multiple conversations at a time
- A customer can upload files that the support agents can access

### Designing the GraphQL Schema

The [GraphQL schema](https://www.graph.cool/docs/reference/platform/data-schema-ahwoh2fohj) represents the _data model_ of an application. It defines the _structure_ of the information that we can retrieve from the backend. Let's try to translate the above requirements into appropriate entities that we will then use to set up the GraphQL API.

#### `Customer`

Our first type is a `Customer`. A customer has a `name` property and is associated with a list of `conversations `.

```graphl
type Customer {
  name: String!
  conversations: [Conversation!]! @relation(name: "ConversationsFromCustomer")
}
```

The `name` will be randomly generated for each customer, so that there is no overhead for them and they can start asking questions right away.

#### `Agent`

Similarly, the `Agent` type represents the _support agents_. Since the support agents will use Slack, they have two fields that represent their Slack identity: `slackUserId` and `slackUserName`.  

Each `Agent` is associated with the list of conversations that they're currently engaged in. However, since the agent that participates in a conversation can potentially change, we also maintain a list of `messages` that each agent sent in their chats. That is so that it's possible to clearly identify the sender of a message. Note that we didn't need to associate the `Customer` with any messages because the `customer` on a `Conversation` can never change, so we always know who was the customer that sent a specific message by tracing back through the `Conversation`. 

```graphql
type Agent {
  slackUserId: String!
  slackUserName: String!
  conversations: [Conversation!]! @relation(name: "ConversationsFromAgent")
  messages: [Message!]! @relation(name: "MessagesFromAgents")
}
```

Everyone joining the Slack channel will be able to act as a support agent using a specific command when sending a message.

#### `Conversation`

A `Conversation` is associated with one `Customer`, one `Agent` and a list of `messages`. 

In general, there will be _one Slack channel per conversation_. The name of a Slack channel is derived from the customer's name and and index that increments with every new conversation the customer initiates, so e.g. `cool-tomato-1` would be the Slack channel that represents the very first conversation of the customer named `cool-tomato`. We thus store the `slackChannelIndex` as specific field on the `Conversation` type.

Notice that the `agent` associated with the conversation might change, while the `customer` will always be the same.

```graphql
type Conversation {
  slackChannelIndex: Int!
  customer: Customer @relation(name: "ConversationsFromCustomer")
  agent: Agent @relation(name: "ConversationsFromAgent")
  messages: [Message!]! @relation(name: "MessagesInConversation")
}
```

#### `Message`

A `Message` only has one property `text` that represents the content of the message. It is further related to a `Conversation` and potentially to an `agent`. Notice that the `agent` field can be `null`! This is how we'll know whether a `Message` was sent either by a `Customer` or by an `Agent` - if the `agent` field is `null`, the `Message` must have been sent by a `Customer` which we can access through the associated `conversation` field.

```graphql
type Message {
  text: String!
  agent: Agent @relation(name: "MessagesFromAgents")
  conversation: Conversation @relation(name: "MessagesInConversation")
}
```

#### Complete Schema

The complete schema that we need in order to create the Graphcool backend looks as follows:

```graphql
type Customer {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  conversations: [Conversation!]! @relation(name: "ConversationsFromCustomer")
}

type Agent {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  slackUserId: String!
  slackUserName: String!
  messages: [Message!]! @relation(name: "MessagesFromAgents")
}

type Conversation {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  slackChannelName: String!
  customer: Customer @relation(name: "ConversationsFromCustomer")
  messages: [Message!]! @relation(name: "MessagesInConversation")
}

type Message {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  text: String!
  agent: Agent @relation(name: "MessagesFromAgents")
  conversation: Conversation @relation(name: "MessagesInConversation")
}
```

Notice that each of the types now also includes the Graphcool [system fields](https://www.graph.cool/docs/reference/platform/system-artifacts-uhieg2shio#id-field), `id`, `createdAt` and `updatedAt`.

Here is a graphical overview of the relations in our final schema:

![](http://i.imgur.com/bd5rQ2k.png)

- **one** `Agent` is related to **many** `Message`s
- **one** `Agent` is related to **many** `Conversation `s
- **one** `Customer` is related to **many** `Conversation`s
- **one** `Conversation ` is related to **many** `Message `s

## Preparing the GraphQL server

You can now either create these model types and relations manually in the Web UI of the  [Graphcool console](https://console.graph.cool) or use the [command-line interface](https://www.npmjs.com/package/graphcool) to create the project including the data model for you. Simply download the complete [schema file](https://github.com/graphcool-examples/schemas/blob/master/freecom.schema) and execute the following command in a Terminal:

```sh
graphcool create freecom.schema
```

Note that this will require you to authenticate with Graphcool by opening a browser window before creating the actual project.

Once the project was created, you can interact with it via two different endpoints:

- [`Simple API`](https://www.graph.cool/docs/reference/simple-api/overview-heshoov3ai/): This endpoint creates an intuitive GraphQL API based on the provided data model and is optimized for usage with Apollo - _it's the one we'll be using in this tutorial!_
- [`Relay API`](https://www.graph.cool/docs/reference/relay-api/overview-aizoong9ah/): This endpoint can be used in applications that use [Relay](https://facebook.github.io/relay/), a GraphQL client developed by Facebook and with some specific requirements for the GraphQL API

![](http://i.imgur.com/6yMWjrA.png)

We'll be using the endpoint for the `Simple API`! If you ever lose the endpoint, you can simply find it in the [Graphcool console](https://console.graph.cool) by selecting your project and clicking the _Endpoints_-button in the bottom-left corner.

![](http://imgur.com/kPF9uqs.png)

![](https://canvas-files-prod.s3.amazonaws.com/uploads/aeaa9c0f-8c1f-4191-aa59-8053e38b6e04/Console 2017-04-04 14-32-56.png)

## Playgrounds

If you're keen on trying out your GraphQL API before we start writing actual code in the next chapter, you can explore the capabilities of the API in a [Playground](https://www.graph.cool/docs/reference/platform/console-uh8shohxie/), a browser-based and interactive environment for interacting with a GraphQL server.

To open up a Playground, simply paste the GraphQL endpoint (so, in your case that's the URL for the `Simple API` ) into the address bar of a browser.

![](http://imgur.com/vamb7WQ.png)

![](https://canvas-files-prod.s3.amazonaws.com/uploads/255f0ee2-ed4f-4b78-a324-d555687365ca/vamb7WQ.png)

The left part of the Playground is for you to enter the queries and mutations you'd like to send to the GraphQL server. Right next to it on the right the responses sent by the server will be displayed. And finally, the rightmost area is for you to explore the documentation of the API, listing all available query, mutation and subscription fields. 

If you want to create some initial data, feel free to send a couple of mutations through the Playground, e.g. for creating a new `Customer`:

```graphql
mutation {
  createCustomer(name: "Weird-Broccoli") {
    id
  }
}
```

After having created this `Customer` in the DB, you can convince yourself that the data was actually stored by either verifying it in the [data browser](https://www.youtube.com/watch?v=XeLKw2BSdI4&t=18s) or by sending the following query for which the response should will now include the newly created user:

```graphql
allCustomers {
  id
  name
}
```

## Wrap Up

That's it for today! We hope you enjoyed this first part of our tutorial series and learning about how to create a GraphQL server from the command line using `graphcool create`.

In the next chapter, we'll start writing some actual code. We are going to write some basic React components and integrate them with the Apollo client, also sending our first queries and mutations to interact with the API. 

Let us know how you like the tutorial or ask any questions you might have. Contact us on [Twitter](www.twitter.com/graphcool) or join our growing community on [Slack](http://slack.graph.cool/)!

Until next time, stay (Graph)cool! 😎
