---
alias: ui0eizishe
path: /docs/tutorials/worldchat-subscriptions-example
layout: TUTORIAL
preview: build-real-time-chat-with-subscriptions-and-apollo.png
shorttitle: How to Build Realtime Chat with GraphQL Subscriptions
description: Build a realtime chat where the users can see the locations of all participants on a map - using GraphQL subscriptions and the Apollo client
tags:
  - subscriptions
  - apollo
  - open-source
related:
  further:
    - aip7oojeiv
    - kengor9ei3
---

# How to build a Real-Time Chat with GraphQL Subscriptions and Apollo 🌍

In this tutorial, we explain how to build a chat application where the users can see their own and the locations of the other participants on a map. Not only the chat, but also the locations on the map get updated in realtime using GraphQL subscriptions.

You can check out a hosted demo of the application [here](https://demo.graph.cool/worldchat).

<iframe height="315" src="https://www.youtube.com/embed/aSLF9f13o2c" frameborder="0" allowfullscreen></iframe>

## What are GraphQL Subscriptions?

_Subscriptions_ are a GraphQL feature that allow to get **realtime updates** from the database in a GraphQL backend. You set them up by _subscribing_ to changes that are caused by specific _mutations_ and then execute some code in your application to react to that change.

Using the Apollo client, you can benefit from the full power of subscriptions. Apollo [implements subscriptions based on web sockets](https://dev-blog.apollodata.com/graphql-subscriptions-in-apollo-client-9a2457f015fb#.fapq8d7yc).

The simplest way to get started with a subscription is to specify a callback function where the modified data from the backend is provided as an argument. In a fully-fledged chat application where you're interested in any changes on the `Message` type, which is either that a _new message has been sent_, that _an existing message was modified_ or an _existing message was deleted_ this could look as follows:

```js
// subscribe to `CREATED`, `UPDATED` and `DELETED` mutations
this.newMessageObserver = this.props.client.subscribe({
  query: gql`
    subscription {
      Message {
        mutation # contains `CREATED`, `UPDATED` or `DELETED`
        node {
          text
          sentBy {
            name
          }
        }
      }
    }
  `,
  }).subscribe({
      next(data) {
      console.log('A mutation of the following type happened on the Message model: ', data.Message.mutation)
      console.log('The changed data looks as follows: ', data.Message.node)
    },
    error(error) {
      console.error('Subscription callback with error: ', error)
    },
})
```

> Note: This code assumes that you have configured and set up the `ApolloClient` and made it available in the `props` of your React component using [`withApollo`](http://dev.apollodata.com/react/higher-order-components.html#withApollo). We'll explain how to setup the `ApolloClient` in just a bit.


#### Figuring out the Mutation Type

The _kind_ of change that happened in the database is reflected by the `mutation` field in the payload which contains either of three values:

- `CREATED`: for a node that was _added_
- `UPDATED`: for a node that was _updated_
- `DELETED`: for a node that was _deleted_


#### Getting Information about the changed Node

The `node` field in the payload allows us to retrieve information about the modified node. It is also possible to ask for the state that node had _before_ the mutation, you can do so by including the `previousValues` field in the payload:

```graphql
subscription {
  Message {
    mutation # contains `CREATED`, `UPDATED` or `DELETED`
    # node carries the new values
    node {
      text
      sentBy {
        name
      }
    }
    # previousValues carries scalar values from before the mutation happened
    previousValues {
      text
    }
  }
}
```

Now you could compare the fields in your code like so:

```js
next(data) {
  console.log('Old text: ', data.Message.previousValues.text)
  console.log('New text: ', data.Message.node.text)
}
```

If you specify `previousValues` for a `CREATED` mutation, this field will just be `null`. Likewise, the `node` for a `DELETED` mutation will be `null` as well.


#### Subscriptions with Apollo

Apollo uses the concept of an `Observable` (which you might be familiar with if you have worked with [RxJS](https://github.com/Reactive-Extensions/RxJS) before) in order to deliver updates to your application.

Rather than using the updated data manually in a callback though, you can benefit from further Apollo features that conventiently allow you to update the local `ApolloStore`. We used this technique in our example Worldchat app and will explain how it works in the following sections.


## Setting up your Graphcool backend

First, we need to configure our backend. In order to do so, you can use the following [schema](!alias-ahwoh2fohj) file that represents the data model for our application:

```graphql
type Traveller {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  location: Location! @relation(name: "TravellerLocation")
  messages: [Message!]! @relation(name: "MessagesFromTraveller")
}

type Message {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  text: String!
  sentBy: Traveller!  @relation(name: "MessagesFromTraveller")
}

type Location {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  traveller: Traveller! @relation(name: "TravellerLocation")
  latitude: Float!
  longitude: Float!
}
```

Since the schema file is already included in this repository, all you have to do is download or clone the project and then use our CLI tool to create your project along with the specified schema:

```sh
git clone https://github.com/graphcool-examples/worldchat-subscriptions-example.git
cd worldchat
graphcool create Worldchat.schema
```

This will automatically create a project called `Worldchat` that you can now access in our [console](https://console.graph.cool).


## Setting up the Apollo Client to use Subscriptions

To get started with subscriptions in the app, we need to configure our instance of the `ApolloClient` accordingly. In addition to the GraphQL endpoint, we also need to provide a `SubscriptionClient` that handles the websocket connection between our app and the server. To find out more about how the `SubscriptionClient` works, you can visit the [repository](https://github.com/apollographql/subscriptions-transport-ws) where it's implemented.

To use the websocket client in your application, you first need to add it as a dependency:

```sh
yarn add subscriptions-transport-ws
```

Once you've installed the package, you can instantiate the `SubscriptionClient` and the `ApolloClient` as follows:

```js
import ApolloClient, { createNetworkInterface } from 'apollo-client'
import {SubscriptionClient, addGraphQLSubscriptions} from 'subscriptions-transport-ws'
import { ApolloProvider } from 'react-apollo'

// Create WebSocket client
const wsClient = new SubscriptionClient(`wss://subscriptions.graph.cool/v1/__PROJECT ID__`, {
  reconnect: true,
  connectionParams: {
    // Pass any arguments you want for initialization
  }
})
const networkInterface = createNetworkInterface({ uri: 'https://api.graph.cool/simple/v1/__PROJECT ID__' })

// Extend the network interface with the WebSocket
const networkInterfaceWithSubscriptions = addGraphQLSubscriptions(
  networkInterface,
  wsClient
)

const client = new ApolloClient({
  networkInterface: networkInterfaceWithSubscriptions,
})
```

> Note: You can get the your `__PROJECT ID__` directly from our [console](https://console.graph.cool). Select your project and navigate to `Settings -> General`.

Now, as usual, you will have to pass the `ApolloClient` as a prop to the `ApolloProvider` and wrap all components that you'd like to access the data that is managed by Apollo. In the case of our chat, this step looks as follows:

```js
class App extends Component {

  // ...

  render() {
    return (
      <ApolloProvider client={client}>
        <WorldChat />
      </ApolloProvider>
    )
  }
}
```

You can view the full implementation [here](https://github.com/graphcool-examples/worldchat-subscriptions-example/blob/master/src/App.js).


## Building a Real-Time Chat with Subscriptions 💬

Let's now look at how we implemented the chat feature in our application. You can refer to the [actual implementation](https://github.com/graphcool-examples/worldchat-subscriptions-example/blob/master/src/Chat.js) whenever you like.

All we need for the chat functionality is _one query_ to retrieve all messages from the database and _one mutation_ that allows us to create a new message:

```graphql
const allMessages = gql`
  query allMessages {
    allMessages {
      id
      text
      createdAt
      sentBy {
        id
        name
      }
    }
  }
`

const createMessage = gql`
  mutation createMessage($text: String!, $sentById: ID!) {
    createMessage(text: $text, sentById: $sentById) {
      id
      text
      createdAt
      sentBy {
        id
        name
      }
    }
  }
`
```

When exporting the component, we're making these two operations available to our component by wrapping them around it using Apollo's higher-order compoment [`graphql`](http://dev.apollodata.com/react/higher-order-components.html#graphql):

```js
export default graphql(createMessage, {name : 'createMessageMutation'})(
  graphql(allMessages, {name: 'allMessagesQuery'})(Chat)
)
```

We then subscribe for changes on the `Message` type, filtering for mutations of type `CREATED`.

> Note: Generally, a mutation can take one of three forms: `CREATED`, `UPDATED` or `DELETED`. Our subscription API allows to use a `filter` to specify which of these you'd like to subscribe to. If you don't specify a filter, you'll subscribe to _all_ of them by default. It is also possible to filter for more complex changes, e.g. for `UPDATED` mutations, you could only subscribe to changes that happen on a specific _field_.

```js
// Subscribe to `CREATED`-mutations
this.createMessageSubscription = this.props.allMessagesQuery.subscribeToMore({
  document: gql`
    subscription {
      Message(filter: {
        mutation_in: [CREATED]
      }) {
        node {
          id
          text
          createdAt
          sentBy {
            id
            name
          }
        }
      }
    }
  `,
  updateQuery: (previousState, {subscriptionData}) => {
    const newMessage = subscriptionData.data.Message.node
    const messages = previousState.allMessages.concat([newMessage])
    return {
      allMessages: messages,
    }
  },
  onError: (err) => console.error(err),
})
```

Notice that we're using a different method to subscribe to the changes compared the first example where we used `subscribe` directly on an instance of the `ApolloClient`. This time, we're calling [`subscribeToMore`](http://dev.apollodata.com/react/receiving-updates.html#Subscriptions) on the `allMessagesQuery` (which is available in the `props` of our compoment because we wrapped it with `graphql` before).

Next to the actual subscription that we're passing as the `document` argument to `subscribeToMore`, we're also passing a function for the `updateQuery` parameter. This function follows the same principle as a [Redux reducer](http://redux.js.org/docs/basics/Reducers.html) and allows us to conveniently merge the changes that are delivered by the subscription into the local `ApolloStore`.  It takes in the `previousState` which is the the former _query result_ of our `allMessagesQuery` and the `subscriptionData` which contains the payload that we specified in our subscription, in our case that's the `node` that carries information about the new message.

> From the Apollo [docs](http://dev.apollodata.com/react/receiving-updates.html#Subscriptions): `subscribeToMore` is a convenient way to update the result of a single query with a subscription. The `updateQuery` function passed to `subscribeToMore` runs every time a new subscription result arrives, and it's responsible for updating the query result.

Fantastic, this is all we need in order for our chat to update in realtime! 🚀


## Adding Geo-Locations to the App 🗺

Let's now look at how to add a geo-location feature to the app so that we can display the chat participants on a map. The full implementation is located [here](https://github.com/graphcool-examples/worldchat-subscriptions-example/blob/master/src/WorldChat.js).

At first, we need one query that we use to initially retrieve all locations and their associated travellers.

```js
const allLocations = gql`
  query allLocations {
    allLocations {
      id
      latitude
      longitude
      traveller {
        id
        name
      }
    }
  }
`
```

Then we'll use two different mutations. The first one is a [nested mutation](!alias-ubohch8quo) that allows us to initially create a `Location` along with a `Traveller`, rather than having to do this in two different requests:

```js
const createLocationAndTraveller = gql`
    mutation createLocationAndTraveller($name: String!, $latitude: Float!, $longitude: Float!) {
        createLocation(latitude: $latitude, longitude: $longitude, traveller: {
            name: $name
        }) {
            id
            latitude
            longitude
            traveller {
                id
                name
            }
        }
    }
`
```

We also have a simpler mutation that will be fired whenever a traveller logs back in to the system and we update their location:

```js
const updateLocation = gql`
  mutation updateLocation($locationId: ID!, $latitude: Float!, $longitude: Float!) {
    updateLocation(id: $locationId, latitude: $latitude, longitude: $longitude) {
      traveller {
        id
        name
      }
      id
      latitude
      longitude
    }
  }
`
```

Like before, we're wrapping our component before exporting it using `graphql`:

```js
export default graphql(allLocations, {name: 'allLocationsQuery'})(
    graphql(createTravellerAndLocation, {name: 'createTravellerAndLocationMutation'})(
      graphql(updateLocation, {name: 'updateLocationMutation'})(WorldChat)
  )
)
```

Finally, we need to subscribe to the changes on the `Location` model. Every time a new traveller and location are created or an existing traveller updates their location, we want to reflect this on the map.

However, in the second case when an existing traveller logs back in, we actually only want to receive a notification if their location is different from before, that is either `latitude` or `longitude` or both have to be changed through the mutation. We'll include this requirement in the subscription using a filter again:

```js
this.locationSubscription = this.props.allLocationsQuery.subscribeToMore({
  document: gql`
    subscription {
      Location(filter: {
        OR: [{
          mutation_in: [CREATED]
        }, {
          AND: [{
            mutation_in: [UPDATED]
          }, {
            updatedFields_contains_some: ["latitude", "longitude"]
          }]
        }]
      }) {
        mutation
        node {
          id
          latitude
          longitude
          traveller {
            id
            name
          }
        }
      }
    }
  `,
  updateQuery: (previousState, {subscriptionData}) => {
    // ... we'll take a look at this in a second
  }
})
```

Let's try to understand the `filter` step by step. We want to get notified in either of two cases:

- A new location was `CREATED`, the condidition that we specified for this is simply:

    ```graphql
    mutation_in: [CREATED]
    ```

- An existing location was `UPDATED`, however, there must have been a change in the `latitude` and/or `longitude` fields. We express this as follows:

    ```graphql
    AND: [{
      mutation_in: [UPDATED]
    }, {
      updatedFields_contains_some: ["latitude", "longitude"]
    }]
    ```

We're then putting these two cases together connecting them with an `OR`:

```graphql
OR: [{
  mutation_in: [CREATED]
}, {
  AND: [{
    mutation_in: [UPDATED]
  }, {
    updatedFields_contains_some: ["latitude", "longitude"]
  }]
}]
```

Now, we only need to specify what should happen with the data that we receive through the subscription - we can do so using the `updateQueries` argument of `subscribeToMore` again:

```js
this.locationSubscription = this.props.allLocationsQuery.subscribeToMore({
  document: gql`
    // ... see above for the implementation of the subscription
  `,
  updateQuery: (previousState, {subscriptionData}) => {
    if (subscriptionData.data.Location.mutation === 'CREATED') {
      const newLocation = subscriptionData.data.Location.node
      const locations = previousState.allLocations.concat([newLocation])
      return {
        allLocations: locations,
      }
    }
    else if (subscriptionData.data.Location.mutation === 'UPDATED') {
      const updatedLocation = subscriptionData.data.Location.node
      const locations = previousState.allLocations.concat([updatedLocation])
      const oldLocationIndex = locations.findIndex(location => {
        return updatedLocation.id === location.id
      })
      locations[oldLocationIndex] = updatedLocation
      return {
        allLocations: locations,
      }
    }
    return previousState
  }
})
```

In both cases, we're simply incorporating the changes that we received from the subscription and specify how they should be merged into the `ApolloStore`. In the `CREATED`-case, we just append the new location to the existing list of locations. In the `UPDATED`-case, we replace the old version of that location in the `ApolloStore`.


## Summing Up

In this tutorial, we've only scratched the surface of what you can do with our subscription API. To see what else is possible, you can check out our [documentation](!alias-aip7oojeiv).


## Help & Community

Join our [Slack community](http://slack.graph.cool/) if you run into issues or have questions. We love talking to you!
