# Must predictable state changes come at a high cost?

In state management of a frontend app, how to change states has been a vital problem since the early days. Today's widely accepted state management libraries have similar ways of getting states but different ways of changing states. For each of the libraries, one of the aspects developers care about most is how predictable state changes are.

After I dealt with various state changes using various libraries, I realized one problem, predictability comes at a high cost in today's practice. Then, I started to ask the question of this article's title and to expect to explore a better practice, if possible.

It has been challenging for me to review the firmly accepted ideas in my head to find out what might not be good enough. And now, I'd like to share some of my thoughts with you to look deeper into state changes together in terms of predictability.

## What does it mean by predictable?

First things first, predictable state changes mean state changes are easy to reason about. That is, in other words, what, why, when, and how states are changed are easy to understand.

Previously in MVC pattern, state changes are pretty hard to reason about. Every model hosts a state on its own and emits events of its state changes. When a model gets interested in a state change in some model, it subscribes to the event of that state change in that model. As a result, models get massively interconnected. Any state change may lead to "a tangled weave of data flow". The understanding of what, why, when, and how states are changed gets lost.

With this knowledge, let's check out how predictable are state changes in today's widely accepted state management libraries. Thinking that the battle of state management is more intense in React than in other frameworks, I'd take some of the typical state management libraries in React as the typical examples for the discussion here, which include Redux, Recoil, and MobX. (Please grab yourself some basic knowledge about the libraries if needed.)

In Redux, a procedure of state changes is a unidirectional data flow that actions are dispatched, reducers respond, then new states are produced. The single direction enforces, no matter whether an action is a plain action that only gets itself dispatched or a thunk action that gets more actions dispatched, once it finishes getting dispatched, no other actions will be dispatched anywhere downstream. Therefore, what, why, and when states are changed become easy to understand. As for how states are changed, if only plain actions are dispatched, it's determined by how responding reducers are defined. If some thunk actions are dispatched, it's determined by both how thunk actions are defined and how responding reducers are defined. Because reducers are pure functions modeled after math functions that don't have side effects, the major part of how states are changed becomes easy to understand. As a result, state changes in Redux become quite predictable.

In Recoil, a procedure of state changes is a combination of directly invoking state setters. The side effect of a state setter is restricted and doesn't get any other state setters invoked. Therefore, what, why, and when states are changed become easy to understand. As for how states are changed, it's determined by how procedures of invoking state setters are defined. Usually, procedures of invoking state setters can be encapsulated as hooks in Recoil, but hooks are not as easy to understand as pure functions because of side effects. So, state changes are less predictable in Recoil than in Redux.

In MobX, a procedure of state changes is a combination of directly assigning values to states that are marked observable. The side effect of an assignment is restricted and doesn't get any value assigned to other observable states. Therefore, what, why, and when states are changed become easy to understand. As for how states are changed, it's determined by how procedures of assigning values to observable states are defined. Usually, procedures of assigning values to observable states can be encapsulated as class methods in MobX, but class methods are not as easy to understand as pure functions because of side effects. So, state changes are less predictable in MobX than in Redux.

To sum up, Redux stands out in terms of predictability.

## The drawback of Redux

Though, Redux is not perfect and has a drawback. If we take a closer look at its unidirectional data flow, `event -> action -> reducer -> state`, it's lengthy. No matter how simple a state change is, always at least one action and at least one reducer are involved. In comparison, a state change in either Recoil or MobX goes much easier. The lengthiness dramatically increases the cost of use in Redux.

But, is the lengthiness necessary for predictability? One thing I notice is, it is the single direction of the data flow making what, why, and when states are changed easy to understand. Another noticeable thing is, it is that reducers are pure functions making how states are changed easy to understand. So, the fact is, actions don't contribute to predictability. Then, the lengthiness can be resolved without affecting predictability.

However, if so, why did actions come into being? I'd trace back Redux to its predecessor, Flux. In Flux, the unidirectional data flow is `event -> action -> dispatcher -> reducer -> state`. Although this is even lengthier because of the presence of the singleton dispatcher, every part of this still makes sense because the sub-flow, `action -> dispatcher -> reducer`, makes it doable for a reducer to wait for other reducers. As for what a reducer waits for other reducers to do, it's almost always for deriving data across states represented by reducers. But, when Redux inherits the core idea from Flux, selectors are added for deriving data, then the dispatcher is discarded so it's no longer doable for a reducer to wait for other reducers. As a result, actions no longer contribute to deriving data so become sort of pointless in terms of either functionality or predictability.

