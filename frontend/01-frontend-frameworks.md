# 前端框架

## 1. React 基础

### 1.1 组件基础
```jsx
// 函数组件
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}

// 类组件
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}

// 使用组件
const element = <Welcome name="张三" />;
```

### 1.2 Hooks使用
```jsx
import { useState, useEffect } from 'react';

function Example() {
  // 状态Hook
  const [count, setCount] = useState(0);
  
  // 效果Hook
  useEffect(() => {
    document.title = `点击了 ${count} 次`;
  }, [count]);
  
  return (
    <div>
      <p>你点击了 {count} 次</p>
      <button onClick={() => setCount(count + 1)}>
        点击我
      </button>
    </div>
  );
}
```

## 2. Vue.js 基础

### 2.1 组件定义
```vue
<!-- Vue 2.x -->
<template>
  <div>
    <h1>{{ message }}</h1>
    <button @click="increment">点击次数: {{ count }}</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      message: 'Hello Vue!',
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}
</script>

<!-- Vue 3.x Composition API -->
<script setup>
import { ref } from 'vue'

const message = ref('Hello Vue!')
const count = ref(0)

function increment() {
  count.value++
}
</script>
```

### 2.2 生命周期
```js
export default {
  beforeCreate() {
    // 实例创建前
  },
  created() {
    // 实例创建后
  },
  beforeMount() {
    // 挂载前
  },
  mounted() {
    // 挂载后
  },
  beforeUpdate() {
    // 更新前
  },
  updated() {
    // 更新后
  },
  beforeDestroy() {
    // 销毁前
  },
  destroyed() {
    // 销毁后
  }
}
```

## 3. Angular 基础

### 3.1 组件定义
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: `
    <h1>Hello {{name}}!</h1>
    <button (click)="increment()">点击次数: {{count}}</button>
  `
})
export class HelloComponent {
  name: string = 'Angular';
  count: number = 0;
  
  increment() {
    this.count++;
  }
}
```

### 3.2 服务和依赖注入
```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  getUsers() {
    return fetch('/api/users');
  }
}

@Component({
  selector: 'app-users',
  template: `<ul><li *ngFor="let user of users">{{user.name}}</li></ul>`
})
export class UsersComponent {
  users = [];
  
  constructor(private userService: UserService) {
    this.userService.getUsers().then(users => this.users = users);
  }
}
```

## 4. 前端工程化

### 4.1 Webpack配置
```js
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        use: 'babel-loader',
        exclude: /node_modules/
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader']
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html'
    })
  ]
};
```

### 4.2 Babel配置
```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-react"
  ],
  "plugins": [
    "@babel/plugin-transform-runtime",
    "@babel/plugin-proposal-class-properties"
  ]
}
```

## 5. 性能优化

### 5.1 代码分割
```js
// React懒加载
const OtherComponent = React.lazy(() => import('./OtherComponent'));

function MyComponent() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <OtherComponent />
    </Suspense>
  );
}

// Vue懒加载
const OtherComponent = () => import('./OtherComponent.vue')
```

### 5.2 性能监控
```js
// React性能分析
import { Profiler } from 'react';

function onRenderCallback(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) {
  console.log(`组件 ${id} 渲染耗时: ${actualDuration}ms`);
}

<Profiler id="Navigation" onRender={onRenderCallback}>
  <Navigation />
</Profiler>
```

## 6. 前端测试

### 6.1 Jest单元测试
```js
// sum.test.js
import { sum } from './sum';

test('adds 1 + 2 to equal 3', () => {
  expect(sum(1, 2)).toBe(3);
});

// 组件测试
import { render, fireEvent } from '@testing-library/react';
import Counter from './Counter';

test('counter increments when clicked', () => {
  const { getByText } = render(<Counter />);
  const button = getByText(/点击/i);
  
  fireEvent.click(button);
  
  expect(getByText(/点击次数: 1/i)).toBeInTheDocument();
});
```

### 6.2 E2E测试
```js
// Cypress测试
describe('Counter', () => {
  it('should increment when clicked', () => {
    cy.visit('/');
    cy.get('button').click();
    cy.contains('点击次数: 1');
  });
});
```

## 7. 前端安全

### 7.1 XSS防护
```js
// React自动转义
const title = response.potentiallyMaliciousInput;
const element = <h1>{title}</h1>;

// 手动转义
import DOMPurify from 'dompurify';

function createMarkup(html) {
  return {
    __html: DOMPurify.sanitize(html)
  };
}

function MyComponent() {
  return <div dangerouslySetInnerHTML={createMarkup(data)} />;
}
```

### 7.2 CSRF防护
```js
// Axios配置
axios.defaults.withCredentials = true;
axios.defaults.headers.common['X-CSRF-TOKEN'] = getCsrfToken();

// 获取CSRF Token
function getCsrfToken() {
  return document.querySelector('meta[name="csrf-token"]').getAttribute('content');
}
```

## 8. 最佳实践

### 8.1 代码规范
```js
// ESLint配置
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:react/recommended'
  ],
  rules: {
    'react/prop-types': 'error',
    'no-console': 'warn'
  }
}
```

### 8.2 目录结构
```
src/
  ├── components/      # 通用组件
  ├── pages/          # 页面组件
  ├── services/       # API服务
  ├── utils/          # 工具函数
  ├── hooks/          # 自定义Hooks
  ├── store/          # 状态管理
  ├── assets/         # 静态资源
  └── styles/         # 样式文件
``` 