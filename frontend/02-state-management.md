# 前端状态管理

## 1. Redux

### 1.1 基本概念
```js
// Action
const ADD_TODO = 'ADD_TODO';
const addTodo = (text) => ({
  type: ADD_TODO,
  payload: { text }
});

// Reducer
const initialState = {
  todos: []
};

function todoReducer(state = initialState, action) {
  switch (action.type) {
    case ADD_TODO:
      return {
        ...state,
        todos: [...state.todos, action.payload]
      };
    default:
      return state;
  }
}

// Store
import { createStore } from 'redux';
const store = createStore(todoReducer);
```

### 1.2 React-Redux使用
```jsx
// Provider
import { Provider } from 'react-redux';

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
);

// 连接组件
import { connect } from 'react-redux';

function TodoList({ todos, addTodo }) {
  return (
    <div>
      <button onClick={() => addTodo('新任务')}>添加任务</button>
      <ul>
        {todos.map((todo, index) => (
          <li key={index}>{todo.text}</li>
        ))}
      </ul>
    </div>
  );
}

const mapStateToProps = state => ({
  todos: state.todos
});

const mapDispatchToProps = {
  addTodo
};

export default connect(mapStateToProps, mapDispatchToProps)(TodoList);
```

## 2. Vuex

### 2.1 基本使用
```js
// store/index.js
import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);

export default new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment(state) {
      state.count++;
    }
  },
  actions: {
    incrementAsync({ commit }) {
      setTimeout(() => {
        commit('increment');
      }, 1000);
    }
  },
  getters: {
    doubleCount: state => state.count * 2
  }
});

// 组件中使用
export default {
  computed: {
    count() {
      return this.$store.state.count;
    },
    doubleCount() {
      return this.$store.getters.doubleCount;
    }
  },
  methods: {
    increment() {
      this.$store.commit('increment');
    },
    incrementAsync() {
      this.$store.dispatch('incrementAsync');
    }
  }
};
```

### 2.2 模块化
```js
// store/modules/user.js
const user = {
  namespaced: true,
  state: {
    userInfo: null
  },
  mutations: {
    setUserInfo(state, userInfo) {
      state.userInfo = userInfo;
    }
  },
  actions: {
    async login({ commit }, credentials) {
      const userInfo = await api.login(credentials);
      commit('setUserInfo', userInfo);
    }
  }
};

// store/index.js
import user from './modules/user';

export default new Vuex.Store({
  modules: {
    user
  }
});
```

## 3. MobX

### 3.1 基本使用
```js
import { makeAutoObservable } from 'mobx';

class TodoStore {
  todos = [];
  
  constructor() {
    makeAutoObservable(this);
  }
  
  addTodo(text) {
    this.todos.push({ text, completed: false });
  }
  
  toggleTodo(index) {
    this.todos[index].completed = !this.todos[index].completed;
  }
  
  get completedTodosCount() {
    return this.todos.filter(todo => todo.completed).length;
  }
}

const store = new TodoStore();
export default store;
```

### 3.2 React集成
```jsx
import { observer } from 'mobx-react-lite';
import todoStore from './stores/todoStore';

const TodoList = observer(() => {
  return (
    <div>
      <button onClick={() => todoStore.addTodo('新任务')}>
        添加任务
      </button>
      <ul>
        {todoStore.todos.map((todo, index) => (
          <li
            key={index}
            onClick={() => todoStore.toggleTodo(index)}
            style={{
              textDecoration: todo.completed ? 'line-through' : 'none'
            }}
          >
            {todo.text}
          </li>
        ))}
      </ul>
      <div>完成的任务数: {todoStore.completedTodosCount}</div>
    </div>
  );
});
```

## 4. Recoil

### 4.1 基本使用
```jsx
import { atom, selector, useRecoilState, useRecoilValue } from 'recoil';

// 定义atom
const todoListState = atom({
  key: 'todoListState',
  default: []
});

// 定义selector
const todoStatsState = selector({
  key: 'todoStatsState',
  get: ({get}) => {
    const todoList = get(todoListState);
    const totalNum = todoList.length;
    const completedNum = todoList.filter(item => item.completed).length;
    
    return {
      totalNum,
      completedNum,
      uncompletedNum: totalNum - completedNum
    };
  }
});

// 使用状态
function TodoList() {
  const [todoList, setTodoList] = useRecoilState(todoListState);
  const stats = useRecoilValue(todoStatsState);
  
  return (
    <div>
      {/* 组件内容 */}
    </div>
  );
}
```

## 5. 状态管理最佳实践

### 5.1 状态分类
```js
// 应用级状态
const globalState = {
  user: null,
  theme: 'light',
  language: 'zh-CN'
};

// 页面级状态
const pageState = {
  loading: false,
  data: null,
  error: null
};

// 组件级状态
const componentState = {
  isOpen: false,
  selectedValue: null
};
```

### 5.2 性能优化
```jsx
// 使用选择器
const selectTodos = state => state.todos;
const selectVisibleTodos = createSelector(
  selectTodos,
  todos => todos.filter(todo => !todo.completed)
);

// 避免不必要的渲染
const TodoItem = React.memo(({ todo, onToggle }) => (
  <li onClick={() => onToggle(todo.id)}>
    {todo.text}
  </li>
));

// 批量更新
function batchedUpdates() {
  batch(() => {
    dispatch(action1());
    dispatch(action2());
    dispatch(action3());
  });
}
```

### 5.3 异步处理
```js
// Redux Thunk
const fetchUserData = (userId) => {
  return async dispatch => {
    dispatch({ type: 'FETCH_USER_REQUEST' });
    try {
      const response = await api.fetchUser(userId);
      dispatch({ type: 'FETCH_USER_SUCCESS', payload: response.data });
    } catch (error) {
      dispatch({ type: 'FETCH_USER_FAILURE', error });
    }
  };
};

// Redux Saga
function* fetchUserSaga(action) {
  try {
    yield put({ type: 'FETCH_USER_REQUEST' });
    const user = yield call(api.fetchUser, action.payload);
    yield put({ type: 'FETCH_USER_SUCCESS', user });
  } catch (error) {
    yield put({ type: 'FETCH_USER_FAILURE', error });
  }
}
```

## 6. 调试工具

### 6.1 Redux DevTools
```js
import { createStore, applyMiddleware, compose } from 'redux';

const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;
const store = createStore(
  reducer,
  composeEnhancers(applyMiddleware(...middleware))
);
```

### 6.2 Vue DevTools
```js
// Vue.js DevTools配置
Vue.config.devtools = process.env.NODE_ENV === 'development';
```

## 7. 状态持久化

### 7.1 Redux持久化
```js
import { persistStore, persistReducer } from 'redux-persist';
import storage from 'redux-persist/lib/storage';

const persistConfig = {
  key: 'root',
  storage,
  whitelist: ['user'] // 只持久化user
};

const persistedReducer = persistReducer(persistConfig, rootReducer);
const store = createStore(persistedReducer);
const persistor = persistStore(store);
```

### 7.2 Vuex持久化
```js
import createPersistedState from 'vuex-persistedstate';

export default new Vuex.Store({
  plugins: [
    createPersistedState({
      paths: ['user'] // 只持久化user模块
    })
  ],
  // ...
});
``` 