## Possibility of predictability at a low cost

Now that the lengthiness is not necessary for predictability, predictable state changes don't have to come at a high cost. Thus, it would be possible to design a new state management library that achieves predictable state changes at a low cost. As the need of managing states is quite common, a new library of this kind would bring huge benefits to the future frontend development. Let's try exploring it as follows.

First, I'd circle back a bit to Redux, Recoil, and MobX. In each of those libraries, what, why, and when states are changed are easy to understand because (1) state changes don't get other unexpected state changes triggered. In Redux, how states are changed is easy to understand because (2) it's mainly defined by reducers as pure functions. That is, these two points are just sufficient for predictable state changes. Anything else is not important in terms of predictability so can be simplified. Thus, one of the feasible ways to design a solution for the goal would be, state changes don't get other unexpected state changes triggered and meanwhile **only** involve reducers.

Then, let's think about the form of a reducer for changing one state in this solution. On one hand, it accepts an old state and produces a new state. On the other hand, it accepts payloads directly as per events. So, what it looks like would be:

```ts
function highlight(state: StateA): StateA {
  return { ...state, highlighted: true };
}

function setText(state: StateA, text: string): StateA {
  return { ...state, text };
}

function setHint(state: StateA, hint: string, hintColor: Color): StateA {
  return { ...state, hint, hintColor };
}
```

Next, let's think about the form of a reducer for changing multiple states in this solution. On one hand, it accepts multiple old states and produces multiple new states in the same order. On the other hand, it accepts payloads directly as per events. So, what it looks like would be:

```ts
function setTextAndSwitchMode(
  [stateA, stateB]: [StateA, StateB],
  text: string,
  mode: Mode
): [StateA, StateB] {
  return [
    { ...stateA, text },
    { ...stateB, mode },
  ];
}
```

Afterward, let's see the behavior of the procedure involving these reducers to change states. The point here is, the procedure doesn't get other unexpected state changes triggered. As reducers are simply pure functions, what the procedure looks like would simply be:

```ts
const oldStateA = getState(keyOfStateA);
const newStateA = highlight(oldStateA);
setState(keyOfStateA, newStateA);
```

With the procedure encapsulated as a reusable function such as `operate`, the procedure can evolve into:

```ts
operate(keyOfStateA, highlight);

operate(keyOfStateA, setText, 'Lorem ipsum');

operate(keyOfStateA, setHint, 'Ut enim', Color.GOLD);

operate([keyOfStateA, keyOfStateB], setTextAndSwitchMode, 'Duis', Mode.QUIET);
```

Furthermore, taking a closer look at the first param of a `operate`, it would be found that the role of a `keyOf...` is to identify a state. But declaring a series of unique strings for that is very costly. To lower this cost, one of the approaches is to make the most of something that already exists necessarily in state management, which, then, reminds me of state definitions.

A state definition requires (1) a default state value and (2) a state type. And now, a state definition is going to be made able to (3) identify the state it defines. So, what would it look like? After thinking for a while, there, in JS, just happen to be a matching concept that can fulfill those three points all together, which is Plain Old JavaScript Object(POJO). A POJO as a state definition is able to host a default state value, declare a state type, and identify the state it defines at the same time. Hence, a procedure of state changes with POJOs as state definitions can further evolve into:

```ts
interface StateA {
  highlighted: boolean;
  text: string;
  hint: string;
  hintColor: Color;
}

const defOfStateA: StateA = {
  highlighted: false,
  text: '',
  hint: '',
  hintColor: Color.BLACK,
};

interface StateB {
  mode: Mode;
}

const defOfStateB: StateB = {
  mode: Mode.NOISY,
};

// ...

operate(defOfStateA, highlight);

operate(defOfStateA, setText, 'Lorem ipsum');

operate(defOfStateA, setHint, 'Ut enim', Color.GOLD);

operate([defOfStateA, defOfStateB], setTextAndSwitchMode, 'Duis', Mode.QUIET);
```

With a solution like this, predictable state changes come at a low cost.

To make the solution complete, fundamental blocks of state management like getting states with or without selectors, subscribing to state changes, and integrating with frameworks are also needed. For your convenience, here is an open-source in-progress project implemented following the idea above: https://github.com/statofu/statofu . You are very welcome to fork it or open issues in it to join the exploration.
