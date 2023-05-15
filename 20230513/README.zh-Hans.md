# 再见了 Redux、Recoil、MobX、Zustand、Jotai 还有 Valtio，状态管理还可以这样做？

Hi，大家好，这是我第一次在掘金上正式更文，还请多包涵。我坚持在一线写前端代码大概有七八年了，写过一些项目，有过一些反思，越来越确信平日里一直用得心安理得某些的东西也许存在着问题，比如：在 **状态管理** 上一直比较流行的实践 🙏，所以试着分享出来探讨一下。

## 为什么要告别 Redux、Recoil、MobX、Zustand、Jotai 还有 Valtio

今天流行的状态管理库有很多，尤其在 React 中。为了把问题说得清晰一些，我想以 React 中的几个主流库切入聊起。

首先看一下 Redux。对于单个状态的变化，可以 dispatch 简单 action。想知道这个简单 action 会改变什么状态，根据 Redux 的设计，检查它声明在哪个 slice 里就可以了：

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

// 因为 `check` 声明在 `checkboxSlice` 里，根据 Redux 的设计可以知道 `check` 改变的是 `checkboxSlice` 代表的状态。
```

而对于多个状态的变化，需要 dispatch 复杂 action。想知道这个复杂 action 会改变什么状态，只检查它声明在哪里是不够的：

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

// 先构建复杂 action `uncheckWithTextCleaned` 要调用的底层简单 action `uncheck`，而这个简单 action 大概率不会在别的地方用到了。
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

// 在只检查 `uncheckWithTextCleaned` 的函数声明的情况下，无法知道这个复杂 action 会改变什么状态。
```

如果不追踪函数体，就无法知道复杂 action 会改变什么状态，那么状态变化就变得不可预测了。如果追踪了函数体，尽管可以知道会改变什么状态，但使用上的总体开发成本也就随着加重了：

```ts
function uncheckWithTextCleaned(): AppThunk {
  return (dispatch) => {
    dispatch(uncheck());
    dispatch(setText(''));
  };
}

// ...

dispatch(uncheckWithTextCleaned());

// 通过追踪函数体发现 `uncheckWithTextCleaned` 调用了 `uncheck` 和 `setText`，由于 `uncheck` 声明在 `checkboxSlice` 里，`setText` 声明在 `textareaSlice`，可以知道 `uncheckWithTextCleaned` 改变的是 `checkboxSlice` 和 `textareaSlice` 代表的状态。
```

此外，在复杂 action 要调用的底层简单 action 还没准备好的时候，就要先构建这些要调用的简单 action，而这些简单 action 大概率不会在别的地方用到了。这样，复杂 action 就与底层 slice 高耦合了，会导致开发困难，也就使成本进一步加重了。

再看一下 Recoil 和 MobX。在 Recoil 中是通过自定义 hook 来封装状态变化的：

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

// 在只检查 `uncheckWithTextCleaned` 或 `useUncheckWithTextCleaned` 的函数声明的情况下，无法知道这个 hook 会改变什么状态。需要通过追踪函数体发现直接或间接发起的 `set` 调用才能知道会改变什么状态。
```

在 MobX 中是通过类的方法来封装状态变化的：

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

// 在只检查 `checkboxStore.uncheckWithTextCleaned` 的函数声明的情况下，无法知道这个方法会改变什么状态。需要通过追踪函数体发现直接或间接改变的 property 才能知道会改变什么状态。
```

与 Redux 类似地，如果不追踪函数体，状态变化就不可预测了。如果追踪了函数体，使用成本就加重了。

此外，由于 Recoil 较多地考虑了异步状态变化，在自定义 hook 中获取状态会比较麻烦，由于 MobX 有独立的订阅机制，妥当使用需要准确理解。这些，都使成本进一步加重了。

而余下的三个库，Zustand、Jotai 和 Valtio，它们用起来分别非常像 Redux、Recoil 和 MobX。或许可以说，前几者基本上是后几者的简化版。

小结一下，React 中的主流状态管理库在 (1) **状态变化的可预测性** 和 (2) **使用上的总体开发成本** 上存在着问题。如果稍微看一下其他框架中最主流的状态管理库，会发现它们也有类似问题。所以可以说，这两个问题是普遍存在的。

## 可预测性与副作用

