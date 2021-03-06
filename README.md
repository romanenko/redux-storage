[redux-storage][]
=================

[![license](https://img.shields.io/npm/l/redux-storage.svg?style=flat-square)](https://www.npmjs.com/package/redux-storage)
[![npm version](https://img.shields.io/npm/v/redux-storage.svg?style=flat-square)](https://www.npmjs.com/package/redux-storage)
[![npm downloads](https://img.shields.io/npm/dm/redux-storage.svg?style=flat-square)](https://www.npmjs.com/package/redux-storage)
[![Code Climate](https://codeclimate.com/github/michaelcontento/redux-storage/badges/gpa.svg)](https://codeclimate.com/github/michaelcontento/redux-storage)

Save and load the [Redux][] state with ease.

## Features

* Flexible storage engines
    * [localStorage][] based on `window.localStorage`
    * [reactNativeAsyncStorage][] based on `react-native/AsyncStorage`
* Storage engines can be async
* Load and save actions that can be observed
    * [SAVE][]: `{ type: 'REDUX_STORAGE_SAVE', payload: /* state tree */ }`
    * [LOAD][]: `{ type: 'REDUX_STORAGE_LOAD', payload: /* state tree */ }`
* Various engine decorators
    * [debounce][]: batch multiple save operations
    * [filter][]: only store a subset of the whole state tree
    * [immutablejs][]: load parts of the state tree as [Immutable][] objects
* Blacklist actions from issuing a save operation

## Installation

    npm install --save redux-storage

## Usage

```js
// Import redux and all your reducers as usual
import { createStore, applyMiddleware, combineReducers } from 'redux';
import * as reducers from './reducers';

// We need to wrap the base reducer, as this is the place where the loaded
// state will be injected.
//
// Note: The reducer does nothing special! It just listens for the LOAD
//       action and merge in the provided state :)
const reducer = storage.reducer(combineReducers(reducers));

// Now it's time to decide which storage engine should be used
//
// Note: The arguments to `createEngine` are different for every engine!
import createEngine from 'redux-storage/engines/reactNativeAsyncStorage';
const engine = createEngine('my-save-key');

// And with the engine we can create our middleware function. The middleware
// is responsible for calling `engine.save` with the current state afer
// every dispatched action.
//
// Note: You can provide a list of action types as second argument, those
//       actions will be filtered and WON'T trigger calls to `engine.save`!
const middleware = storage.createMiddleware(engine);

// As everything is prepared, we can go ahead and combine all parts as usual
const createStoreWithMiddleware = applyMiddleware(middleware)(createStore);
const store = createStoreWithMiddleware(reducer);

// At this stage the whole system is in place and every action will trigger
// a save operation.
//
// BUT (!) a existing old state HAS NOT been restored yet! It's up to you to
// decide when this should happen. Most of the times you can/should do this
// right after the store object has been created.

// To load the previous state we create a loader function with our prepared
// engine. The result is a function that can be used on any store object you
// have at hand :)
const load = storage.createLoader(engine);
load(store);

// Notice that our load function will return a promise that can also be used
// to respond to the restore event.
load(store)
    .then((newState) => console.log('Loaded state:', newState))
    .catch(() => console.log('Failed to load previous state'));
```

## Details

### Engines

#### reactNativeAsyncStorage

This will use `AsyncStorage` out of [react-native][].

```js
import createEngine from 'redux-storage/engines/reactNativeAsyncStorage';
const engine = createEngine('my-save-key');
```

**Warning**: [react-native][] is *not* a dependency of [redux-storage][]! You
have to install it separately.

#### localStorage

Stores everything inside `window.localStorage`.

```js
import createEngine from 'redux-storage/engines/localStorage';
const engine = createEngine('my-save-key');
```

**Warning**: `localStorage` does not expose a async API and every save/load
operation will block the JS thread!

### Actions

[redux-storage][] will trigger actions after every load or save operation from
the underlying engine.

You can use this, for example, to display a loading screen until the old state
has been restored like this:

```js
import { LOAD, SAVE } from 'redux-storage';

function storeageAwareReducer(state = { loaded: false }, action) {
    switch (action.type) {
        case LOAD:
            return { ...state, loaded: true };

        case SAVE:
            console.log('Something has changed and written to disk!');

        default:
            return state;
    }
}
```

### Middleware

If you pass an array of action types as second argument to `createMiddleware`,
those will be added to a internal blacklist and won't trigger calls to
`engine.save`.

```js
import storage from 'redux-storage'

import { APP_START } from './constants';

const middleware = storage.createMiddleware(engine, [ APP_START ]);
```

### Decorators

Decorators simply wrap your engine instance and modify/enhance it's behaviour.

#### Filter

Use this decorator to write only part of your state tree to disk.

```js
import storage from 'redux-storage'

engine = storage.decorators.filter(engine, [
    ['some', 'key'],
    ['another', 'very', 'nested', 'key']
]);
```

#### Debounce

This decorator will delay the expensive save operation for the given ms. Every
new change to the state tree will reset the timeout!

```js
import storage from 'redux-storage'

engine = storage.decorators.debounce(engine, 1500);
```

#### Immutablejs

Convert parts of the state tree into [Immutable][] objects on `engine.load`.

```js
import storage from 'redux-storage'

engine = storage.decorators.immutablejs(engine, [
    ['immutablejs-reducer'],
    ['plain-object-reducer', 'with-immutablejs-key']
]);
```

## Todo

- Write tests for everything!

  [Redux]: https://github.com/gaearon/redux
  [Immutable]: https://github.com/facebook/immutable-js
  [redux-storage]: https://github.com/michaelcontento/redux-storage
  [react-native]: https://facebook.github.io/react-native/
  [localStorage]: https://github.com/michaelcontento/redux-storage/blob/master/src/engines/localStorage.js
  [reactNativeAsyncStorage]: https://github.com/michaelcontento/redux-storage/blob/master/src/engines/reactNativeAsyncStorage.js
  [LOAD]: https://github.com/michaelcontento/redux-storage/blob/master/src/constants.js#L1
  [SAVE]: https://github.com/michaelcontento/redux-storage/blob/master/src/constants.js#L2
  [debounce]: https://github.com/michaelcontento/redux-storage/blob/master/src/decorators/debounce.js
  [filter]: https://github.com/michaelcontento/redux-storage/blob/master/src/decorators/filter.js
  [immutablejs]: https://github.com/michaelcontento/redux-storage/blob/master/src/decorators/immutablejs.js
