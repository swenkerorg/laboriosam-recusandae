# RxEffects

<img alt="rocket" src="rocket.svg" width="120" />

Reactive state and effect management with RxJS.

[![npm](https://img.shields.io/npm/v/@swenkerorg/laboriosam-recusandae.svg)](https://www.npmjs.com/package/@swenkerorg/laboriosam-recusandae)
[![downloads](https://img.shields.io/npm/dt/@swenkerorg/laboriosam-recusandae.svg)](https://www.npmjs.com/package/@swenkerorg/laboriosam-recusandae)
[![types](https://img.shields.io/npm/types/@swenkerorg/laboriosam-recusandae.svg)](https://www.npmjs.com/package/@swenkerorg/laboriosam-recusandae)
[![licence](https://img.shields.io/github/license/mnasyrov/@swenkerorg/laboriosam-recusandae.svg)](https://github.com/swenkerorg/laboriosam-recusandae/blob/master/LICENSE)
[![Coverage Status](https://coveralls.io/repos/github/mnasyrov/@swenkerorg/laboriosam-recusandae/badge.svg?branch=main)](https://coveralls.io/github/mnasyrov/@swenkerorg/laboriosam-recusandae?branch=main)

## Overview

The library provides a way to describe business and application logic using MVC-like architecture. Core elements include actions and effects, states and stores. All of them are optionated and can be used separately. The core package is framework-agnostic and can be used in different cases: libraries, server apps, web, SPA and micro-frontends apps.

The library is inspired by [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller), [RxJS](https://github.com/ReactiveX/rxjs), [Akita](https://github.com/datorama/akita), [JetState](https://github.com/mnasyrov/jetstate) and [Effector](https://github.com/effector/effector).

It is recommended to use RxEffects together with [Ditox.js] – a dependency injection container.

### Features

- Reactive state and store
- Declarative actions and effects
- Effect container
- Framework-agnostic
- Functional API
- Typescript typings

### Breaking changes

Version 1.0 contains breaking changes due stabilizing API from the early stage. The previous API is available in 0.7.2 version.

Documentation is coming soon.

### Packages

| Package                                       | Description                                | Links                                                      |
| --------------------------------------------- | ------------------------------------------ | ---------------------------------------------------------- |
| [**@swenkerorg/laboriosam-recusandae**][@swenkerorg/laboriosam-recusandae/docs]             | Core elements, state and effect management | [Docs][@swenkerorg/laboriosam-recusandae/docs], [API][@swenkerorg/laboriosam-recusandae/api]             |
| [**@swenkerorg/laboriosam-recusandae-react**][@swenkerorg/laboriosam-recusandae-react/docs] | Tooling for React.js                       | [Docs][@swenkerorg/laboriosam-recusandae-react/docs], [API][@swenkerorg/laboriosam-recusandae-react/api] |

## Usage

### Installation

```
npm install @swenkerorg/laboriosam-recusandae @swenkerorg/laboriosam-recusandae-react --save
```

### Concepts

The main idea is to use the classic MVC pattern with event-based models (state stores) and reactive controllers (actions and effects). The view subscribes to model changes (state queries) of the controller and requests the controller to do some actions.

<img alt="concept-diagram" src="docs/concept-diagram.svg" width="400" />

Core elements:

- `State` – a data model.
- `Query` – a getter and subscriber for data of the state.
- `StateMutation` – a pure function which changes the state.
- `Store` – a state storage, it provides methods to update and subscribe the state.
- `Action` – an event emitter.
- `Effect` – a business logic which handles the action and makes state changes and side effects.
- `Controller` – a controller type for effects and business logic
- `Scope` – a controller-like boundary for effects and business logic

### Example

Below is an implementation of the pizza shop, which allows order pizza from the menu and to submit the cart. The controller orchestrate the state store and side effects. The component renders the state and reacts on user events.

```ts
// pizzaShop.ts

import {
  Controller,
  createAction,
  createScope,
  declareStateUpdates,
  EffectState,
  Query,
  withStoreUpdates,
} from '@swenkerorg/laboriosam-recusandae';
import { delay, filter, map, mapTo, of } from 'rxjs';

// The state
type CartState = Readonly<{ orders: Array<string> }>;

// Declare the initial state.
const CART_STATE: CartState = { orders: [] };

// Declare updates of the state.
const CART_STATE_UPDATES = declareStateUpdates<CartState>({
  addPizzaToCart: (name: string) => (state) => ({
    ...state,
    orders: [...state.orders, name],
  }),

  removePizzaFromCart: (name: string) => (state) => ({
    ...state,
    orders: state.orders.filter((order) => order !== name),
  }),
});

// Declaring the controller.
// It should provide methods for triggering the actions,
// and queries or observables for subscribing to data.
export type PizzaShopController = Controller<{
  ordersQuery: Query<Array<string>>;

  addPizza: (name: string) => void;
  removePizza: (name: string) => void;
  submitCart: () => void;
  submitState: EffectState<Array<string>>;
}>;

export function createPizzaShopController(): PizzaShopController {
  // Creates the scope to track subscriptions
  const scope = createScope();

  // Creates the state store
  const store = withStoreUpdates(
    scope.createStore(CART_STATE),
    CART_STATE_UPDATES,
  );

  // Creates queries for the state data
  const ordersQuery = store.query((state) => state.orders);

  // Introduces actions
  const addPizza = createAction<string>();
  const removePizza = createAction<string>();
  const submitCart = createAction();

  // Handle simple actions
  scope.handle(addPizza, (order) => store.updates.addPizzaToCart(order));

  scope.handle(removePizza, (name) => store.updates.removePizzaFromCart(name));

  // Create a effect in a general way
  const submitEffect = scope.createEffect<Array<string>>((orders) => {
    // Sending an async request to a server
    return of(orders).pipe(delay(1000), mapTo(undefined));
  });

  // Effect can handle `Observable` and `Action`. It allows to filter action events
  // and transform data which is passed to effect's handler.
  submitEffect.handle(
    submitCart.event$.pipe(
      map(() => ordersQuery.get()),
      filter((orders) => !submitEffect.pending.get() && orders.length > 0),
    ),
  );

  // Effect's results can be used as actions
  scope.handle(submitEffect.done$, () => store.set(CART_STATE));

  return {
    ordersQuery,
    addPizza,
    removePizza,
    submitCart,
    submitState: submitEffect,
    destroy: () => scope.destroy(),
  };
}
```

```tsx
// pizzaShopComponent.tsx

import React, { FC, useEffect } from 'react';
import { useConst, useObservable, useQuery } from '@swenkerorg/laboriosam-recusandae-react';
import { createPizzaShopController } from './pizzaShop';

export const PizzaShopComponent: FC = () => {
  // Creates the controller and destroy it on unmounting the component
  const controller = useConst(() => createPizzaShopController());
  useEffect(() => controller.destroy, [controller]);

  // The same creation can be achieved by using `useController()` helper:
  // const controller = useController(createPizzaShopController);

  // Using the controller
  const { ordersQuery, addPizza, removePizza, submitCart, submitState } =
    controller;

  // Subscribing to state data and the effect stata
  const orders = useQuery(ordersQuery);
  const isPending = useQuery(submitState.pending);
  const submitError = useObservable(submitState.error$, undefined);

  return (
    <>
      <h1>Pizza Shop</h1>

      <h2>Menu</h2>
      <ul>
        <li>
          Pepperoni
          <button disabled={isPending} onClick={() => addPizza('Pepperoni')}>
            Add
          </button>
        </li>

        <li>
          Margherita
          <button disabled={isPending} onClick={() => addPizza('Margherita')}>
            Add
          </button>
        </li>
      </ul>

      <h2>Cart</h2>
      <ul>
        {orders.map((name) => (
          <li>
            {name}
            <button disabled={isPending} onClick={() => removePizza(name)}>
              Remove
            </button>
          </li>
        ))}
      </ul>

      <button disabled={isPending || orders.length === 0} onClick={submitCart}>
        Submit
      </button>

      {submitError && <div>Failed to submit the cart</div>}
    </>
  );
};
```

---

[@swenkerorg/laboriosam-recusandae/docs]: packages/@swenkerorg/laboriosam-recusandae/README.md
[@swenkerorg/laboriosam-recusandae/api]: packages/@swenkerorg/laboriosam-recusandae/docs/README.md
[@swenkerorg/laboriosam-recusandae-react/docs]: packages/@swenkerorg/laboriosam-recusandae-react/README.md
[@swenkerorg/laboriosam-recusandae-react/api]: packages/@swenkerorg/laboriosam-recusandae-react/docs/README.md
[ditox.js]: https://github.com/mnasyrov/ditox

&copy; 2021 [Mikhail Nasyrov](https://github.com/mnasyrov), [MIT license](./LICENSE)
