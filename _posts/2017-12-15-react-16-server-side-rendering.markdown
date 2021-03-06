---
layout: post
title:  "Server Side Rendering in ReactJS - from v15 to v16"
page-section: Blog
date:   2017-12-15 14:40:00
categories:
- coding
- reactjs
- ssr
- devlog
images: 
- url: /images/blog/2017-12-15-react-16-server-side-rendering/react-ssr.jpg
  alt: React v16 - Server Side Rendering (SSR)
thumbnail:
- url: /images/blog/2017-12-15-react-16-server-side-rendering/thumb/react-ssr.jpg
  alt: React v16 - Server Side Rendering (SSR)
---
<p class="text-center">This was a year of big changes in the ReactJS world, with the coming of a new version, v16. And one of those affected the Server Side Rendering scripts used in the old version, v15.</p>

<!--more-->
![React v16 SSR!](/images/blog/2017-12-15-react-16-server-side-rendering/thumb/react-ssr.jpg)

# How SSR Works In React 15
First, let's have a look of how SSR works in React 15. To do so, it is necessary to run a Node server (Express in my case), and run `renderToString` in order to get the rendered elements and include them as HTML when ending the response.

So, it should start with the store construction, in order to create the initial view, and check any routing match:

```
app.use(function(req, res){
  // Create store with the middlewares that you need, the `createStore` method from Redux and your reducers
  const store = applyMiddleware(multi, thunkMiddleware, apiMiddleware)(createStore)(reducers);

  // Check any match with the given routes and current request location
  match({ routes: appRoutes, location: req.url }, function(err, redirectLocation, renderProps){
    
    // Handle server error
    if(err) {
      console.error(err);
      return res.status(500).end('Internal server error');
    }

    // If no match based on the props for the routing context, then give 404
    if(!renderProps)
      return res.status(404).end('Not found');

    ...
```

Next, it is necessary to start provide the Routing components with the given context in renderProps and return the HTML that will be filled with the initialState and the rendered React Components, from the `ReactDOM.renderToString()` method:

```
  ...

  function renderView() {
    // Start initial view with the store data and routing context
    const InitialView = (
      <Provider store={store}>
        <RouterContext {...renderProps} />
      </Provider>
    );

    // Render ReactJS Components
    const componentHTML = ReactDOM.renderToString(InitialView);      

    // Get initial state from store to append in the DOM
    const initialState = store.getState();

    // Set HTML with the initial state and rendered components
    const HTML = `
      <!DOCTYPE html>
        <head>
          <title>This is a SSR post</title>
          <script>
            window.__INITIAL_STATE__ = ${JSON.stringify(initialState)};
          </script>
        </head>
        <body>
          <div id="site-canvas"><div>${componentHTML}</div></div>
        </body>
       </html>`;
    return HTML;
  }
```

Have in mind that it is necessary to have the `<Provider />` component as a `<Router />` ancestor in order to work.
 
In the end, the `renderView` function is called after fetching the components data, sending the final HTML to the client:

```
  ...
  
  fetchComponentData(store.dispatch, renderProps.components, renderProps.params)
      .then(renderView)
      .then(html => res.end(html))
      .catch(err => res.json({message:err.message}));

}); // Closing `app.use()`
```

About this last line of code, you should have a `needs` variable inside each parent component. In my way of organizing the components, I create a wrapper component called `<App />` that wraps all components, and after that wrapper, there is a component for each route (e.g. `<AppsPage />`). Inside each page component, there is a `needs` variable with all the actions that should be despatched before sending the final HTML to the client:
```
AppsPage.needs = [    
    fetchConfig,
    fetchApps
];
```

and my routing looks like so:

```
<Route path="/" name="Index" component={App}>
    <Route name="Applications" path="Apps" component={AppsPage} />
</Route>
```

# How SSR Works In React 16

### render() Becomes hydrate()
The first change to do in your code is to use hydrate() instead of render(). You can still use `render()`, but it will become deprecated in React v17, but I would highly recomend use the new method in order to avoid warnings. So now, your code should be:

```
import { hydrate } from "react-dom";

...

ReactDOM.hydrate(
  <Provider store={store}>
    // Custom component with the react-router v4 routes centralized in a single component
    <CustomRoutes />
  </Provider>
  , document.getElementById('site-canvas'));
```

