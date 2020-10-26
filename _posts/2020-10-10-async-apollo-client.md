---
layout: post
title: Asynchronously instantiating Apollo Client with React
---
Instantiating an Apollo Client synchronously is covered thoroughly in [Apollo's documentation](https://www.apollographql.com/docs/react/get-started/). What it doesn't cover are cases where the developer might have to instantiate the client asynchronously.

#### Synchronous instantiation

```jsx
import { ApolloClient, InMemoryCache } from '@apollo/client';

const client = new ApolloClient({
  uri: 'https://48p1r2roz4.sse.codesandbox.io',
  cache: new InMemoryCache()
});

function App() {
  return (
    <ApolloProvider client={apolloClient}>
      <h2>My first Apollo App
    </ApolloProvider>
  );
}

render(<App />, document.getElementById('root'));
```

This is fairly straightforward. We first instantiate the `ApolloClient` so that we can we pass into `ApolloProvider` as the `client` prop. This allows any child component of `ApolloProvider` to access the client when the child uses any of Apollo's hooks (e.g. `useQuery`, `useMutation`).

#### Asynchronous instantiation (Part 1)

In some cases, you might have to do instantiate the `ApolloClient` asynchronously. This might be the case if you need to perform some async logic to create your `ApolloClient`. For illustration, let's abstract that async logic into the async function `createApolloClient`. One way of doing this is shown below:

```jsx
async function createApolloClient() {
  // ...
  return apolloClient;
}

function App() {
  const [apolloClient, setApolloClient] = useState(null);

  if (!apolloClient) {
    createApolloClient().then(function(client) {
      setApolloClient(client);
    });

    return <LoadingScreen />;
  }

  return (
    <ApolloProvider client={apolloClient}>
      <h2>My first Apollo App
    </ApolloProvider>
  );
}

render(<App />, document.getElementById('root'));
```

When the `App` component is first rendered, we don't have an `apolloClient` yet so we call the async function `createApolloClient`. Because it's an async function, we can't simply use its return value and set it to state. Instead, we attach a callback function that takes the resolved value (our `client`) and sets it to state. After this we render a `LoadingScreen` component.

Once `createApolloClient` resolves with the `apolloClient`, our callback function is called and the new client is set into state. This change in state triggers re-renders the `App` component, which now has the `apolloClient`.

#### Asynchronous instantiation (Part 2)

Consider a similar set up from the previous example, but this time, `createApolloClient` requires a value that's provided by a [React context](https://reactjs.org/docs/context.html#contextconsumer). That might sound convoluted, but it happens. For example, what if you're using Auth0 for your app's authentication and had to generate a user token to attach to your `ApolloClient`'s headers?

```jsx
async function createApolloClient(auth0Client) {
  // ...
  return apolloClient;
}

function App() {
  return (
    <Auth0ProviderWithHistory>
      <Auth0Context.Consumer>
        {auth0Client => {
          return (
            <ApolloProvider client={createApolloClient(auth0Client)}>
              <h2>My first Apollo App
            </ApolloProvider>
          );
        }}
      </Auth0Context.Consumer>
    </Auth0ProviderWithHistory>
  );
}

render(<App />, document.getElementById('root'));
```

The main problem with this code is that the `createApolloClient` returns a promise, not the `apolloClient`. This is a bad implementation because if your app uses any of the Apollo hooks (e.g. `useQuery`, `useMutation`), those hooks will go up the component tree searching for an `apolloClient`. They'll see the `ApolloProvider`'s `client prop, assume that that's the client, and use it as such. Because it's a promise, this leads to errors.

Then there's another problem. Even if we were able to get the actual `apolloClient` which the promise resolves to, we still have to deal with a race condition. For example, it's possible for a `useQuery` hook to send a query with a client that's not completely initialized yet. So we need to find a way to keep any components from using any of the Apollo hooks before the client is fully initialized.

The solution is to initialize the `apolloClient` in a wrapper component. We will use the <AppWrapper /> component in our example:

```jsx
async function createApolloClient() {
  // ...
  return apolloClient;
}

function AppWrapper() {
  return (
    <Auth0ProviderWithHistory>
      <Auth0Context.Consumer>
        {auth0Client => {
          return (
            <ApolloProvider client={createApolloClient(auth0Client)}>
              <App />
            </ApolloProvider>
          );
        }}
      </Auth0Context.Consumer>
    </Auth0ProviderWithHistory>
  );
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

In this example, we call `createApolloClient` in the `AppWrapper` component. That means that the `ApolloProvider` client prop is a promise that resolves to the `apolloClient`.

**So far, that's not unlike what we had before, but that's okay!**

Because our `App` component is smarter this time around. It waits for the promise from the `ApolloProvider` to resolve, and once it does, it sets the `apolloClient` to its state. It then uses this `apolloClient` by passing it into a **second** `ApolloProvider`, which will wrap all of `App`'s children.

What this means is that if any of `App`'s children try to use one of Apollo's hooks such as `useQuery` or `useMutation`, those hooks will go up the component tree and find the second `ApolloProvider` with the real `apolloClient`. It won't go up any further and accidentally use first `ApolloProvider` with the promise.