# 前端路由管理

## 1. React Router

### 1.1 基本路由
```jsx
import { BrowserRouter, Route, Switch, Link } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">首页</Link>
        <Link to="/about">关于</Link>
        <Link to="/users">用户</Link>
      </nav>

      <Switch>
        <Route exact path="/" component={Home} />
        <Route path="/about" component={About} />
        <Route path="/users" component={Users} />
        <Route path="*" component={NotFound} />
      </Switch>
    </BrowserRouter>
  );
}
```

### 1.2 动态路由
```jsx
// 路由参数
<Route path="/users/:id" component={UserDetail} />

// 组件中获取参数
function UserDetail({ match }) {
  const { id } = match.params;
  return <div>用户ID: {id}</div>;
}

// 编程式导航
import { useHistory } from 'react-router-dom';

function NavigateButton() {
  const history = useHistory();
  
  return (
    <button onClick={() => history.push('/users/1')}>
      查看用户详情
    </button>
  );
}
```

### 1.3 路由守卫
```jsx
// 私有路由组件
function PrivateRoute({ component: Component, ...rest }) {
  return (
    <Route
      {...rest}
      render={props =>
        isAuthenticated() ? (
          <Component {...props} />
        ) : (
          <Redirect
            to={{
              pathname: "/login",
              state: { from: props.location }
            }}
          />
        )
      }
    />
  );
}

// 使用私有路由
<Switch>
  <Route path="/login" component={Login} />
  <PrivateRoute path="/dashboard" component={Dashboard} />
</Switch>
```

## 2. Vue Router

### 2.1 基本配置
```js
import Vue from 'vue';
import VueRouter from 'vue-router';

Vue.use(VueRouter);

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/about',
    name: 'About',
    component: () => import('./views/About.vue') // 懒加载
  },
  {
    path: '*',
    component: NotFound
  }
];

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes
});

export default router;
```

### 2.2 导航守卫
```js
// 全局前置守卫
router.beforeEach((to, from, next) => {
  if (to.matched.some(record => record.meta.requiresAuth)) {
    if (!isAuthenticated()) {
      next({
        path: '/login',
        query: { redirect: to.fullPath }
      });
    } else {
      next();
    }
  } else {
    next();
  }
});

// 路由独享守卫
const routes = [
  {
    path: '/admin',
    component: Admin,
    beforeEnter: (to, from, next) => {
      if (isAdmin()) {
        next();
      } else {
        next('/403');
      }
    }
  }
];

// 组件内守卫
export default {
  beforeRouteEnter(to, from, next) {
    // 在渲染该组件的对应路由被验证前调用
    next(vm => {
      // 通过 vm 访问组件实例
    });
  },
  beforeRouteUpdate(to, from, next) {
    // 在当前路由改变，但是该组件被复用时调用
    next();
  },
  beforeRouteLeave(to, from, next) {
    // 导航离开该组件的对应路由时调用
    next();
  }
};
```

## 3. 路由最佳实践

### 3.1 路由配置组织
```js
// 模块化路由配置
// routes/user.js
export default {
  path: '/user',
  component: UserLayout,
  children: [
    {
      path: 'profile',
      component: UserProfile
    },
    {
      path: 'settings',
      component: UserSettings
    }
  ]
};

// routes/index.js
import userRoutes from './user';
import adminRoutes from './admin';

export default [
  {
    path: '/',
    component: Layout,
    children: [
      userRoutes,
      adminRoutes
    ]
  }
];
```

### 3.2 路由权限控制
```js
// 权限配置
const routePermissions = {
  '/admin': ['admin'],
  '/user/settings': ['user', 'admin']
};

// 权限检查函数
function hasPermission(route, roles) {
  if (!routePermissions[route]) return true;
  return roles.some(role => routePermissions[route].includes(role));
}

// 过滤路由
function filterRoutes(routes, roles) {
  return routes.filter(route => {
    if (hasPermission(route.path, roles)) {
      if (route.children) {
        route.children = filterRoutes(route.children, roles);
      }
      return true;
    }
    return false;
  });
}
```

### 3.3 路由动画
```vue
<!-- Vue路由动画 -->
<template>
  <transition name="fade" mode="out-in">
    <router-view></router-view>
  </transition>
</template>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>

<!-- React路由动画 -->
import { CSSTransition, TransitionGroup } from 'react-transition-group';

function App() {
  return (
    <Route
      render={({ location }) => (
        <TransitionGroup>
          <CSSTransition
            key={location.key}
            timeout={300}
            classNames="fade"
          >
            <Switch location={location}>
              {routes}
            </Switch>
          </CSSTransition>
        </TransitionGroup>
      )}
    />
  );
}
```

## 4. 路由性能优化

### 4.1 路由懒加载
```js
// Vue懒加载
const routes = [
  {
    path: '/user',
    component: () => import('./views/User.vue')
  }
];

// React懒加载
const UserComponent = React.lazy(() => import('./components/User'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserComponent />
    </Suspense>
  );
}
```

### 4.2 预加载路由
```js
// 预加载函数
function preloadRoute(path) {
  const component = routes.find(route => route.path === path)?.component;
  if (typeof component === 'function') {
    component();
  }
}

// 在合适的时机预加载
function handleMouseEnter(path) {
  preloadRoute(path);
}

<Link 
  to="/user" 
  onMouseEnter={() => handleMouseEnter('/user')}
>
  用户中心
</Link>
```

## 5. 路由状态管理

### 5.1 与Redux集成
```js
import { connectRouter, routerMiddleware } from 'connected-react-router';
import { createBrowserHistory } from 'history';

const history = createBrowserHistory();

const store = createStore(
  combineReducers({
    router: connectRouter(history),
    // 其他reducers
  }),
  applyMiddleware(routerMiddleware(history))
);

// 在组件中使用
import { push } from 'connected-react-router';

function NavigateButton({ dispatch }) {
  return (
    <button onClick={() => dispatch(push('/user'))}>
      跳转到用户页面
    </button>
  );
}
```

### 5.2 与Vuex集成
```js
import VueRouter from 'vue-router';
import Vuex from 'vuex';

const store = new Vuex.Store({
  state: {
    route: null
  },
  mutations: {
    updateRoute(state, route) {
      state.route = route;
    }
  }
});

const router = new VueRouter({ ... });

router.afterEach((to) => {
  store.commit('updateRoute', to);
});
```

## 6. 路由测试

### 6.1 路由单元测试
```js
// Jest测试
import { render, fireEvent } from '@testing-library/react';
import { createMemoryHistory } from 'history';
import { Router } from 'react-router-dom';

test('navigation test', () => {
  const history = createMemoryHistory();
  const { getByText } = render(
    <Router history={history}>
      <App />
    </Router>
  );
  
  fireEvent.click(getByText('用户中心'));
  expect(history.location.pathname).toBe('/user');
});
```

### 6.2 路由集成测试
```js
// Cypress测试
describe('Route Test', () => {
  it('should navigate to user page', () => {
    cy.visit('/');
    cy.get('a[href="/user"]').click();
    cy.url().should('include', '/user');
    cy.get('h1').should('contain', '用户中心');
  });
});
``` 