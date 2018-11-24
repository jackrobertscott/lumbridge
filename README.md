# Lumbridge

> 🏰 React application management made simple.

## Main Features

- `Router`: Easy to use route guards, enter and leave hooks, simplified route structure to improve maintainability.
- `Store`: Quick to setup, modular and scalable local state management.
- `Persistor`: Keep your application data syncronized easily.
- `Theme`: Functional styles which act like lego when building components.

## Usage

Using [npm](https://www.npmjs.com/package/lumbridge):

```shell
npm i --save lumbridge
```

Using yarn:

```shell
yarn add lumbridge
```

Import into a your app:

```js
import { Store } from 'lumbridge';

const authStore = Store.create({
  // code...
});
```

**Note:** the components will usually be created in their own files (e.g. `src/stores/authStore.js`) which can then be imported into a React component file.

## API

### Router

Structured and intuitive routing which allows you to easily create routes and manage the events related to those routes.

[See detailed docs.](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-router)

Considerations in design:

- Routing into a component can be aborted by checking data on a routes `enter` method.
- Prevent a user from routing away from a component (such as when a form has not been saved) with a route's `leave` method.
- Record and monitor all changes in the route location with the `change` method.
- Routes are smartly prioritised by showing only the route which was found be the closest to real path.
- Exact match routes are checked first and will have priority over non-exact routes.
- Each route has a key which you can use to generate a link which you may route to.

```js
import { Router } from 'lumbridge';
import { HomePage, LoginPage, SignUpPage } from '../components/pages';

/**
 * Define your routes in a regular JavaScript method.
 */
const authRouter = Router.create({
  routes: {
    home: {
      path: '/',
      exact: true,
      component: HomePage,
    },
    login: {
      path: '/login',
      component: LoginForm,
      enter: {
        before: () => userIsNotLoggedIn(),
      },
      leave: {
        before: () => saveDataBeforeLeaving(),
      },
    },
    signUp: {
      path: '/sign-up',
      component: SignUpPage,
      alias: ['hello', 'yellow'],
      enter: {
        before: () => userIsNotLoggedIn(),
      },
    },
  },
  change: {
    after: ({ match: { path } }) => {
      recordRouteForAnalytics(path);
    },
  },
});
```

- The routes can be easily inserted into a regular app component.
- Nesting components are not supported on purpose. Instead, we encourage you to make multiple router instances and nest them in your normal components.

```js
import authRouter from '../routers/authRouter';

/**
 * Add the routes into your React component like a regular component.
 */
const App = () => (
  <Dashboard>
    <authRouter.Routes />
  </Dashboard>
);
```

### Persistor

Easily communicate and interact with persistant data stores such as `localStorage`, GraphQL Servers, REST Servers, etc.

[See detailed docs.](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-persistor)

Considerations in design:

- Built to be flexible so you can use interface with multiple kinds of data stores.
- Seperates the data *requests* from the data *retrieval* so you can access the data in multiple locations.
- Data *scopes* allow multiple requests to depend on each other and update efficiently with minimal overhead.
- Easily integrate the data persistors into existing apps.

```js
import { Persistor } from 'lumbridge';
import { string, object } from 'yup';
import apolloClient from '../client';

/**
 * Create a persistor which will be the interface with the data source.
 */
const apolloPersistor = Persistor.create({
  validations: {
    query: string().required(),
    variables: object(),
  },
  handler: () => ({ query, variables }) => {
    return apolloClient.query({ query, variables })
      .then(({ data }) => data);
  },
});
```

Create instances of the persistor which you can listen to in your components.

```js
import React from 'react';
import { apolloPersistor } from '../persistors/apolloPersistor';

/**
 * Create an instance from the persistor, this will contain the common
 * request information (such as a GraphQL query).
 */
const meInstance = apolloPersistor.instance({
  map: () => ({
    query: `
      query($id: String) {
        me(id: $id) { name }
      }
    `,
  }),
  scopes: {
    user: true
  },
});

/**
 * Here is how you can use the instance in a React component using hooks.
 */
const MyProfile = ({ id }) => {
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const unwatch = meInstance.watch(({ data, loading, error }) => {
        setError(error);
        setLoading(loading);
        setData(data);
      });
    return () => unwatch();
  }, []); // fire only on mount and unmount (empty array)

  useEffect(() => {
    meInstance.execute({
      variables: { id },
    });
  }, [id]); // fire only when id changes

  return (
    <div>
      {data.me.name}
      {error && error.message}
      {loading}
    </div>
  );
};
```

### Store

Share and validate data in multiple places around an app.

[See detailed docs.](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-store)

Considerations in design:

- Access data in multiple locations throughout app without passing data through components.
- Get awesome data validation by using [Yup](https://www.npmjs.com/package/yup) the third party validation library.
- Is effective at managing form data.
- Updates state using `store.update({})` which is similar to how `this.setState({})` works, only partially updating the store.
- Reset the store by using `store.reset()` which will set the stores values back to their defaults.

```js
import { Store } from 'lumbridge';
import { string, boolean, object } from 'yup';

/**
 * Create a data source which can be accessed from anywhere in the app.
 */
const authStore = Store.create({
  schema: {
    token: {
      default: null,
    },
    userId: {
      default: null,
      validate: string(),
    },
    loggedIn: {
      default: false,
      validate: boolean().required(),
    },
    big: {
      default: null,
      validate: object({
        one: string().required(),
        two: string().required(),
      }),
    },
  },
  actions: {
    loginUser: store => ({ token, userId }) => {
      store.update({
        token,
        userId,
        loggedIn: Boolean(token && userId),
      });
    },
  },
});
```

You can then access the store in your components.

```js
import React from 'react';
import { authStore } from '../stores/authStore';
import { authRouter } from '../routers/authRouter';

/**
 * Normal stateless React component.
 */
const HeaderBar () => (
  <Menu>
    <Item onClick={() => authStore.reset() && authRouter.navigate('login')}>Logout</Item>
  </Menu>
);
```

Manage routes effectively with stores.

```js
import { Router } from 'lumbridge';
import { authStore } from '../stores/authStore';

/**
 * Use the store as a route guard.
 */
const authRouter = Router.create({
  routes: {
    home: {
      path: '/',
      exact: true,
      component: HomePage,
      enter: {
        before: () => authStore.state.loggedIn,
      },
    },
  },
});
```

You can also effectively handle forms with your stores.

```js
import React from 'react';
import { authStore } from '../stores/authStore';
import FieldError from '../components/FieldError';

/**
 * Normal stateless React component.
 */
const LoginForm = ({ onSubmit }) => (
  <authStore.Listen>
    {({ state, errors }) => (
      <form onSubmit={onSubmit}>
        <div>
          <input
            type="text"
            name="username"
            value={state.username}
            onChange={event => authStore.update({ username: event.target.value })}
          />
          {errors.username && <div>{errors.username}</div>}
        </div>
        <div>
          <authStore.Field name="password" />
          <!-- FieldError will only show when error present and the message will be the children -->
          <authStore.Error name="password" component={FieldError} />
        </div>
      </form>
    )}
  </authStore.Listen>
);

/**
 * Here is the same form but using React hooks.
 */
const LoginFormHooks = ({ onSubmit }) => {
  const [errors, setErrors] = useState({});
  const [state, setState] = useState({});
  
  useEffect(() => {
    const unwatch = authStore.watch(({ state, errors }) => {
      setErrors(errors)
      setState(state);
    });
    return () => unwatch();
  }, []); // fire only on mount and unmount (empty array)

  return (
    <form onSubmit={onSubmit}>
      <div>
        <input
          type="text"
          name="username"
          value={state.username}
          onChange={event => authStore.update({ username: event.target.value })}
        />
        {errors.username && <div>{errors.username}</div>}
      </div>
      <div>
        <authStore.Field name="password" />
        <authStore.Error name="password" component={FieldError} />
      </div>
    </form>
  );
};
```

### Theme

A new approach to styling components, taking a more functional approach.

[See detailed docs.](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-theme)

Considerations in design:

- Arrange your styles in groups of functional properties and use those groups to compose component styles.
- Write your styles as JavaScript objects which makes code reuse and refactoring simple.
- Default CSS is removed from all HTML elements so that you don't have to undo styles all the time.
- Properties values can only be strings and numbers to promote good code design.
- Mutations can be created to alter few of the properties and provides flexibility with styles.
- Actions are methods which can map component `props` onto mutations and styles.

```js
import { Theme } from 'lumbridge';

/**
 * Create a group of CSS properties and add some potential mutations.
 */
const paddings = Theme.create({
  defaults: { padding: '14px' },
  mutations: {
    tiny: { padding: '16px' },
    small: { padding: '16px' },
    medium: { padding: '16px' },
    big: { padding: '16px' },
  },
});

/**
 * Groups of multiple properties can be created as well.
 */
const fonts = Theme.create({
  defaults: {
    lineHeight: '1.15',
    fontFamily: 'inherit',
    fontSize: '100%',
    fontWeight: 'normal',
  },
  mutations: {
    primary: {
      fontSize: '20px',
      fontWeight: 'bold',
    },
    mono: {
      fontFamily: 'monospace',
    },
    active: {
      color: 'blue',
    },
  },
  actions: {
    active: theme => ({ active }) =>
      theme.mutation({
        active,
        mono: !active,
      }),
  },
});
```

Import the themes and use them to construct new components.

```js
import React from 'react';
import { Theme } from 'lumbridge';
import { paddings, fonts } from '../themes';

/**
 * Create a styled React component with your themes.
 */
const Wrap = Theme.combo({
  as: 'div',
  theme: [
    paddings.mutations({ big: true }),
    fonts.mutations({ primary: true }).actions({ active: true }),
  ],
  extra: {
    backgroundColor: 'green',
  },
});

/**
 * Use the styled react component like normal, or get create an element
 * directly from the theme.
 */
const StyledComponent = ({ active }) => (
  <Wrap active={active}>
    <fonts.Element as="span" mutation="mono">
      Hello world!
    </fonts.Element>
  </Wrap>
);
```

## Packages

- [lumbridge](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge)
- [lumbridge-persistor](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-persistor)
- [lumbridge-router](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-router)
- [lumbridge-store](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-store)
- [lumbridge-theme](https://github.com/jackrobertscott/lumbridge/tree/master/packages/lumbridge-theme)