### New way of matching routes
So now, there is no `match()` method from react-router to help us in matching any routes. I will not explain it since there are other websites that do that better then me.

In order to match the routes, create a new function inside the `app.use()` context:

```
import { StaticRouter, matchPath } from 'react-router-dom';

...

app.use(function(req, res){
  // Create store with the middlewares that you need, the `createStore` method from Redux and your reducers
  const store = applyMiddleware(multi, thunkMiddleware, apiMiddleware)(createStore)(reducers);
  
  // These variables are going to be used in the end, in order to have enought information to fetch the state from the rendered components
  let params = {},
      components = [];

  // Match routes
  function routerMatch(req, routes = null) {
    // For each route in the provided variable
    routes.map(route => {
  
      // Check for any hierarchy. If so, use recursive
      if(route.routes && !route.path){
        routerMatch(req, route.routes);

      }else{
        // Check for any match
        const match = matchPath(req.path, route);
        
        // If any, assign `params` to the object
        // And push the component to the `components` array
        if (match && components.length==0){
          Object.assign(params, match.params);
          components.push(route.component);
        }
      }
    });
  }

  ...

```

In this new version, `react-router-dom`, the new `react-router` package, instead of using the  `<RouterContext />`, we should use `<StaticRouter />` with all the routes as children.

So now, in the `renderView()` function:

```
  function renderView() {
    const context = {};

    const InitialView = (
      <Provider store={store}>
        <StaticRouter location={req.url} context={context}>
            <RoutesObj />
        </StaticRouter>
      </Provider>
    );

    ...
```

First, we should provite store to the provider in order to share it with the whole app. Then, we provide the current url and the context to the `StaticRouter`, in order to keep track of the current location and also detect any redirects.

In the end, provide any routes that will be part of your app.


Everything keeps almost the same:

```
    ...

    const componentHTML = ReactDOMServer.renderToString(InitialView);

    const initialState = store.getState();

    // Set HTML with the initial state and rendered components
    const HTML = `
      <!DOCTYPE html>
        <head>
          <title>This is a SSR post</title>
          <script>
            window.__INITIAL_STATE__ = ${JSON.stringify(initialState)};
          </script>
        </head>
        <body>
          <div id="site-canvas"><div>${componentHTML}</div></div>
        </body>
       </html>`;
    return HTML;
  }

  ...
```

After creating the HTML, we search for a match with the router configuration and fetch all data from the `components` and `params` variables:

```
  ...

  // Get any match from the router configuration (`routesServer`)
  routerMatch(req, routesServer);

  // Fetch data from `needs` array
  fetchComponentData(store.dispatch, components, params)
      .then(renderView)
      .then(html => res.end(html))
      .catch(err => res.json({message:err.message}));
  }

}); // Closing `app.use()`
```

