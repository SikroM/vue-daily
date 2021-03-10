# Module

由于使用单一状态树，应用的所有状态会集中到一个比较大的对象。当应用变得非常复杂的时候，store对象就有可能变得相当臃肿。

为了解决以上问题，Vuex允许我们将store分割成模块。每个模块拥有自己的state、mutation、action、getter, 甚至是嵌套子模块————从上至下进行同样方式的分割：

```js
const moduleA = {
  state: () => ({...}),
  mutations: {...},
  actions: {...}
  getters: {...}
}

const moduleB = {
  state: () => ({...}),
  mutations: {...},
  actions: {...}
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})
```

store.state.a //-> moduleA 的状态
store.state.b //-> moduleB 的状态

## 模块的局部状态

对于模块内部的mutation和getter，接收到的第一个参数是模块的局部状态对象。

```js
const moduleA = {
  state: () => ({
    count: 0
  }),
  mutations: {
    increment(state) {
      // 这里的 state 对象是模块的局部状态
      state.count++
    }
  },
  getters: {
    doubleCount(state) {
      return state.count * 2
    }
  }
}
```

同样，对于模块内部的action，局部状态通过`context.state`暴露出来，根节点状态则为`context.rootState`

```js
const moduleA = {
  actions: {
    incrementIfOddOnRootSum({state, commit, rootState}) {
      if((state.count + rootState.count) % 2 === 1) {
        commit('increment')
      }
    }
  }
}
```

对于模块内部的getter，根节点状态会作为第三个参数暴露出来：

```js
const moduleA = {
  getters: {
    sumWithRootCount(state, getters, rootState) {
      return state.count + rootState.count;
    }
  }
}
```

### 命名空间

默认情况下，模块内部的action, mutation, getter是注册在全局命名空间的————这样使得多个模块能够对同一mutation或action做出响应。

如果希望你的模块具有更高的封装度和复用性，你可以通过添加`namespaced: true`的方式使其成为带命名空间的模块。当模块被注册后，它的所有getter, action, mutation都会自动根据模块注册的路径调整命名。例如：

```js
const store = new Vuex.Store({
  modules: {
    account: {
      namespaced: true,

      // 模块内容
      state: () => ({...}),  // 模块内的状态已经是嵌套的了，使用 namespaced 属性不会对其产生影响
      getters: {
        isAdmin() { ... } // -> getters['account/isAdmin']
      },
      actions: {
        login() {...} // -> dispatch('account/login')
      },
      mutations: {
        login() { ... } // -> commit('account/login')
      },

      // 嵌套模块
      modules: {
        // 继承父模块的命名空间
        myPage: {
          state: () => ({...}),
          getters: {
            profile() {...} // -> getters['account/profile']
          }
        },
        posts: {
          namespaced: true,

          state: () => ({...}),
          getters: {
            popular() {...} // -> getters['account/posts/popular']
          }
        }
      }
    },
  }
})
```

启用了命名空间的getter和action会收到局部化的`getter`, `dispatch`和`commit`。换言之，你在使用模块内容时不需要在同一个模块内额外添加空间名前缀。更改`namespaced`属性后不需要修改模块内的代码。


### 在带命名空间的模块内访问全局内容

如果你希望使用全局state和getter, `rootState`和`rootGetters`会作为第三和第四参数传入getter，也会通过`context`对象的属性传入action。

若需要在全局命名空间内分发action或提交mutation, 将 `{root: true}`作为第三参数传给`dispatch`或`commit`即可。


```js
modules: {
  foo: {
    namespaced: true,

    getters: {
      /**
       * 在这个模块的 getter 中， getters 被局部化了
       * 你可以使用 getter 的第四个参数来调用 rootGetters
       */ 
      someGetter(state, getters, rootState, rootGetters) {
        getters.someOtherGetter // -> foo/someOtherGetter
        rootGetters.someOtherGetter // -> someOtherGetter
      },
      someOtherGetter: state => { ... }
    },

    actions: {
      /**
       * 在这个模块中，dispatch 和 commit 也被局部化了
       * 它们可以接受 root 属性以访问根 dispatch 或 commit
       */ 
      someAction({ dispatch, commit, getters, rootGetters }) {
        getters.someGetter  // -> foo/someGetter
        rootGetters.someGetter // -> someGetter

        dispatch('someOtherAction')
        dispatch('someOtherAction', null, { root: true }) // -> someOtherAction

        commit('someMutation') // -> foo/someMutation
        commit('someMutation', null, { root: true })  //-> someMutation
      },
      someOtherAction(ctx, payload) { ... }
    }
  }
}
```

### 在带命名空间的模块注册全局action

若需要在带命名空间的模块注册全局action，你可以添加`{ root: true }`，并将这个action的定义放在函数`handler`中。例如：

```js
{
  actions: {
    someOtherAction({dispatch}) {
      dispatch('someAction')
    }
  },
  modules: {
    foo: {
      namespaced: true,

      actions: {
        someAction: {
          root: true,
          handler(namespacedContext, payload) {...}
        }
      }
    }
  }
}
```

### 带命名空间的绑定函数

当使用`mapState`, `mapGetters`, `mapActions`, `mapMutations`这些函数来绑定带命名空间的模块时，写起来比较繁琐：

```js
computed: {
  ...mapState({
    a: state => state.some.nested.module.a,
    b: state => state.some.nested.module.b
  })
},
methods: {
  ...mapActions([
    'some/nested/module/foo', // -> this['some/nested/module/foo']()
    'some/nested/module/bar'  // -> this['some/nested/module/bar']()
  ])
}
```

对于这种情况，你可以将模块的空间名称字符串作为第一个参数传给上述函数，这样所有绑定都会自动将该模块作为上下文。于是上面的例子可以简化为：

```js
computed: {
  ...mapState('some/nested/module', {
    a: (state) => state.a,
    b: state => state.b
  })
},
methods: {
  ...mapActions('some/nested/module', [
    'foo',
    'bar'
  ])
}
```

而且，你可以通过使用`createNamespacedHelpers`创建基于某个命名空间辅助函数。它返回一个对象，对象里有新的绑定在给定命名空间值上的组件绑定辅助函数：

```js
import { createNamespacedHelpers } from 'vuex'

const { mapState, mapActions } = createNamespacedHelper('some/nested/module')

export default {
  computed: {
    // 在 some/nested/module 中查找
    ...mapState({
      a: state => state.a,
      b: state => state.b
    })
  },
  methods: {
    ...mapActions([
      'foo',
      'bar'
    ])
  }
}
```

