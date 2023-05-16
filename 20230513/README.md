# Farewell to Redux、Recoil、MobX、Zustand、Jotai and Valtio, another way of state management?

After my writing frontend code for seven more years, finalizing projects, having rethinks, I feel more and more certain that something being used on a daily basis can be questioned. One of the questions is about today's widely-accepted practices in **state management** so I would like to share some thoughts by this article for an exploration 🙏.

## Why not to stick with Redux, Recoil, MobX, Zustand, Jotai or Valtio

Today, there are many libraries of state management, especially in React. To describe the problems clearly, I would get started by looking into today's widely-accepted libraries in React.

Firstly, let's take a look at Redux. When a simple action changes one state, what one state it changes can be understood clearly by only checking the slice in which it is declared:

```ts
const checkboxSlice = createSlice({
  name: 'checkbox',
  initialState: {
    checked: false,
  },
  reducers: {
    check(state) {
      // ...
    },
  },
});

const { check } = checkboxSlice.actions;

// ...

dispatch(check());

// By checking `check` is declared in `checkboxSlice`, we know, by the design of Redux, `check` changes the one state represented by `checkboxSlice`.
```

But, when a complicated action changes multi states, what multi states it changes can't be understood by only checking where it is declared:

```diff
const checkboxSlice = createSlice({
  name: 'checkbox',
  initialState: {
    checked: false,
  },
  reducers: {
    check(state) {
      // ...
    },
+
+    uncheck(state) {
+      // ...
+    }
  },
});

-const { check } = checkboxSlice.actions;
+const { check, uncheck } = checkboxSlice.actions;

// The underlying simple action `uncheck` needs to be built in advance for the complicated action `uncheckWithTextCleaned` but may never get invoked anywhere else.
```

```ts
const textareaSlice = createSlice({
  name: 'textarea',
  initialState: {
    text: '',
  },
  reducers: {
    setText(state, action: PayloadAction<string>) {
      // ...
    },
  },
});

const { setText } = textareaSlice.actions;

function uncheckWithTextCleaned(): AppThunk {
  return (dispatch) => {
    // ...
  };
}

// ...

dispatch(uncheckWithTextCleaned());

// By only checking the function declaration of `uncheckWithTextCleaned`, we don't know what multi states `uncheckWithTextCleaned` changes.
```

Without tracking function bodies, the multi states to be changed remain unknown so multi-state changing goes unpredictable. With tracking function bodies, the multi states to be changed get known but overall cost of development on use increases:

```ts
function uncheckWithTextCleaned(): AppThunk {
  return (dispatch) => {
    dispatch(uncheck());
    dispatch(setText(''));
  };
}

// ...

dispatch(uncheckWithTextCleaned());

// By tracking `uncheckWithTextCleaned` invokes `uncheck` declared in `checkboxSlice` and `setText` declared in `textareaSlice`, we know `uncheckWithTextCleaned` changes the multi states represented by `checkboxSlice` and `textareaSlice`.
```

In addition, when underlying simple actions to be invoked in a complicated action to be built are not yet ready, they need to be built in advance only for it but may never get invoked anywhere else. Then, complicated actions become high-coupling with their underlying slices, which brings difficulties in development, thus the cost increases further.

Next, let's check out Recoil and MobX. In Recoil, states changing is defined by state-changing hooks:

```ts
const checkboxState = atom({
  key: 'checkbox',
  default: {
    checked: false,
  },
});

const textareaState = atom({
  key: 'textarea',
  default: {
    text: '',
  },
});

// ...

function useSetText() {
  return useRecoilCallback(
    ({ set }) =>
      (text: string) => {
        // ...
      },
    []
  );
}

function useUncheckWithTextCleaned() {
  const setText = useSetText();

  return useRecoilCallback(
    ({ set }) =>
      () => {
        // ...
      },
    []
  );
}

// ...

const uncheckWithTextCleaned = useUncheckWithTextCleaned();

// ...

uncheckWithTextCleaned();

// By only checking the function declaration of `uncheckWithTextCleaned` or `useUncheckWithTextCleaned`, we don't know what states the hook changes. To know that, what set calls the hook directly or indirectly invokes needs to be figured out by tracking function bodies.
```

In MobX, states changing is defined by store methods:

```ts
class CheckboxStore {
  private textareaStore: TextareaStore;

  checked: boolean;

  constructor(textareaStore: TextareaStore) {
    makeAutoObservable(this);
    this.textareaStore = textareaStore;
    this.checked = false;
  }

  uncheckWithTextCleaned(): void {
    // ...
  }
}

class TextareaStore {
  text: string;

  constructor() {
    makeAutoObservable(this);
    this.text = '';
  }

  setText(text): void {
    // ...
  }
}

// ...

checkboxStore.uncheckWithTextCleaned();

// By only checking the function declaration of `checkboxStore.uncheckWithTextCleaned`, we don't know what states the method changes. To know that, what store properties the method directly or indirectly changes needs to be figured out by tracking function bodies.
```

Similarly to Redux, without tracking function bodies, states changing goes unpredictable. With tracking function bodies, the cost increases.

Besides, as Recoil cares a bit much about asynchronousness, it's inconvenient to get states in state-changing hooks. As MobX has its own independent subscription mechanism, a strong understanding to correctly use it is required. Then, these increase the cost further.

And, for the rest 3 libraries, Zustand, Jotai and Valtio act very like Redux, Recoil and MobX separately in fact. Or, in other words, the former ones feel much like lightweight versions of the latter ones.

