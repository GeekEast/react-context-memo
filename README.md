
### Memo基础
- **小知识**: **父组件**的重新渲染会引起**子组件**的`props`发生改变, 所以memo默认是`shallow compare`
- **何时使用**: 取消父组件引起的`不必要`子组件渲染时
- **针对对象**: `memo`仅仅针对`props`, 不会干涉子组件的`state`或者`store`或者`context`引起的渲染
- **默认比较方法**: `shallow compare`， 修改:
```javascript
import { memo } from 'react';
const myComponent = (props) => {...}
const areEqual = (prevProps, nextProps) => {...}
export default memo(MyComponent, areEqual);
```

### SmartStrictEqual
- **讨论前提**: **data** is `immutable`
- **注意事项**: **function** is `mutable`
- 引用比较: 针对**0**个`{}`的有效
- [浅比较](https://github.com/facebook/react/blob/v16.8.6/packages/shared/shallowEqual.js): 针对只有**1**个`{}`的对象有效
- [深比较](https://github.com/epoberezkin/fast-deep-equal): 针对**大于1**个`{}`的对象有效

```javascript
import deepEqual from 'fast-deep-equal';
import { equal } from 'fast-shallow-equal';

const getType = (sth) => {
  return Object.prototype.toString.call(sth).slice(8, -1);
}

const deepObject = (obj) => {
  const keys = Object.keys(obj);
  for (let i = 0; i < keys.length; i++) {
    const type = getType(obj[keys[i]]);
    if (type === 'Object' || type === 'Array') return true
  }
  return false
}

export const smartStrictEqual = (prev, next) => {
  const prevType = getType(prev);
  const nextType = getType(next);
  if (prevType !== nextType) return Object.is(prev, next);
  if (prevType === 'Array') return deepEqual(prev, next);
  if (prevType !== 'Object') return Object.is(prev, next)
  if (deepObject(prev) || deepObject(next)) return deepEqual(prev, next)
  return equal(prev, next)
}
```

### memo在组件Composition模式下的失效问题
#### 组合的两种方式
- 作为内部元素
```javascript
// A的re-render会引起B的re-render
const ComponentA = () => (
  <ComponentB/>
)
```
- 作为`props`传入
```javascript
// A的re-render不会引起B的re-render
// App的re-render会引起A和B的re-render
const App = () => (
  <ComponentA>
    <ComponentB/>
  <ComponentA/>
)
```
#### 两者区别
- **内部元素**: A的re-render会引起B的re-render
- **props传入**: 
  - **App**的re-render才会引起**B**的re-render，**A**不会; 
  - **B**的re-render势必引起**A**的re-render, 因为**B**作为`props`传入了**A**
  - 对**A**使用memo是无效的，`因为children中包含函数`，结果一定不同
#### 实例分析
```javascript
const C0 = (props) => {
  return (
    <div>
      C0 Component
      <C1>
          <C2>
            <C3/>
          </C2>
      </C1>
    </div>
  )
}
```
- 背景: 无任何`memo`
  - 问题: `C0` re-render时, 哪些组件会跟着re-render?
  - 分析: `C1`, `C2`, `C3`都会re-render, 因为**字面上**，它们都是`C0`的子组件
- 背景: 除`C0`外全部使用`memo`, 采用`smartStrictEqual`方法
  - 问题: `C0`re-render时, 哪些组件会跟着re-render?
  - 分析:
    - `C3`的`props`中**无**children，使用memo**能阻止**渲染;
    - `C2`的`props`中**有**children，memo**无法**阻止渲染;
    - `C1`的`props`中**有**children, memo**无法**阻止渲染;
- 解决方案:
  - 自定义比较方法，忽略对children的比较
  - 作为props的函数，在传入前要进行useCallback, 要注意添加适当的deps

### Patterns
<div style="text-align:center; margin:auto">
<img width="200" height="200" src="img/2020-02-27-16-11-46.png">
</div>

#### Container Pattern
- `Container`和`Contained`之间存在**数据传递**
- `灰层数据`传入需要通过`Container`
- `Container`和`Contained`**强**`耦合`
<div style="text-align:center; margin:auto weight:100px height: 50px">
<img width="200" height="300" src="img/2020-02-27-16-10-47.png"></div>

#### High Order Component Pattern
- `High Order Component`和`Low Order Component`之间存在**数据传递**
- `灰层数据`传入需要通过`High Order Component`
- `High Order Component`和`Low Order Component`**低**`耦合`
- 能够优雅地`多层嵌套`
<div style="text-align:center; margin:auto"><img width="200" height="300" src="img/2020-02-27-16-11-03.png"></div>

#### Render Props Pattern
- `Provider`和`Render Component`之间存在**数据传递**
- `灰层数据`传入**无需**经过`Provider`层中继
- `Provider`和`Render Component`**低**`耦合`
- 多层嵌套可以说是**非常丑陋**
- `可读性`很强，能够一眼看出`组件间的关系`
<div style="text-align:center; margin:auto"><img width="450" height="400" src="img/2020-02-27-16-11-19.png"></div>

#### Composite Component Pattern
- `Parent`和`Children`之间不存在**数据传递**
- `灰层数据`传入**无需**经过`Provider`层中继
- `Parent`和`Children`**弱**`耦合`
- 可以优雅地`多层嵌套`，就像`HTML`一样
- **可读性强**
<div style="text-align:center; margin:auto"><img width="450" height="400" src="img/2020-02-27-16-11-35.png"></div>



