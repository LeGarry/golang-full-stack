# 前端组件开发

## 1. React组件开发

### 1.1 函数组件
```jsx
import React, { useState, useEffect } from 'react';

// 基础函数组件
function Button({ text, onClick }) {
  return (
    <button onClick={onClick}>
      {text}
    </button>
  );
}

// Hook使用
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    document.title = `点击了 ${count} 次`;
  }, [count]);
  
  return (
    <div>
      <p>点击次数: {count}</p>
      <Button 
        text="点击" 
        onClick={() => setCount(count + 1)} 
      />
    </div>
  );
}
```

### 1.2 自定义Hook
```jsx
// 自定义Hook
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });
  
  const setValue = value => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  };
  
  return [storedValue, setValue];
}

// 使用自定义Hook
function App() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  
  return (
    <div className={`app ${theme}`}>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        切换主题
      </button>
    </div>
  );
}
```

## 2. Vue组件开发

### 2.1 组件基础
```vue
<!-- 基础组件 -->
<template>
  <div class="card">
    <div class="card-header">
      <slot name="header"></slot>
    </div>
    <div class="card-body">
      <slot></slot>
    </div>
    <div class="card-footer">
      <slot name="footer"></slot>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Card',
  props: {
    title: String
  }
}
</script>

<style scoped>
.card {
  border: 1px solid #ddd;
  border-radius: 4px;
}
</style>

<!-- 使用组件 -->
<template>
  <Card>
    <template #header>
      <h3>标题</h3>
    </template>
    <p>内容</p>
    <template #footer>
      <button>确定</button>
    </template>
  </Card>
</template>
```

### 2.2 组合式API
```vue
<template>
  <div>
    <p>计数: {{ count }}</p>
    <button @click="increment">增加</button>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';

const count = ref(0);
const props = defineProps({
  initial: Number
});

const emit = defineEmits(['change']);

function increment() {
  count.value++;
  emit('change', count.value);
}

watch(count, (newValue) => {
  console.log('count changed:', newValue);
});

onMounted(() => {
  count.value = props.initial || 0;
});
</script>
```

## 3. 组件设计模式

### 3.1 复合组件模式
```jsx
// React复合组件
const Select = {
  Root: ({ children, ...props }) => (
    <div className="select" {...props}>
      {children}
    </div>
  ),
  
  Option: ({ children, value, ...props }) => (
    <div className="select-option" {...props}>
      {children}
    </div>
  ),
  
  Trigger: ({ children, ...props }) => (
    <button className="select-trigger" {...props}>
      {children}
    </button>
  )
};

// 使用复合组件
function App() {
  return (
    <Select.Root>
      <Select.Trigger>选择选项</Select.Trigger>
      <Select.Option value="1">选项1</Select.Option>
      <Select.Option value="2">选项2</Select.Option>
    </Select.Root>
  );
}
```

### 3.2 高阶组件
```jsx
// 高阶组件
function withLoading(WrappedComponent) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) {
      return <div>加载中...</div>;
    }
    return <WrappedComponent {...props} />;
  };
}

// 使用高阶组件
const UserListWithLoading = withLoading(UserList);

function App() {
  return <UserListWithLoading isLoading={true} users={[]} />;
}
```

## 4. 组件性能优化

### 4.1 React优化
```jsx
// memo使用
const MemoizedComponent = React.memo(function MyComponent(props) {
  return (
    <div>{props.value}</div>
  );
});

// useMemo使用
function ExpensiveComponent({ data }) {
  const processedData = useMemo(() => {
    return expensiveOperation(data);
  }, [data]);
  
  return <div>{processedData}</div>;
}

// useCallback使用
function Parent() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []);
  
  return <Child onClick={handleClick} />;
}
```

### 4.2 Vue优化
```vue
<!-- 使用v-show代替v-if -->
<template>
  <div v-show="isVisible">
    <!-- 内容 -->
  </div>
</template>

<!-- 使用computed属性 -->
<script setup>
import { ref, computed } from 'vue';

const items = ref([]);
const filteredItems = computed(() => {
  return items.value.filter(item => item.active);
});
</script>

<!-- 使用v-once -->
<template>
  <div v-once>
    <!-- 静态内容 -->
  </div>
</template>
```

## 5. 组件测试

### 5.1 单元测试
```jsx
// React测试
import { render, fireEvent } from '@testing-library/react';

test('按钮点击测试', () => {
  const handleClick = jest.fn();
  const { getByText } = render(
    <Button text="点击" onClick={handleClick} />
  );
  
  fireEvent.click(getByText('点击'));
  expect(handleClick).toHaveBeenCalled();
});

// Vue测试
import { mount } from '@vue/test-utils';

test('计数器测试', async () => {
  const wrapper = mount(Counter);
  
  await wrapper.find('button').trigger('click');
  expect(wrapper.text()).toContain('计数: 1');
});
```

### 5.2 快照测试
```jsx
// Jest快照测试
import renderer from 'react-test-renderer';

test('组件快照', () => {
  const tree = renderer
    .create(<Button text="测试" />)
    .toJSON();
  expect(tree).toMatchSnapshot();
});
```

## 6. 组件文档

### 6.1 Storybook
```js
// Button.stories.js
import { Button } from './Button';

export default {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    variant: {
      control: { type: 'select', options: ['primary', 'secondary'] }
    }
  }
};

const Template = (args) => <Button {...args} />;

export const Primary = Template.bind({});
Primary.args = {
  variant: 'primary',
  text: '主要按钮'
};

export const Secondary = Template.bind({});
Secondary.args = {
  variant: 'secondary',
  text: '次要按钮'
};
```

### 6.2 组件注释
```jsx
/**
 * 按钮组件
 * @component
 * @param {string} text - 按钮文本
 * @param {string} [variant='primary'] - 按钮样式变体
 * @param {function} onClick - 点击事件处理函数
 * @example
 * return (
 *   <Button
 *     text="点击"
 *     variant="primary"
 *     onClick={() => console.log('clicked')}
 *   />
 * )
 */
function Button({ text, variant = 'primary', onClick }) {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {text}
    </button>
  );
}
``` 