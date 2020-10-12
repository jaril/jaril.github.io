---
layout: post
title: Asynchronously instantiating Apollo Client with React
---
This is in addition to the excellent docs that cover [how to initialize an apolloClient](https://www.apollographql.com/docs/react/get-started/).

While instantiating an Apollo Client, it's possible that you might need to do it asynchronously. For our particular use case, we had to do it that way because we needed to fetch an auth0 token that was then set in the default header of the Apollo Client.

This is what our first attempt looked like:

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

The solution is to initialize the client in a parent component. We will use the <AppWrapper /> component as our example here.

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

While that's being initialized, we'll wait for that client in the <App /> component. The app component will render a loading state until the apolloClient is non-null and is stored in state. This way, the <App /> component's children do not have to perform a client check, and can just presume that the client that they are receiving is fully initialized.