当函数在输出返回值之外还产生了其他效果，那这个函数就是有副作用（side effect）的。像上面例子中的函数，副作用都是改变状态。

而函数有副作用不等同于函数行为是不可预测的。只要副作用是可控的，函数行为就是可预测的。像 Redux 例子中的简单 action，根据 Redux 的设计只能改变声明各自的 slice 所代表的状态。但是，对函数的副作用不加以控制的话，随着函数体的复杂度上升副作用的可控性就会下降，最终，不可控的副作用就会让函数行为变得不可预测。

而函数没有副作用的话，函数行为就自然而然的可预测了。

这么想一想，要解决状态变化的可预测性问题，要么一直保持改变状态的函数的副作用可控，要么彻底去除改变状态的函数的副作用。

## 使用上的总体开发成本与偏好

除了可预测性问题对使用上的总体开发成本的影响，状态管理库自身的偏好也会较大程度地影响使用上的总体开发成本。像 Redux 中创建一个新的 store、像 Recoil 中自定义 hook 访问状态、像 MobX 中妥当使用订阅机制，都受到库自身的偏好影响变得有些费时费力。

当由于状态管理库自身的偏好加重了最基本的状态管理功能上的使用成本时，就会对这个状态管理库方方面面的使用产生负面影响，这是应该避免的。

## 状态管理的新做法

分析好了问题，接下来就可以想一下状态管理的新做法了，也就是，如何设计一个能够解决以上两个问题的新状态管理库。对于解决状态变化的可预测性问题，上面提到的两种做法尽管都可行，但出于对简洁性的追求，先尝试一下 “彻底去除改变状态的函数的副作用” 的做法。

对于单个状态的变化，可以引用以单个状态为入参、以新的单个状态为返回值的纯函数来完成：

```ts
function check(checkboxState: CheckboxState): CheckboxState {
  return {
    /* ... */
  };
}
```

对于多个状态的变化，可以引用以多个状态为入参、以新的多个状态为返回值的纯函数来完成：

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

同时这些函数还应该能够接受除了状态之外的更多参数：

```ts
function setText(textarea: TextareaState, text: string): TextareaState {
  return {
    /* ... */
  };
}
```

由于纯函数没有副作用，引用这些函数改变不论单个还是多个状态都只会改变函数声明中的状态，也就是，可以在不追踪函数体的情况下知道会改变什么状态。

然后，引用这些函数改变状态的过程大致是这样的，(1) 读取状态、(2) 将状态传入函数计算新的状态 和 (3) 写入新的状态：

```ts
const oldCheckboxState = getState(keyOfCheckboxState);
const newCheckboxState = check(oldCheckboxState);
setState(keyOfCheckboxState, newCheckboxState);
```

这个过程可以进一步封装成一个通用的函数 `operate`：

```ts
operate(keyOfCheckboxState, check);
operate(keyOfTextareaState, setText, '');
operate([keyOfCheckboxState, keyOfTextareaState], uncheckWithTextCleaned);
```

这样，新状态管理库的雏形就有了。

接下来，再从减轻使用成本的角度试着做一做优化。

稍微看一下 `operate` 的第一个参数 `keyOf...`，作用是 (1) 唯一标识状态。但是单独为了唯一标识状态，就声明一系列唯一的字符串常量，成本是比较高的。而完整定义状态，还需要 (2) 状态的默认值 和 (3) 状态的类型。如果把这三点关联在一起的话，就会发现对应到了 JS 中的一个常用概念，Plain Old JavaScript Object（POJO）。那么，通过 POJO 来定义状态的话，使用成本就进一步减轻了：

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

然后，再以无偏好地方式加上其他最基本的状态管理功能，(1) 获取状态、(2) 订阅状态变化 和 (3) 取消订阅：

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

这样，能够解决 (1) 状态变化的可预测性 和 (2) 使用上的总体开发成本 上问题的新状态管理库就大致做好了。

## 展望

在前端开发中状态管理是非常基础但极其重要的部分，而今天恰恰缺少了一个状态变化可预测、使用总体成本低的状态管理库，这给前端开发带来了许多挑战。

好的前端应用需要好的状态管理，作为前端开发者的我们也许都可以想一想怎么做状态管理才是好的。

此外，我也试着按照上面的思路写了一个状态管理库 https://github.com/statofu/statofu ，方便一起进一步尝试。

欢迎大家留言进一步探讨。
