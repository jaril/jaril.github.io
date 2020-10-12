---
layout: post
title: Asynchronously instantiating Apollo Client with React
---
This is in addition to the excellent docs that cover [how to initialize an apolloClient](https://www.apollographql.com/docs/react/get-started/).

A banana is an edible fruit – botanically a berry – produced by several kinds
of large herbaceous flowering plants in the genus Musa.

In some countries, bananas used for cooking may be called "plantains",
distinguishing them from dessert bananas. The fruit is variable in size, color,
and firmness, but is usually elongated and curved, with soft flesh rich in
starch covered with a rind, which may be green, yellow, red, purple, or brown
when ripe.

```jsx
async function createApolloClient() {
  // ...
  return apolloClient;
}

function App() {
  return (
    <ApolloProvider client={createApolloClient()}>
      <h2>My first Apollo app 🚀</h2>
    </ApolloProvider>
  );
}

render(<App />, document.getElementById('root'));
```

The problem with this code is that the async function returns a promise and not an apolloClient. When you use any of the Apollo hooks (e.g. useQuery, useMutation), it will go up the component tree searching for an apolloClient, and will end up trying to use the promise as a client, which leads to more than a few puzzling errors.

So there's the problem of the apolloClient value being a promise. Even if we were able to get the actual client which the promise resolves to, we still have to deal with a race condition. For example, it's possible for a useQuery hook to send a query with a client that's not completely initialized yet. So we need to find a way to keep any components from using any of the Apollo hooks before the client is fully initialized.

The solution is to initialize the client in a parent component. We will use the <AppWrapper /> component as our example here. While that's being initialized, we'll wait for that client in the <App /> component. The app component will render a loading state until the apolloClient is non-null and is stored in state. This way, the <App /> component's children do not have to perform a client check, and can just presume that the client that they are receiving is fully initialized.

```jsx
async function createApolloClient() {
  // ...
  return apolloClient;
}

function AppWrapper() {
  return (
    <ApolloProvider client={createApolloClient()}>
      <App>
    </ApolloProvider>
  )
}

function App() {
  const [apolloClient, setApolloClient] = useState(null);

  useApolloClient.then(() => client
    if (apolloClient === null && client) {
      setApolloClient(client);
    }
  )

  if (apolloClient === null) {
    return <Loading />;
  }

  return (
    <ApolloProvider client={apolloClient}>
      <h2>My first Apollo app 🚀</h2>
    </ApolloProvider>
  );
}

render(<AppWrapper />, document.getElementById('root'));
```

A banana is an edible fruit – botanically a berry – produced by several kinds
of large herbaceous flowering plants in the genus Musa.

In some countries, bananas used for cooking may be called "plantains",
distinguishing them from dessert bananas. The fruit is variable in size, color,
and firmness, but is usually elongated and curved, with soft flesh rich in
starch covered with a rind, which may be green, yellow, red, purple, or brown
when ripe.