To sum up, two problems that today's widely-accepted libraries of state management in React don't actually handle well are (1) predictability of states changing and (2) overall cost of development on use. Further more, with a glimpse of the most widely-accepted library of state management in each of different frameworks, the problems can be considered to exist universally.

## Predictability, and side effects

A function is said to have side effects if it makes any effects other than outputing a return value. In the examples above, the sides effects of the functions are all states changing.

Though, a function with side effects is not bound to behave unpredictably. As long as side effects of a function are well controlled, it can behave predictably. Like the example of Redux, a simple action is restricted, by the design of Redux, to change the one state represented by the slice in which it is delcared. But, with side effects of a function badly controlled, while the function body goes more and more complicated, the side effects can become more and more out of control, which makes the function behaves unpredictably at last when the side effects become out of control completely.

On the other hand, a function without side effects is naturally bound to behave predictably.

So, to solve the problem of predictability of states changing, we can either keep side effects of state-changing functions well controlled all the time, or eliminate side effects of state-changing functions thoroughly.

## Overall cost of development on use, and preferences

Besides that the problem of predictability increases overall cost of development on use, preferences of libraries of state management can increase the cost, too. Like the examples above, creating a new store in Redux, getting states in state-changing hooks in Recoil and correctly using the subscription mechanism in MobX are all costly just because of preferences of each library.

If the cost on the fundamental usages of state management increases by preferences of a library, the library can feel not good enough in almost every aspect, which is supposed to be avoided.

## Another way of state management

Having analyzed the problems, it's time to try to think up another way of state management, which is to think about how to design another library of state management that well handles the two problems above. Although both of the two ideas metioned above of solving the problem of predictability are workable, the idea of eliminating side effects of state-changing functions thoroughly is preferred here for now in persuit of simplicity.

For one-state changes, here a pure function that processes old one state to return new one state can be involved:

```ts
function check(checkboxState: CheckboxState): CheckboxState {
  return {
    /* ... */
  };
}
```

For multi-state changes, here a pure function that processes old multi states to return new multi states can be involved:

```ts
function uncheckWithTextCleaned([checkboxState, textareaState]: [
  CheckboxState,
  TextareaState
]): [CheckboxState, TextareaState] {
  return [
    /* ... */
  ];
}
```

Meanwhile, the functions can process payloads besides states:

```ts
function setText(textarea: TextareaState, text: string): TextareaState {
  return {
    /* ... */
  };
}
```

Because pure functions don't have any side effects, involving the functions to change either one or multi states only changes the states in the function declarations, which means what states they change can be understood clearly without tracking any function bodies.

After that, state-changing steps of involving the functions can work like this, (1) reading old states, (2) passing the old states into the functions to evaluate new states and (3) writing the new states:

```ts
const oldCheckboxState = getState(keyOfCheckboxState);
const newCheckboxState = check(oldCheckboxState);
setState(keyOfCheckboxState, newCheckboxState);
```

The steps can also be defined as a reuseable function `operate`:

```ts
operate(keyOfCheckboxState, check);
operate(keyOfTextareaState, setText, '');
operate([keyOfCheckboxState, keyOfTextareaState], uncheckWithTextCleaned);
```

By now, the propotype gets shaped.

Next, more improvements can be found out in the perspective of decreasing overall cost of development on use.

With a slightly closer look at the first parameter `keyOf...` of `operate`, it can be realized that its role is to (1) identify states. But, to fully defining states, it's also needed to (2) host the default states and (3) declare the states types. If the three points are put together, there can be found a matching concept in JS, which is Plain Old JavaScript Object(POJO). So, the cost can decrease further by defining states by POJOs:

```ts
interface CheckboxState {
  checked: boolean;
}

const defOfCheckboxState: CheckboxState = {
  checked: false,
};

interface TextareaState {
  text: string;
}

const defOfTextareaState: TextareaState = {
  text: '',
};

// ...

operate(defOfCheckboxState, check);
operate(defOfTextareaState, setText, '');
operate([defOfCheckboxState, defOfTextareaState], uncheckWithTextCleaned);
```

Besides, the rest parts for the fundamental usages of state management are added without any preferences, which includes (1) getting states, (2) subscribing to state changes and (3) unsubcribing:

```ts
const checkboxState1 = snapshot(defOfCheckboxState);
const textareaState1 = snapshot(defOfTextareaState);
const [checkboxState2, textareaState2] = snapshot([
  defOfCheckboxState,
  defOfTextareaState,
]);

const unsubscribeCheckboxStateChanges = subscribe(
  defOfCheckboxState,
  onCheckboxStateChange
);
const unsubscribeTextareaStateChanges = subscribe(
  defOfTextareaState,
  onTextareaStateChange
);
const unsubscribeCheckboxTextareaStatesChanges = subscribe(
  [defOfCheckboxState, defOfTextareaState],
  onCheckboxTextareaStatesChange
);
```

That's it. Another library of state management that well handles the problems of (1) predictability of states changing and (2) overall cost of development on use is roughly built.

## Prospect

State management is a very basic but very vital part in frontend development, but today there is just no library that achieves predictable states changing and low overall cost of development on use, which brings challenges in development.

As good state management constitues a necessity for a good client app, perhaps, every of us, as frontend devs, may think about what is the best in state management.

Also, for the convenience of our trials together, I followed the thoughts above and tried to build a library of state management https://github.com/statofu/statofu .

Comments are welcomed anywhere for the exploration.
