# Vue SSR Boilerplate
Vue.js Server Side Rendering Boilerplate without Polluting Vuex


## Features:
* Doesn't dependent on Vuex. Putting every thing into Vuex is so ugly.
* Vuex is there, global states can still put into Vuex store.
* Customizable webpack config.
* Hot module replacement.
* Codes can run with or without SSR. Just use different `npm run` commands.
* And so on.


## Initialize
First, download or clone this project.

Then install npm packages via `npm install`.


## Start without SSR
```sh
npm run dev
```
I recommend develop in this mode at first. So you can focus on your view things,
not bother with server side things.


## Start with SSR
```sh
npm run dev:ssr
```
When your pages look fine, then you step into SSR mode to check the server side is OK.


## Some Example Pages
When you start the project, you can visit `http://localhost/8100` to look around.
Whenever you start with or without SSR, the URL is the same.


## How to Write Page
Every thing is the same as developing a SPA, except one thing, you need to define a
`prefetch` method in your component. `prefetch` must return a `Promise`,
the resolved result will be merge into `this.$data` during rendering.
The first argument of `prefetch` is the Vuex store object. so you can set some Vuex state in `prefetch`.

`src/views/Index.vue`:
```html
<template>
  <div class="foo">
    <p>Hello world!</p>
    <p>this.a: {{a}}</p>
    <p><router-link to="/foo">goto /foo</router-link></p>
    <p><router-link to="/page-not-exist">goto /page-not-exist</router-link></p>
    <p><router-link to="/show-error-page">goto /show-error-page</router-link></p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      a: 0
    };
  },

  prefetch() {
    return Promise.resolve({
      a: 123
    });
  },

  // will be called on server side. check your console
  created() {
    console.log(this.a); //eslint-disable-line
  },

  // won't run on server side
  beforeMount() {
    console.log(this.a); //eslint-disable-line
  }
};
</script>
```

`src/views/Foo.vue`:
```html
<template>
  <div class="foo">
    <p>this.a: {{a}}</p>
    <p>this.$store.state.count: {{$store.state.count}}</p>
    <p>Enviroment Variables Defined by webpack.DefinePlugin:</p>
    <pre>{{config}}</pre>
    <p><router-link to="/">goto /</router-link>
  </div>
</template>

<script>
export default {
  data() {
    return {
      a: 0,
      config: null
    };
  },

  prefetch(store) {
    return Promise.all([
      new Promise(resolve => {
        setTimeout(() => {
          resolve({
            a: 123
          });
        });
      }),

      store.dispatch('asyncIncrement')
    ]).then(([componentData]) => componentData);
  },

  // won't run on server side
  beforeMount() {
    console.log(this.a); //eslint-disable-line

    /*
    can not be defined in data(),
    because the TARGET is different between server side (TARGET: node) and client side (TARGET: web)
    and this will cause the client-side rendered virtual DOM tree not matching server-rendered content

    can not use object-shorthand, because the tokens will be replaced by webpack.DefinePlugin
    */
    this.config = JSON.stringify({
      DEBUG: DEBUG, //eslint-disable-line
      TARGET: TARGET, //eslint-disable-line
      VERSION: VERSION, //eslint-disable-line
      CONFIG: CONFIG //eslint-disable-line
    }, null, 2);
  }
};
</script>
```

`prefetch` is only effective on components which are defined as route components.
And `prefetch` is optional, you can omit it if the component don't need SSR.


## Handling Errors

### 404 Not Found
When route is not found on server side, the server will send a HTTP 404 status code, and with `dist/index.html` (yield by html-webpack-plugin from `src/index.html`) as payload. Thus, the page runs as if a SPA without SSR.

We define a catch-all routes in `src/router.js` when code is run in browser:
```js
import HTTP404 from './views/HTTP404.vue';

// ...

if (TARGET === 'web') {
  routes.push(
    // catch-all route must be placed at the last
    { path: '*', component: HTTP404 }
  );
}
```
So our SPA can serve a Not Found page for that request.

Your can check it at whatever URL that not exists like `http://localhost:8100/page-not-exist`


#### Why not rendering the 404 page on server side?
Because if we define the catch-all route on server side, then no 404 HTTP status code will be sent.
The search engine will handle it as a normal response.

And it let you resolve a 404 in different ways, such as, in navigation guard of vue-router.
Also, it simplified the server side code.


### 500 Internal Server Error
When `prefetch` return a rejected promise, the server will send a HTTP 500 status code, and response with `dist/index.html`.

In your component, you can handle it whatever you like.

`src/views/ShowErrorPage.vue`:
```html
<template>
  <div class="show-error-page">

  </div>
</template>

<script>
export default {
  data() {
    return {
      foo: 0
    };
  },

  prefetch() {
    return Promise.reject();
  },

  beforeMount() {
    this.prefetched.catch(() => {
      alert('500 Internal Server Error');
    });
  }
};
</script>
```
`this.prefetched` is the promise return by `prefetch` method.
`prefetch` will be called again during initializing the component in the browser.
Because the server won't preserve the failed data and rendering the page,
it give you the flexibility to handle it whatever you like.

Checkout `http://localhost:8100/show-error-page` to see effects.


## Build Distribution
```sh
npm run build
```
That's it.

Files will be output to `dist` folder. In `npm run dev:ssr` mode, files are output to `tmp` folder.


## Run in Production
```sh
node server.js
```


## Configuration
By default, the boilerplate provides two sets of config files.
`config/dev.js` is used in development mode, `config/default.js` is used in production mode.
You can override by
```sh
npm run dev --config=YOUR-CONFIG-FILE-NAME
npm run dev:ssr --config=YOUR-CONFIG-FILE-NAME
```
in development.

And
```sh
node server.js --config=YOUR-CONFIG-FILE-NAME
```
in production.

The `ssrPort` is the port number that the server-side listened on.

And we also defined some environment variables using webpack.DefinePlugin:
* `DEBUG`: `true` in development, `false` in production.
* `VERSION`: `version` in `package.json`.
* `TARGET`: `node` on server-side, `web` on client-side.
* `CONFIG`: `runtimeConfig` exported by `config/*` files.

## License
MIT. Fork and PR are welcome!