Keep in mind that the `routesServer` must be a configuration array, just like in [react-router-config](https://github.com/ReactTraining/react-router/tree/master/packages/react-router-config){:target="_blank"}{:rel="external"}. 

**The `matchPath`, from `react-router`, will not work directly with Router components.**

### Why not renderToNodeStream?
React v16 also supports rendering from a Node stream using [renderToNodeStream](https://reactjs.org/docs/react-dom-server.html#rendertonodestream){:target="_blank"}{:rel="external"}, but in this case, there is a gotcha, at least for some projects that I'm developing.

If you are injecting dynamic data in the `HTML` constant, such as meta information, CSS or script files, then this method will not work since there is not possible to embed a Readable stream as an element in a component, just like we are doing in the `HTML` definition with the `${componentHTML}`. This is only possible with strings returned from the `renderToString()`.

### In the end...
Keep in mind that there are a few changes that must be performed in the client-side code, such as the routes configuration that should match the new `react-router` version and documentation.

These are major changes in the new version of React that in some cases, require a major refactor of your whole app, but at least SSR has only a few changes, based on the "older ways".

I hope that this article helps you migrating your code to the new React, and be free to comment in the section bellow with sugestions, questions and other solutions that would improve our code base. 

# Full code
Since the project repos are private for now, here I provide you the code for both versions, with comments to help understand each line of code.

## React v15
```
...

//
//  React Server Render
//
app.use(function(req, res){
  // Create store with the middlewares that you need, the `createStore` method from Redux and your reducers
  const store = applyMiddleware(multi, thunkMiddleware, apiMiddleware)(createStore)(reducers);

  // Check any match with the given routes and current request location
  match({ routes: appRoutes, location: req.url }, function(err, redirectLocation, renderProps){
    
    // Handle server error
    if(err) {
      console.error(err);
      return res.status(500).end('Internal server error');
    }

    // If no match based on the props for the routing context, then give 404
    if(!renderProps)
      return res.status(404).end('Not found');

    function renderView() {
      const InitialView = (
        <Provider store={store}>
          <RouterContext {...renderProps} />
        </Provider>
      );
      // Render ReactJS Components
      const componentHTML = ReactDOM.renderToString(InitialView);      

      // Get initial state from store to append in the DOM
      const initialState = store.getState();

      // Set the JS Template to return
      const HTML = `
      <!DOCTYPE html>
        <head>
          <title>This is a SSR post</title>
          <script>
            window.__INITIAL_STATE__ = ${JSON.stringify(initialState)};
          </script>
        </head>
        <body>
          <div id="site-canvas"><div>${componentHTML}</div></div>
        </body>
       </html>`;

       return HTML;
    }
  
    // Fetch data from `needs` array
    fetchComponentData(store.dispatch, renderProps.components, renderProps.params)
      .then(renderView)
      .then(html => res.end(html))
      .catch(err => res.json({message:err.message}));
  });
});

...
```

## React v16
```

// Libs
import React from 'react';
import ReactDOMServer from 'react-dom/server';
import { StaticRouter, matchPath } from 'react-router-dom';
import { Provider } from 'react-redux';
import { createStore } from 'redux';

// Components
import RoutesObj, {routesServer} from '../public/assets/scripts/routes/routes';

// Utils
import fetchComponentData  from './utils/fetchComponentData';

...

//
//  React Server Render
//
app.use(function(req, res){
  
  // Handle the request for the favicon
  // Otherwise, the `/*` route for the 404 will match in this case and break the routing
  if (req.url === '/favicon.ico') {
    res.writeHead(200, {'Content-Type': 'image/x-icon'} );
    res.end();
    console.log('favicon requested');
    return;

  }else{
    // Create store with the middlewares that you need, the `createStore` method from Redux and your reducers
    const store = applyMiddleware(multi, thunkMiddleware, apiMiddleware)(createStore)(reducers);

     // These variables are going to be used in the end, in order to have enought information to fetch the state from the rendered components
    let params = {},
      components = [];

    /**
     * Get match of routes
     */
    function routerMatch(req, routes = null) {

      routes.map(route => {

        if(route.routes && !route.path){
          routerMatch(req, route.routes);

        }else{
          // use `matchPath` here
          const match = matchPath(req.path, route);
          debug("Match of "+req.path+":",match);

          if (match && components.length==0){
            Object.assign(params, match.params);
            components.push(route.component);
          }
        }
      });
    }

    /**
     * Render view with data
     */
    function renderView() {
      // This context object contains the results of the render
      const context = {};

      const InitialView = (
        <Provider store={store}>
          <StaticRouter location={req.url} context={context}>
              <RoutesObj />
          </StaticRouter>
        </Provider>
      );
      
      // context.url will contain the URL to redirect to if a <Redirect> was used
      if (context.url) {
        res.redirect(context.url);
        return;
      }

      // Render ReactJS Components
      const componentHTML = ReactDOMServer.renderToString(InitialView);      
      
      // Get initial state from store to append in the DOM
      const initialState = store.getState();
        
      // Set the JS Template to return
      const HTML = `
      <!DOCTYPE html>
        <head>
          <title>This is a SSR post</title>
          <script>
            window.__INITIAL_STATE__ = ${JSON.stringify(initialState)};
          </script>
        </head>
        <body>
          <div id="site-canvas"><div>${componentHTML}</div></div>
        </body>
       </html>`;

       return HTML;
    }

    // Get any match from the router configuration (`routesServer`)
    routerMatch(req, routesServer);
  
    // Fetch data from `needs` array
    fetchComponentData(store.dispatch, components, params)
      .then(renderView)
      .then(html => res.end(html))
      .catch(err => res.json({message:err.message}));
  }
});

...
```