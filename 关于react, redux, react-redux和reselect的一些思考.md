<font face="Times New Roman">
# <center>关于react, redux, react-redux和reselect的一些思考</center>

我在很早之前就知道reselect这个库了，它可以将selector的计算结果缓存下来，避免不必要的重复计算，可以用来做性能优化。因为时间等一些因素，一直没有将其应用到实际项目中去。随着项目的发展，页面中接口请求数的增加，引起了很多与性能相关的问题。尤其是手机端app，用户一直反馈说app启动慢、页面跳转慢。去年十二月，借着项目有较大调整的时机，我被抽调过来做一些性能优化相关的工作。我在整个过程中，我重新看了reselect相关的文档，并在git上找了一些相关的讨论，由此萌生出了许多关于react, redux, react-redux以及reselect的思考，便想着将它们记录下来，分享给大家，希望大家阅读完本文后能有所收获。

本文主要围绕‘因store中无关数据改变而引起界面渲染’这一问题展开，并结合react-redux的源码，分析了导致这一问题的原因，并给出了相关解决方案。同时就reselect在实际应用中的一些局限性展开了讨论。最后提到了Per-Component Memoization这个概念，以及如何利用react-redux的这一特性来做性能优化。最后，本文中还穿插了一些关于‘在什么地方做connect’，‘store中应该采取什么样的数据模型’等基本问题的讨论。

## 一、从遇到的问题说起

### 1. 场景描述

现假设有store结构如下：

![](http://upload-images.jianshu.io/upload_images/1042695-dd5586c7454bcd4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

界面如下：

![](http://upload-images.jianshu.io/upload_images/1042695-8b3e14cec09ee419.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

其中，demo\_1，demoe\_2为reducer key，CounterView\_1，CounterView\_2通过connect分别显示demo\_1，demo\_2中的counter值；当点击Button 1和Button 2时，分别通过DEMO\_ACTION\_1和DEMO\_ACTION\_2来更新demo\_1，demo\_2中的counter值。

### 2. 期望结果

当点击Button 1时，demo\_1中counter值加一，CounterView\_1重新渲染以显示最新的counter值，CounterView\_2不重新渲染（因为demo\_2中的counter值没有改变）；反之亦然

### 3. 部分代码实现

#### src文件目录结构如下：

```javascript
./src
├── App.css
├── App.js   
├── App.test.js
├── CounterView1.js
├── CounterView2.js
├── constants.js
├── index.css
├── index.js
├── logo.svg
├── registerServiceWorker.js
└── store
    ├── createStore.js
    ├── demoReducer_1.js
    ├── demoReducer_2.js
    └── reducers.js
```

#### view部分代码如下：

```javascript
//
// src/app.js
//
import React, { Component } from 'react'
import { createAction } from 'redux-actions'

import logo from './logo.svg'
import * as actionTypes from './constants'

import CounterView1 from './CounterView1'
import CounterView2 from './CounterView2'
import { getStore } from './index'

class App extends Component {
  render() {
    console.log('......App props:', this.props)
    return (
      <div style={{ marginTop: 20, marginLeft: 20 }}>
        <button onClick={this.buttonEvent_1}>
          Button 1
        </button>
        <button
          onClick={this.buttonEvent_2}
          style={{ marginLeft: 20 }}
        >
          Button 2
        </button>
        <CounterView1 />
        <CounterView2 />
      </div>
    )
  }

  buttonEvent_1 = () => {
    let store = getStore()
    if (!store) {
      console.log('buttonEvent_1')
      return
    }
    store.dispatch(createAction(actionTypes.DEMO_ACTION_1)({}))
  }

  buttonEvent_2 = () => {
    let store = getStore()
    if (!store) {
      console.log('buttonEvent_2')
      return
    }
    store.dispatch(createAction(actionTypes.DEMO_ACTION_2)({}))
  }
}

export default App;


//
// src/CounterView1.js
//
import React, { Component } from 'react'
import { connect } from 'react-redux'
import { bindActionCreators } from 'redux'

class CounterView1 extends Component {
  render () {
    console.log('......CounterView1 props:', this.props)
    return (
      <div style={{
        backgroundColor: '#00ff00',
        width: 300,
        height: 100
      }}>
        <h1 className="App-title">CounterView 1: {this.props.demoData.counter}</h1>
      </div>
    )
    
  }
}

const mapStateToProps = (state) => {
  // 拷贝demo_1以避免直接修改store
  let demo_1 = Object.assign({}, state.demo_1)
  if (!demo_1.counter) {
    demo_1.counter = 0
  }
  // 考虑到实际应用场景的复杂性，可能还存在demoData_1, demoData_2 ...，故在此做了一层封装
  return { demoData: demo_1 }
} 

const mapDispatchToPeops = (dispatch) => {
  return bindActionCreators({}, dispatch)
}
export default connect(mapStateToProps, mapDispatchToPeops)(CounterView1)


//
// src/CounterView2.js
//
import React, { Component } from 'react'
import { connect } from 'react-redux'
import { bindActionCreators } from 'redux'

class CounterView2 extends Component {
  render () {
    console.log('......CounterView2 props:', this.props)
    return (
      <div style={{
        backgroundColor: '#ff0000',
        width: 300,
        height: 100
      }}>
        <h1 className="App-title">CounterView 2: {this.props.demoData.counter}</h1>
      </div>
    )
  }
}

const mapStateToProps = (state) => {
  let demo_2 = Object.assign({}, state.demo_2)
  if (!demo_2.counter) {
    demo_2.counter = 0
  }
  return { demoData: demo_2 }
} 

const mapDispatchToPeops = (dispatch) => {
  return bindActionCreators({}, dispatch)
}
export default connect(mapStateToProps, mapDispatchToPeops)(CounterView2)
```

#### reducer部分代码如下：

```javascript
// src/store/demoReducer_1.js
import * as actionTypes from '../constants'

export default function reducer (state = {}, action) {
  switch (action.type) {
    case actionTypes.DEMO_ACTION_1:
      return handleDemoAction1(state, action)

    default:
      return state
  }
}

function handleDemoAction1 (state, action) {
  let counter = state.counter || 0
  state = Object.assign({}, state, { counter: counter + 1 })
  return state
}


// src/store/demoReducer_2.js
import * as actionTypes from '../constants'

export default function reducer (state = {}, action) {
  switch (action.type) {
    case actionTypes.DEMO_ACTION_2:
      return handleDemoAction2(state, action)

    default:
      return state
  }
}

function handleDemoAction2 (state, action) {
  let counter = state.counter || 0
  state = Object.assign({}, state, { counter: counter + 1 })
  return state
}
```

#### 4. 测试结果

运行上述代码，当点击Button 2时，CounterView\_2显示最新的counter值，CounterView\_1显示的值不变，符合预期；但是通过控制台输出发现，CounterView\_1和CounterView\_2都重新渲染了，与预期相矛盾。控制台截图如下：

![action_log.png](http://upload-images.jianshu.io/upload_images/1042695-81fbb94d53f7a4d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

#### 5. 思考：为什么store中不相关的数据的改变会引起界面的重新渲染？

##### (1) 简单回顾一下redux相关知识

redux中有三个核心元素：store，reducer和action。其中store作为应用的唯一数据源，用于存储应用在某一时刻的状态数据；store是只读的，且只能通过action来改变，暨通过action将状态1下的store转换为状态2下的store；reducer用于定义状态转换的具体细节，并与action相对应；应用的UI部分可以通过store提供的subscribe方法来监听store的改变，当状态1下的store转换为状态2下的store时，所有通过subscribe方法注册的监听器(listener)都会被调用。

##### (2) connect返回的是一个监听了store改变的HOC

关于HOC，请阅读react官方文档: [higher order components](https://reactjs.org/docs/higher-order-components.html)，不再赘述。这里我们主要讨论connect相关细节。

*__注意：本文中所有react-redux源码片段均来自react-redux@4.4.6，下文中将不再重复强调。__*

下面是react-redux的部分源码：

```javascript
export default function connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {
  // ...
  return function wrapWithConnect(WrappedComponent) {
    // ...
    class Connect extends Component {
	   // ...
	   trySubscribe() {
	     if (shouldSubscribe && !this.unsubscribe) {
	       this.unsubscribe = this.store.subscribe(this.handleChange.bind(this))
	       this.handleChange()
	     }
	   }
	
	   tryUnsubscribe() {
	     if (this.unsubscribe) {
	       this.unsubscribe()
	       this.unsubscribe = null
	     }
	   }
	
	   componentDidMount() {
	     this.trySubscribe()
	   }
	  
	   componentWillUnmount() {
	 	  this.tryUnsubscribe()
	     // ...
	   }
	  
	   handleChange() {
	     if (!this.unsubscribe) { return }
	
	     const storeState = this.store.getState()
	     const prevStoreState = this.state.storeState
	     if (pure && prevStoreState === storeState) {
	       return
	     }
	     if (pure && !this.doStatePropsDependOnOwnProps) {
	       // ...
	     }
	
	     this.hasStoreStateChanged = true
	     this.setState({ storeState })
	   }
	   // ...
	 }
	 
	 render() {
	   const {
	     // ...
	     renderedElement
	   } = this
	   
	   // ...
	   if (!haveMergedPropsChanged && renderedElement) {
	     return renderedElement
	   }
	   if (withRef) {
	     this.renderedElement = createElement(WrappedComponent, {
	       ...this.mergedProps,
	       ref: 'wrappedInstance'
	     })
	   } else {
	     this.renderedElement = createElement(WrappedComponent,
	       this.mergedProps
	     )
	   }
	   return this.renderedElement
	 }
	 // ...
	 return hoistStatics(Connect, WrappedComponent)
  }
  // ...
}
```
通过这部分源码可知，Connect HOC在componentDidMount中通过store的subscribe方法来监听了store的改变，在handleChange回调中，首先判断store是否改变，如果store改变，通过setState方法来重新渲染Connect HOC。

注意，在Connect HOC的render方法中，当haveMergedPropsChanged = false且renderedElement存在时，会返回已存在的renderedElement，此时WrappedComponent不会被重新渲染；否则会创建新的renderedElement并返回，此时会导致WrappedComponent重新渲染。

##### (3) 为什么是haveMergedPropsChanged？

connect方法接收四个参数mapStateToProps, mapDispatchToProps, mergeProps和options，我们最熟悉、使用得最多的是前两个参数，当需要拿到WrappedComponent的引用是，我们会使用第四个参数options中的withRef属性，{ withRef: true }。对于第三个参数mergeProps接触得比较少(至少在我做过的项目中接触得比较少)，在解释mergeProps之前，需要知道Connect HOC主要处理哪几部分数据。

Connect HOC主要处理三种类型的数据：stateProps，dispatchProps和ownProps。其中stateProps由mapStateToProps计算得到，dispatchProps由mapDispatchToProps计算得到，ownProps是父控件传递给Connect HOC的。

根据react-redux[文档](https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options)，mergeProps的定义是：

```
[mergeProps(stateProps, dispatchProps, ownProps): props] (Function)
```
mergeProps将stateProps，dispatchProps和ownProps作为参数，并返回一个props，这个props暨是最终传递给WrappedComponent的props。如果没有为connect指定mergeProps，则默认使用```Object.assign({}, ownProps, stateProps, dispatchProps)```。在使用默认值的情况下，如果stateProps和ownProps中存在同名属性，stateProps中的对应值会覆盖ownProps中的值。

下面给出几个mergeProps的使用场景：

```
/*
 * 当stateProps和dispatchProps的内部属性过多时(尤其是dispatchProps),
 * 默认情况下，mergeProps会依次将这些属性复制到WrappedComponent的props中，
 * 从而导致WrappedComponent的props过大，从而增大调试的复杂度。
 * 
 * mergeProps的这种实现可以有效地避免上述问题。
 */ 
function mergeProps (stateProps, dispatchProps, ownProps) {
  return {
    stateProps, dispatchProps, ownProps
  }
}

/*
 * 现假设有一个文章列表，列表中的任意一篇文章都含有id, abstract, creator等
 * 信息。又因为某些原因，这些信息只存在于Component的state中，而没存到store中(
 * 在Component中直接调接口，并将结果以setState方式保存)。当进入某篇文章详情时，
 * 已存在于客户端的数据应立即展示出来，如creator。为达到这一目的，需要通过ownProps
 * 来传递这些数据。
 * 
 * 在文章详情页面，会请求文章的详细数据，并存储到store中，然后通过mapStateToProps
 * 从store获取文章相关数据。当接口返回之前，通常会使用一些默认值来替代真实值。因此，
 * stateProps.creator可能是是默认值{id: undefined, avatar: undefined, ...}。
 * 
 * 又因为mergeProps在默认情况下，ownProps中同名属性会被stateProps中的值覆盖，暨
 * 最终从WrappedComponent的props中取得的creator是未初始化的默认状态，也就不能在
 * 进入文章详情后马上显示creator相关信息，即使文章列表中已存在相关的数据。
 * 
 * 利用mergeProps可以在一定程度上解决这个问题，下面是示例代码。
 */
function mergeProps (stateProps, dispatchProps, ownProps) {
  if (stateProps.creator && ownProps.creator) {
    if (!stateProps.creator.id) {
      delete stateProps.creator
    }
  }
  return Object.assign({}, ownProps, stateProps, dispatchProps)
}

/*
 * 完全丢弃stateProps，dispatchProps和ownProps，并返回其他对象
 */
function mergeProps (stateProps, dispatchProps, ownProps) {
  return { a: 1, b: 2, ... }
}
```

##### (4) WrappedComponent在什么情况下会被重新渲染？

由上述Connect HOC的render方法的片段可知，当haveMergedPropsChanged = true或renderedElement不存在时，WrappedComponent会重新渲染。其中renderedElement是对上一次调用createElement的结果的缓存，除第一次执行Connect HOC的render方法外，createElement一直有值(不考虑出错情况)。因此WrappedComponent是否重新渲染由haveMergedPropsChanged的值决定，暨mergedProps是否改变。mergedProps改变，WrappedComponent重新渲染；反之则不重新渲染。

下面是Connect HOC的render方法中的部分逻辑：

```javascript
render () {
  const {
    // ...
    renderedElement
  } = this
  
  // ...
  
  let haveStatePropsChanged = false
  let haveDispatchPropsChanged = false
  haveStatePropsChanged = this.updateStatePropsIfNeeded()
  haveDispatchPropsChanged = this.updateDispatchPropsIfNeeded()

  let haveMergedPropsChanged = true
  if (
    haveStatePropsChanged ||
    haveDispatchPropsChanged ||
    haveOwnPropsChanged
  ) {
    haveMergedPropsChanged = this.updateMergedPropsIfNeeded()
  } else {
    haveMergedPropsChanged = false
  }

  if (!haveMergedPropsChanged && renderedElement) {
    return renderedElement
  }
  if (withRef) {
    // ...
  } else {
    // ...
  }
}
```

上述代码中，当stateProps，dispatchProps和ownProps三者中任意一者改变时，便会去检测mergedProps是否改变。

```
updateMergedPropsIfNeeded() {
  const nextMergedProps = computeMergedProps(this.stateProps, this.dispatchProps, this.props)
  if (this.mergedProps && checkMergedEquals && shallowEqual(nextMergedProps, this.mergedProps)) {
    return false
  }
  
  this.mergedProps = nextMergedProps
  return true
}
```

__当mergeProps为默认值时，通过简单的推导可知，stateProps，dispatchProps和ownProps三者中任意一者改变时，mergedProps也会改变，从而导致WrappedComponent的重新渲染。__

##### (5) 如何判断ownProps是否改变？

在Connect HOC中，在componentWillReceiveProps中判断ownProps是否改变，代码如下：

```javascript
componentWillReceiveProps(nextProps) {
  if (!pure || !shallowEqual(nextProps, this.props)) {
    this.haveOwnPropsChanged = true
  }
}
```
其中pure为一可选配置，其值取自connect的第四个参数options，默认为true(关于pure的更多细节，请自行阅读react-redux源码，这里不做过多讨论)。

```
const { pure = true, withRef = false } = options
```

若pure为默认值，当父控件传递给Connect HOC的props改变时，ownProps改变。

##### (6) 如何判断stateProps和dispatchProps是否改变？以stateProps为例

Connect HOC通过在render中调用updateStatePropsIfNeeded方法来判断stateProps是否改变：

```
updateStatePropsIfNeeded() {
  const nextStateProps = this.computeStateProps(this.store, this.props)
  if (this.stateProps && shallowEqual(nextStateProps, this.stateProps)) {
    return false
  }

  this.stateProps = nextStateProps
  return true
}
      
computeStateProps(store, props) {
  if (!this.finalMapStateToProps) {
    return this.configureFinalMapState(store, props)
  }

  const state = store.getState()
  const stateProps = this.doStatePropsDependOnOwnProps ?
    this.finalMapStateToProps(state, props) :
    this.finalMapStateToProps(state)

  if (process.env.NODE_ENV !== 'production') {
    checkStateShape(stateProps, 'mapStateToProps')
  }
  return stateProps
}
```
由此可知，Connect HOC是通过对this.stateProps和nextStateProps的shallowEqual来判断stateProps是否改变的。dispatchProps与stateProps类似，不再重复讨论。

##### (7) 回答我们的问题：为什么store中不相关的数据的改变会引起界面的重新渲染？

首先我们再看一下上述例子中mapStateToProps的实现:

```
const mapStateToProps = (state) => {
  let demo_1 = Object.assign({}, state.demo_1)
  if (!demo_1.counter) {
    demo_1.counter = 0
  }
  return { demoData: demo_1 }
} 
```
注意这一句```let demo_1 = Object.assign({}, state.demo_1)```，每次调用mapStateToProps，都会创建一个新的Object实例并赋值给demo\_1。当连续调用两次mapStateToProps，则有：

```
let thisStateProps = mapStateToProps(state)
let nextStateProps = mapStateToProps(state)
assert(thisStateProps.demoData !== nextStateProps.demoData)
```

在updateStatePropsIfNeeded中，会将nextStateProps和thisStateProps做shallowEqual，因为```thisStateProps.demoData !== nextStateProps.demoData```，updateStatePropsIfNeeded会返回true，stateProps改变。

综上所叙，当点击Button 2时，会dispatch出DEMO\_ACTION\_2并改变store。CounterView1的Connect HOC的handleChange回调检测到store改变，并通过setState的方式让Connect HOC重新渲染。在Connect HOC的render方法中，因为```this.stateProps.demoData !== nextStateProps.demoData```，this.updateStatePropsIfNeeded返回true，表示stateProps发生了改变。又由我们在第四小点‘WrappedComponent在什么情况下会被重新渲染？’中得出的结论可知，当stateProps改变时，CounterView1会被重新渲染。

#### 6. 再思考：我们该如何写mapStateToProps？

从上述分析可知，当点击Button 2时，CounterView1之所以会被重新渲染，是因为每次调用mapStateToProps时，都会创建新的Object实例并赋给demo\_1，这导致了updateStatePropsIfNeeded中的shallowEqual失败，stateProps改变，CounterView1重新渲染。

##### (1) 优化上述mapStateToProps写法

先看下面这段代码：

```
let obj = { p1: { a: 1 }, p2: { b: 2 } }
let obj2 = Object.assign({}, obj, { p3: { c: 3 } })
assert(obj.p1 === obj2.p1)
```
由此可知，当DEMO\_ACTION\_2改变了store之后，有```thisStore !== nextStore```, 但是```thisStore.demo_1 === nextStore.demo_1```，store中demo\_1所指向的对象并没有发生改变。在上述mapStateToProps的实现中，最大的问题是每次调用mapStateToProps时，stateProps.demoData都会指向新的对象。如果直接将store.demo\_1直接赋给stateProps.demoData呢？修改后的代码如下：

```
const mapStateToProps = (state) => {
  let demo_1 = state.demo_1
  if (!demo_1.counter) {
    demo_1 = { counter: 0 }
  }
  return { demoData: demo_1 }
}
```
执行修改后代码，控制台输出如下：

![](http://upload-images.jianshu.io/upload_images/1042695-3a1fdfddf70bc81d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由该控制台日志可知，当第一次点击Button 1时，CounterView1和CounterView2都重新渲染了；随后点击Button 2时，只有CounterView2重新渲染了，为什么？

分析优化后的mapStateToProps可知，当第一次点击Button 1时，store.demo\_2是初始化时的默认值，因此会进入```if (!demo_2.counter) { demo_2 = { counter: 0 } }```这一逻辑分支，在mapStateToProps中，我们每次都创建了新的默认值。再次优化mapStateToProps如下：

```
const DefaultDemoData = { counter: 0 }
const mapStateToProps = (state) => {
  let demo_1 = state.demo_1
  if (!demo_1.counter) {
    demo_1 = DefaultDemoData
  }
  return { demoData: demo_1 }
}
```
执行此优化代码，控制台输出如下：

![](http://upload-images.jianshu.io/upload_images/1042695-f514aa5d8ae39a08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当点击Button 1或Button 2时，均只有对应的CounterView被重新渲染。

##### (2) 尝试归纳一些基本原则

* 不要在mapStateToProps构建新的对象，直接使用store中对应的对象
* 提供全局的默认值，以使每次返回的默认值都指向同一个对象  

使用immutable时的一个常见的错误写法：

```
let immutableRecord = state['reducer_key'].immutableRecord || DefaultImmutableRecord
immutableRecord = immutableRecord.toJS()
```
每次调用toJS()方法时，都会生成新的对象，从而导致stateProps的改变，重新渲染界面。

##### (3) 我们无法避免在mapStateToProps中构建新的对象

现假设有如下store结构，为了方便操作，此处会使用immutable：

```
new Map({
  feedIdList_1: new List([id_1, id_2, id_3]),
  feedIdList_2: new List([id_1, id_4, id_5]),
  feedById: new Map({
    id_1: FeedRecord_1,
    ...
    id_5: FeedRecord_5
  })
})
```
为了在界面上渲染一个feed列表，例如feedIdList_1，我们会将这个feed列表connect到store上，然后在mapStateToProps中构建出一个feed数组，数组中的每个元素对应一条feed，mapStateToProps部分代码如下：

```
const DefaultList = new List()
const mapStateToProps = (state, ownProps) => {
  let feedIdList = state['reducer_key'].feedIdList_1 || DefaultList
  // 因react无法渲染immutable list，故需要转换为数组
  feedIdList = feedIdList.toJS()
  
  let feedList = feedIdList.map((feedId) => {
    return state['reducer_key'].getIn(['feedById', feedId])
  })
  
  return {
    feedList: feedList,
    // other data
  }
}
```

在上述mapStateToProps的实现中，我次调用mapStateToProps都会创建一个新的feedList对象，由上述的讨论可知，即使feedIdList\_1和id\_1, id\_2, id\_3对应的FeedRecord都没有改变，当store其他部分改变时，也会引起该feed列表的重新渲染。

*注意：当需要在界面上渲染一个列表时，我们一般会选择将这个列表connect到store上，而不是将列表的每个元素分别connect到store上，更多关于在什么地方使用connect，请参考[redux/issues/815](https://github.com/reactjs/redux/issues/815), [redux/issues/1255](https://github.com/reactjs/redux/issues/1255)和[At what nesting level should components read entities from Stores in Flux?](https://stackoverflow.com/questions/25701168/at-what-nesting-level-should-components-read-entities-from-stores-in-flux/25701169#25701169)*

#### 7. 再再思考：我们该如何避免由store中无关数据的改变引起的重新渲染？

##### (1) shouldComponentUpdate

首先我们会想到利用shouldComponentUpdate，通过比较this.state，this.props和nextState，nextProps，判断state和props都是否改变(deepEqual，shallowEqual或其他方法)，从而决定界面是否重新渲染。

虽然利用shouldComponentUpdate可以避免由store中无关数据的改变引起的重新渲染，但每次store改变时，所有的mapStateToProps都会被重新执行，这可能会导致一些性能上的问题。

##### (2) 一个极端的例子

```
const DefaultDemoData = { counter: 0 }
const mapStateToProps = (state, ownProps) => {
  let demo_1 = state.demo_1
  if (!demo_1.counter) {
    demo_1 = DefaultDemoData
  }
  
  // tons of calculation here, for example:
  let counter = demo_1.counter
  for (let i = 0, i < 1000000000000; i++) {
  	counter *= i
  }
  demo_1.counter = counter
  
  return { demoData: demo_1 }
}
```
在这个极端的例子中，mapStateToProps有一段非常耗时的计算。虽然shouldComponentUpdate可以有效地避免重新渲染，我们该如何有效地避免这段复杂的计算呢？

##### (3) 一个新的想法

redux充分利用了纯函数的思想，我们的mapStateToProps其本身也是一个纯函数。纯函数的特点是当输入不变时，多次执行同一个纯函数，其返回结果不变。既然在demo\_1.counter的值未改变的情况下，每次执行完这段耗时操作的返回值都相同，我们能不能将这个结果缓存起来，当demo\_1.counter没有发生改变时，直接去读取这个缓存值呢？修改上述代码如下：

```
const DefaultDemoData = { counter: 0 }

const tonsOfCalculationCb = (counter) => {
	for (let i = 0, i < 1000000000000; i++) {
		counter *= i
	}
	return counter
}

const createTonsOfCalculation = (cb) => {
	let lastCounter, lastResult
	return (counter) => {
		if (!lastResult || counter != lastCounter) {
			lastResult = cb(counter)
		}
		lastCounter = counter
		return lastResult
	}
}

const tonsOfCalculation = createTonsOfCalculation(tonsOfCalculationCb)

const mapStateToProps = (state, ownProps) => {
	let demo_1 = state.demo_1
	if (!demo_1.counter) {
		demo_1 = DefaultDemoData
	}
	demo_1.counter = tonsOfCalculation(demo_1.counter)

	return { demoData: demo_1 }
}
```

在tonsOfCalculation中，我们通过纪录传入的counter值并将其与当前传入的counter做比较来判断输入是否改变，当counter改变时，重新计算并缓存结果；当counter不变且缓存值存在时，直接读取缓存值，从而有效地避免了不必要的耗时计算。

##### (4) 将缓存的思想应用到mapStateToProps中

借助缓存的思想，我们需要达到两个目的:

* 当store中相关依赖数据没有发生改变时，直接从缓存中读取上一次构建的对象，避免重新渲染。例：当feedIdList\_1和id\_1, id\_2, id\_3对应的FeedRecord都没有改变，直接从缓存中读取feedList，避免构建新的对象
* 避免mapStateToProps中可能存在的耗时操作。例：当counter未改变时，直接读取缓存值

*store中相关依赖数据：即mapStateToProps/selector为达到其特定的计算目的而需要从store中读取的数据，以我们的feedList\_1为例，相关依赖数据分别是feedIdList\_1，feedById.id\_1，feedById.id\_2和feedById.id\_3*


修改上述feed列表例子的代码如下：

```
const LRUMap = require('lru_map').LRUMap
const lruMap = new LRUMap(500)

const DefaultList = new List()
const DefaultFeed = new FeedRecord()

const mapStateToProps = (state, ownProps) => {
  const hash = 'mapStateToProps'
  let feedIdList = memoizeFeedIdListByKey(state, lruMap, 'feedIdList_1')
  let hasChanged = feedIdList.hasChanged
  if (!hasChanged) {
    hasChanged = feedIdList.result.some((feedId) => {
      return memoizeFeedById(state, lruMap, feed).hasChanged
    })
  }
  
  if (!hasChanged && lruMap.has(hash)) {
    return lruMap.get(hash)
  }
  
  let feedIds = feedIdList.result.toJS()
  let feedList = feedIds((feedId) => {
    return memoizeFeedById(state, lruMap, feed).result
  })
  // do some other time consuming calculations here
  let result = {
    feedList: feedList,
    // other data
  }
  lruMap.set(hash, result)
  return result
}

function memoizeFeedIdListByKey (state, lruMap, idListKey) {
  const hash = `hasFeedIdListChanged:${idListKey}`
  let cached = lruMap.get(hash)
  let feedIds = state['reducerKey'][idListKey]
  let hasChanged = feedIds && cached !== feedIds
  if (hasChanged) {
    lruMap.set(hash, feedIds)
  }
  return { hasChanged: hasChanged, result: feedIds || DefaultList }
}

function memoizeFeedById (state, lruMap, feedId) {
  const hash = `hasFeedChanged:${feedId}`
  let cached = lruMap.get(hash)
  let feed = state['reducer_key'].getIn(['feedById', feedId])
  let hasChanged = feed && cached !== feed
  if (hasChanged) {
    lruMap.set(hash, feed)
  }
  return { hasChanged: hasChanged, result: feed || DefaultFeed }
}
```
上述代码，首先检测相关的依赖数据是否改变(feedIdList\_1和id\_1, id\_2, id\_3对应的FeedRecord)，如果没有改变且缓存存在，直接返回缓存的数据，界面不会重新渲染；如果发生了改变，重新计算并设置缓存，界面重新渲染。

##### (5) 介绍一个新的库：reselect

[reselect](https://github.com/reactjs/reselect)利用上述思想，通过检测store中相关依赖数据是否改变，来避免mapStateToProps的重复计算，同时避免界面的不必要渲染。下面我们会着重讨论reselect的使用场景及其局限性。

##### (6) 还需要在WrappedComponent中使用shouldComponentUpdate吗？

既然利用缓存的思想，可以在mapStateToProps中避免不必要的界面渲染，我们还需要在WrappedComponent中使用shouldComponentUpdate吗？前面我们有说到，connect HOC主要处理三种类型的数据stateProps，dispatchProps和ownProps，利用缓存的思想可以有效地避免由stateProps和dispatchProps引起的不必要渲染，那么当ownProps改变时会怎样呢？看下面的例子：

```
// src/app.js

render () {
  return (
    <div>
      <CounterView1 otherProps={{ a: 1 }}>
    </div>
  )
}
```
在CounterView1的componentWillReceiveProps中，你会发现```nextProps.otherProps !== this.props.otherProps```，从而导致CounterView1重新渲染。这是因为src/app.js每次重新渲染时，都会构建一个新的otherProps对象并传递给CounterView1。此时，我们可以借助shouldComponentUpdate来避免此类由ownProps引起的不必要渲染。

shouldComponentUpdate还有许多其他的应用场景，但这不属于本文考虑的范畴，故不再一一列举。

## 二、reselect

reselect是基于下述三个原则来设计的：

* selectors可以用来计算衍生数据(derived data)，从而允许Redux只存储最小的、可能的state
* selectors是高效的，一个selector只有在传给它的参数发生改变时，才重新计算
* selectors是可以组合的，它们可以被当作其他selector的输入

在上述第二个原则中，为保证selector的高效性，需要用到前文提到的缓存思想。

*注意：以上三个原则均翻译自reselect文档，更多详情请查看[这里](https://github.com/reactjs/reselect)*

#### 1. 如何使用reselect

下面以feed列表的例子来展示如何使用reselect，部分store结构如下，为方便操作，这里使用了immutable：

```
{
  feed: new Map({
    feedIdList_1: new List([id_1, id_2, id_3]),
    feedIdList_2: new List([id_1, id_4, id_5]),
    feedById: new Map({
      id_1: FeedRecord_1,
      ...
      id_5: FeedRecord_5
    })
  }),
  ...
}

```

以下是部分代码实现：

```javascript
import { createSelector } from 'reselect'

const getFeedById = state => state['feed'].get('feedById')
const getFeedIds = (state, idListKey) => state['feed'].get(idListKey)

const feedListSelectorCb = (feedIds, feedMap) => {
  feedIds = feedIds.toJS ? feedIds.toJS() : feedIds
  let feedList = feedIds.map((feedId) => {
    return feedMap.get(feedId)
  })
}
const feedListSelector = createSelector(getFeedIds, getFeedById, feedListSelectorCb)

const mapStateToProps = (state, ownProps) => {
  const idListKey = 'feedIdList_1'
  let feedList = feedListSelector(state, idListKey)
  return {
    feedList: feedList,
    // other data
  }
}
```

这里，我们利用reselect提供的createSelector方法，创造出了feedListSelector，并在mapStateToProps中调用feedListSelector来计算feedList。feedListSelector的相关依赖数据是feedById和feedIdList\_1，当这两者中任意一个发生改变时，reselect内部机制会判断出这个改变，并调用feedListSelectorCb重新计算新的feedList。稍后我们会详细讨论reselect这一内部机制。

相比之前利用lruMap实现的feed列表，这段代码简洁了许多，但上述代码存在一个问题。

##### (1) 上述代码存在因store中无关数据改变而导致界面重新渲染的问题

上述feedListSelector的相关依赖数据是feedById和feedIdList\_1，通过观察store的结构可知，feedById中存在与feedList\_1无关的数据。也就是说，为了计算出feedList\_1，feedListSelector依赖了与feedList\_1无关的数据，即FeedRecord\_4和FeedRecord\_5。当FeedRecord\_5发生改变时，feedById也随之改变，导致feedListSelectorCb会被重新调用并返回一个新的feedList。由上文的讨论可知，当在mapStateToProps中创建新的对象时，会导致界面的重新渲染。

在FeedRecord\_5改变前和改变后的两个feedList\_1中，虽然feedList\_1中的每个元素都没有发生改变，但feedList\_1本身发生了改变(两个不同的对象)，最后导致界面渲染，这是典型的因store中无关数据改变而引起界面渲染的例子。


##### (2) 一个更复杂的例子

在实际应用中，一个feed还存在创建者属性，同时creator作为一个用户，还可能存在所属组织机构等信息，部分store结构如下：

```
{
  feed: {
    feedIdList_1: new List([feedId_1, feedId_2, feedId_3]),
    feedIdList_2: new List([feedId_1, feedId_4, feedId_5]),
    feedById: new Map({
      feedId_1: new FeedRecord({
        id: feedId_1,
        creator: userId_1,
        ...
      }),
      ...
      feedId_5: FeedRecord_5
    })
  },
  user: {
    userIdList_1: new List([userId_2, userId_3, userId_4])
    userById: new Map({
      userId_1: new UserRecord({
        id: userId_1,
        organization: organId_1,
        ...
      }),
      ...
      userId_3: UserRecord_3
    })
  },
  organization: {
    organById: new Map({
      organId_1: new OrganRecord({
        id: organId_1,
        name: 'Facebook Inc.',
        ...
      }),
      ...
    })
  }
}
```

上述store主要由feed, user和organization三部分组成，它们分别由不同的reducer来更新其内部数据。在渲染feedList\_1时，每条feed都需要展示创建者以及创建者所属组织等信息。为达到这个目的，我们的feedListSelector需要做如下更改。

```javascript
import { createSelector } from 'reselect'

const getFeedById = state => state['feed'].get('feedById')
const getUserById = state => state['user'].get('userById')
const getOrganById = state => state['organization'].get('organById')
const getFeedIds = (state, idListKey) => state['feed'].get(idListKey)

const feedListSelectorCb = (feedIds, feedMap, userMap, organMap) => {
  feedIds = feedIds.toJS ? feedIds.toJS() : feedIds
  let feedList = feedIds.map((feedId) => {
    let feed = feedMap.get(feedId)
    let creator = userMap.get(feed.creator)
    let organization = organMap.get(creator.organization)
    
    feed = feed.set('creator', creator)
    feed = feed.setIn(['creator', 'organization'], organization)
    return feed
  })
}
const feedListSelector = createSelector(
  getFeedIds,
  getFeedById,
  getUserById,
  getOrganById
  feedListSelectorCb
)

const mapStateToProps = (state, ownProps) => {
  const idListKey = 'feedIdList_1'
  let feedList = feedListSelector(state, idListKey)
  return {
    feedList: feedList,
    // other data
  }
}
```

上述代码中，feedListSelector的相关依赖数据是feedIdList\_1，feedById，userById和organById。相对于之前简单的feed列表的例子，这里多了userById和organById这两个依赖。此时会有一个有趣的现象：当我们从服务器端请求userList\_1的数据并存入store中时，会导致feedList\_1的重新渲染，因为userById改变了。从性能的角度考虑，这不是我们期望的结果。

##### (3) 能否通过改变store的结构来解决上述问题呢？

出现上述问题的最主要原因是feedListSelector的相关依赖数据feedById，userById等中含有与feedList\_1无关的数据。那么我们能不能将相关数据存储在一起，这样feedListSelector就不会依赖无关数据了。

以上文提到的简单的feed列表为例，其修改后的store结构如下：

```
{
  feed: new Map({
    feedList_1: new Map({
      idList: new List([id_1, id_2, id_3]),
      feedById: new Map({
        id_1: FeedRecord_1,
        id_2: FeedRecord_2,
        id_3: FeedRecord_3
      })
    }),
    feedList_2: new Map({
      idList: new List([id_1, id_4, id_5]),
      feedById: new Map({
        id_1: FeedRecord_1,
        id_4: FeedRecord_4,
        id_5: FeedRecord_5
      })
    })
  }),
  ...
}
```

这里，每个feedList拥有属于自己的idList和feedById，在渲染feedList\_1时，feedListSelector之需要依赖feedList\_1这一处数据了，修改后的获取feedList的代码如下：

```javascript
import { createSelector } from 'reselect'

const getFeedList = (state, feedListKey) => state['feed'].get(feedListKey)

const feedListSelectorCb = (feedListMap) => {
  let feedMap = feedListMap.get('feedById')
  let feedIds = feedListMap.get('idList').toJS()
  let feedList = feedIds.map((feedId) => {
    return feedMap.get(feedId)
  })
}
const feedListSelector = createSelector(getFeedList, feedListSelectorCb)

const mapStateToProps = (state, ownProps) => {
  const feedListKey = 'feedList_1'
  let feedList = feedListSelector(state, feedListKey)
  return {
    feedList: feedList,
    // other data
  }
}
```

因为我们的feedListSelector不再依赖无关数据，当id\_4或id\_5对应的FeedRecord发生改变时，不会再引起feedList\_1的重新渲染。但时，这种store结构存在以下问题：

* store中存在重复的数据，id\_1对应的FeedRecord同时存在于feedList\_1和feedList\_2中，可能会出现较大的数据冗余
* 当feedList_2中id\_1对应的FeedRecord发生改变时，feedList\_1也不会重新渲染，暨相关数据改变不引起界面渲染的问题

#### 2. 我们该如何定义我们的数据模型: 'Normalized Data Model' vs 'Embedded Data Model'

上文中在分析简单的feed列表时，提到了两种数据模型。第一种是Normalized Data Model，该模型与reselect一起使用时，会导致'因store中无关数据的改变而引起不必要的界面渲染' 的问题；第二种是Embedded Data Model，该模型与reselect一起使用时，存在'store中相关数据改变不引起界面渲染'的问题。那么我们该如何定义store中的数据模型呢？

##### (1) 介绍两个概念：store model和display model，以及一些通用做法

使用Redux时，我们需要处理的数据主要分为两部分：体现应用全局状态的store部分数据和用于渲染特定界面而计算出的衍生数据。这两部分数据一般采用不同的数据模型：

* store model: store中存储数据时采用的数据模型，一般为Normalized Data Model 
* display model: 渲染界面时需要的数据模型，一般为Embedded Data Model

我们通过mapStateToProps和selector将store中Normalized的数据转换成界面需要的Embedded类型数据。

##### (2) 简单分析一下Normalized和Embedded两种数据模型的优缺点

一个Normalized Data Model的例子：

```
{
  feedList: [feedId_1, feedId_2, feedId_3, ...],
  feedById: {
    feedId_1: {
      id: feedId_1,
      title: 'Feed Title',
      content: 'Feed Content',
      creator: userId_1
    },
    feedId_2: { ... },
    ...
  },
  userList: [userId_1, userId_2, ...],
  userById: {
    userId_1: {
      id: userId_1 , nickname: 'nickname', avatar: 'avatar.png', ...
    },
    ...
  }
}
```

一个Embedded Data Model的例子：

```
{
  feedList: [
    {
      id: feedId_1,
      title: 'Feed Title',
      content: 'Feed Content',
      creator: {
        id: userId_1 , nickname: 'nickname', avatar: 'avatar.png', ...
      },
      ...
    },
    ...
  ],
  userList: [
    {
      id: userId_1 , nickname: 'nickname', avatar: 'avatar.png', ...
    },
    ...
  ]
}
```

Normalized Data Model:

* 优点: 通过id来关联数据，数据的存储是扁平化的，无数据冗余，数据一致性高且更新操作比较简单
* 缺点: 为了渲染相关数据，需要de-normalized，暨将数据转换成适合UI渲染的Embedded结构的数据，当数据量较大时，这个过程可能比较耗时；还可能因为前文提到的创建新对象的问题，引起不必要的界面渲染

Embedded Data Model:
 
* 优点: 渲染数据的效率较高
* 缺点: 数据是嵌套的，存在较大的数据冗余，为了保证数据的一致性，需要复杂(有时可能是低效)的数据更新逻辑。例如，当userId_1的avatar发生改变时，在Normalized Data Model结构中，只需要根据对应的id在userById中找到UserRecord，然后更新其avatar值即可。而在Embedded Data Model中，需要分别遍历feedList和userList，找到对应的UserRecord，然后进行更新操作。

更多关于Normalized Data Model与Embedded Data Model的讨论，请参考[Data Model Design](https://docs.mongodb.com/manual/core/data-model-design/)。Git上一些相关这两种模型的讨论与资料(react，redux体系下):

- [How to handle case where the store "model" differs from the "display" model ](https://github.com/este/este/issues/519)   
- [Memoizing Hierarchial Selectors](https://github.com/reactjs/reselect/issues/47)   
- [Performance Issue with Hierarchical Structure](https://github.com/reactjs/redux/issues/764)   
- [TreeView.js](https://gist.github.com/ronag/bc8b9a33da172520e123)   
- [normalizr](https://github.com/paularmstrong/normalizr)

#### 3. reselect内部机制分析

*注意：本节中关于reselect的源码片段均来自reselect@3.0.1，后续部分将不再强调*

reselect源码较少，其中最重要的是createSelectorCreator和defaultMemoize这两个函数：

```javascript
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null
  let lastResult = null
  // position 1
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      // position 4
      lastResult = func.apply(null, arguments)
    }

    lastArgs = arguments
    return lastResult
  }
}

export function createSelectorCreator(memoize, ...memoizeOptions) {
  return (...funcs) => {
    let recomputations = 0
    const resultFunc = funcs.pop()
    const dependencies = getDependencies(funcs)
    
    const memoizedResultFunc = memoize(
      // position 3
      function () {
        recomputations++
        return resultFunc.apply(null, arguments)
      },
      ...memoizeOptions
    )

    const selector = defaultMemoize(
      // position 2
      function () {
        const params = []
        const length = dependencies.length

        for (let i = 0; i < length; i++) {
          params.push(dependencies[i].apply(null, arguments))
        }
        // position 5
        return memoizedResultFunc.apply(null, params)
      }
    )

    selector.resultFunc = resultFunc
    selector.recomputations = () => recomputations
    selector.resetRecomputations = () => recomputations = 0
    return selector
  }
}

export const createSelector = createSelectorCreator(defaultMemoize)
```

下面结合上文中提到的简单feed列表的例子来分析reselect相关源码。

首先我们利用reselect提供的createSelector方法创建了一个feedListSelector：

```
const feedListSelector = createSelector(getFeedIds, getFeedById, feedListSelectorCb)
```

此时createSelectorCreator源码中的dependencies等于[ getFeedIds, getFeedById ], resultFunc等于feedListSelectorCb。feedListSelector等于defaultMemoize返回的内部函数，即position 1处的内部函数。

在mapStateToProps中计算feedList：

```
let feedList = feedListSelector(state, idListKey)
```

我们调用feedListSelector并传递了两个参数state和idListKey，上面我们说到feedListSelector指向的是position 1处的内部函数，在该函数内部，首先判断当前传入的参数是否与上次传入的参数相同。当state和idListKey均没有发生改变时，返回上次计算的结果；否则执行position 4处的代码，重新计算feedList。

position 4处的func指向的是position 2处的内部函数，又因为```func.apply(null, arguments)```，我们传递给feedListSelector的参数全部传给了position 2处的内部函数。在position 2处函数的内部，首先计算出相关依赖数据，即依次执行getFeedIds和getFeedById。由```dependencies[i].apply(null, arguments)```可知，传递给feedListSelector参数全部传给了getFeedIds和getFeedById。

在position 5处，将所有的相关依赖数据作为参数依次传递给了memoizedResultFunc。由于createSelector使用的是defaultMemoize，故memoizedResultFunc也指向position 1处的内部函数。在position 1处函数的内部，首先会判断相关依赖数据是否改变，如果相关依赖数据不变，直接返回缓存的结果。当相关依赖数据改变时，会执行position 4处的代码，此时position 4处的func指向的是resultFunc即feedListSelectorCb。

##### (1) reselect如何访问React Props？

由上面的分析可知，所有传递给feedListSelector的参数，都会按顺序、依次传递给getFeedById和getFeedIds。因此我们可以这样传递React Props:

```
const mapStateToProps = (state, ownProps) => {
  let feedList = feedListSelector(state, ownProps)
  return {
    feedList: feedList,
    // other data
  }
}
```

##### (2) 由createSelector创建出来的selector的默认缓存大小是1

由上述defaultMemoize的实现可知，它只是利用了JS的闭包，用自由变量([free variable](http://dmitrysoshnikov.com/ecmascript/javascript-the-core/#scope-chain))分别记住了上次传入的参数和计算的结果，由于一个变量只能存储一个值，故其缓存大小为1。这可能会引起一些问题，看下面的例子：

```javascript
let feedList_1 = feedListSelector(state, 'feedIdList_1')
let feedList_2 = feedListSelector(state, 'feedIdList_2')
let feedList_3 = feedListSelector(state, 'feedIdList_1')
assert(feedList_1 !== feedList_3)
```

上述例子中，我们多次调用了feedListSelector并传入不同的feedIdListKey，由```feedList_1 !== feedList_3```可知，feedListSelector没有达到预期的缓存效果。

当第一次调用feedListSelector计算出feedList\_1，defaultMemoize中```lastArgs = [ state, 'feedIdList_1' ]```，当计算feedList\_2时，```arguments = [ state, 'feedIdList_2' ]```，通过shallowEqual比较发现lastArgs不等于arguments，并调用回调函数计算feedList_2。在第二次调用完feedListSelector后，defaultMemoize中```lastArgs = [ state, 'feedIdList_2' ]```。第三次调用feedListSelector计算feedList\_3，虽然参数与第一次调用时相同，但因为lastArgs不等于当前的arguments，会重新计算feedList，故```feedList_1 !== feedList_3```。

更多相关讨论，请查看reselect文档：[Sharing Selectors with Props Across Multiple Component Instances](https://github.com/reactjs/reselect#sharing-selectors-with-props-across-multiple-component-instances)

#### 4. reselect与Normalized Data Model

在上文中我们提到，当store中采用Normalized Data Model存储数据时，使用reselect会出现'因store中无关数据的改变而引起不必要的界面渲染'的问题，尤其在复杂的feed列表例子中，这个问题变得尤为凸出。

在简单feed列表例子中，产生这个问题的主要原因是feedListSelector依赖了feedById，而feedById中存在与feedList\_1无关的数据。既然feedList\_1相关的数据只有feedIdList\_1以及id\_1，id\_2和id\_3对应的FeedRecord。我们能不能让feedListSelector只依赖这些数据呢？如下面这段伪代码：

```javascript
const getFeedIds = (state, idListKey) => state['feed'].get(idListKey)
const createGetFeed = (feedId) => (state) => state['feed'].getIn(['feedById', feedId])

let feedListSelector = createSelector(
  createGetFeed(id_1),
  createGetFeed(id_2),
  createGetFeed(id_3),
  (feed_1, feed_2, feed_3) => [feed_1, feed_2, feed_3]
)
```

答案是不能。使用reselect时，我们需要使用createSelector来创建selector，并在创建selector时确定该selector的相关依赖数据。而在创建feedListSelector时，我们无法知道将要渲染的feedList中含有哪些feed，故上述想法是不可行的。这是reselect与Normalized Data Model一起使用时的一个局限性，下面将介绍reselect其他的局限性。

##### (1) 缓存每个feed的计算结果

上文中在我们的复杂feed列表例子中，feedListSelectorCb实现如下：

```javascript
const feedListSelectorCb = (feedIds, feedMap, userMap, organMap) => {
  feedIds = feedIds.toJS ? feedIds.toJS() : feedIds
  let feedList = feedIds.map((feedId) => {
    let feed = feedMap.get(feedId)
    let creator = userMap.get(feed.creator)
    let organization = organMap.get(creator.organization)
    
    feed = feed.set('creator', creator)
    feed = feed.setIn(['creator', 'organization'], organization)
    return feed
  })
}
```

上述代码中，计算feed的过程较复杂，希望将计算feed的结果缓存下来，看下面的代码(这里我们将整个列表connect到store上)：

```javascript
const getFeedById = state => state['feed'].get('feedById')
const getUserById = state => state['user'].get('userById')
const getOrganById = state => state['organization'].get('organById')
const getFeedIds = (state, idListKey) => state['feed'].get(idListKey)
const getFeed = (state, feedId) => state['feed'].getIn(['feedById', feedId])

const feedSelectorCb = (feed, userMap, organMap) => {
  let creator = userMap.get(feed.creator)
  let organization = organMap.get(creator.organization)
    
  feed = feed.set('creator', creator)
  feed = feed.setIn(['creator', 'organization'], organization)
  return feed
}

const feedSelector = createSelector(
  getFeed, getUserById, getOrganById, feedSelectorCb
)

const feedListSelectorCb = (feedIds, feedMap, userMap, organMap) => {
  feedIds = feedIds.toJS ? feedIds.toJS() : feedIds
  let feedList = feedIds.map((feedId) => {
    return feedSelector(state, feedId)
  })
}

const feedListSelector = createSelector(
  getFeedIds, getFeedById, getUserById, getOrganById, feedListSelectorCb
)
```

上述代码，虽然在feedListSelectorCb不需要使用feedMap, userMap和organMap，但是feedListSelector必须依赖相关数据，否则，例如当feedId\_1对应的的FeedRecord改变时，feedList\_1不会重新渲染。

同时我们还注意到，在feedListSelectorCb中调用feedSelector时，feedSelector拿不到state对象。回忆一下reselect源码，在调用feedListSelectorCb，只是将调用getFeedIds, getFeedById, getUserById和getOrganById得到的结果传递给了feedListSelectorCb，feedListSelectorCb参数表中没有state对象(虽然可以通过全局变量、或其他途径来获取state，但这似乎违背了reselect的初衷，故不推荐使用)。

在feedListSelectorCb中，会多次调用feedSelector，每次传入不同的feedId。由于reselect默认缓存大小为1的问题，每次调用feedSelector时，都会去重新计算，没有达到我们期望的缓存效果。

##### (2) 对feedList中的每个feed单独做connect

在上一小节的例子中，我们将feedList connect到store上，并在其mapStateToProps中计算出渲染整个列表需要所有数据。当我们需要缓存每个feed的计算结果时，上一小节中的方法似乎不太可行(欢迎吐槽)。

如果我们对每个feed单独做connect呢？修改后的代码：

```javascript
// feedList.js
const getFeedIds = (state, idListKey) => state['feed'].get(idListKey)

const feedIdListSelector = createSelector(getFeedIds, (feedIds) => {
  return feedIds.toJS ? feedIds.toJS() : feedIds
})

const mapStateToProps = (state, ownProps) => {
  let idListKey = ownProps.idListKey || 'feedIdList_1'
  return feedIdListSelector(state, idListKey)
}

// feedCell.js
const getUserById = state => state['user'].get('userById')
const getOrganById = state => state['organization'].get('organById')
const getFeed = (state, feedId) => state['feed'].getIn(['feedById', feedId])

const feedSelectorCb = (feed, userMap, organMap) => {
  let creator = userMap.get(feed.creator)
  let organization = organMap.get(creator.organization)
    
  feed = feed.set('creator', creator)
  feed = feed.setIn(['creator', 'organization'], organization)
  return feed
}

const mapStateToProps = (state, ownProps) => {
  const feedSelector = createSelector(
    getFeed, getUserById, getOrganById, calculateFeedCb
  )

  return (state, ownProps) => {
    return feedSelector(state, ownProps.feedId)
  }
}
```

*注意：在feedCell.js中，mapStateToProps的这种用法，主要是为了解决reselect默认缓存大小为1的问题，详见[Sharing Selectors with Props Across Multiple Component Instances](https://github.com/reactjs/reselect#sharing-selectors-with-props-across-multiple-component-instances)。在下一章per-component memoization中，也会着重分析它。*

上述代码达到了我们缓存每个feed计算结果的目的。但因为feedSelector需要依赖userById和organById这两部分数据，依然存在'因store中无关数据的改变而引起不必要的界面渲染'的问题。此外由于对每个feed都做了connect，这会导致整个应用中connect过多的问题，这会导致一些新的问题。更多关如在什么地方做connect讨论，请看[At what nesting level should components read entities from Stores in Flux?
](https://stackoverflow.com/questions/25701168/at-what-nesting-level-should-components-read-entities-from-stores-in-flux)。

##### (3) reselect与Normalized Data Model

reselect的一个特点是，它需要在创建selector时就确定相关依赖数据，并通过判断这些相关依赖数据是否改变来决定是否需要重新计算selector的结果。而Normalized Data Model一个特点是将存储数据集中存储(如feedById)，然后通过id来引用这些数据。当reselect与Normalized Data Model一起使用时，很难避免'因store中无关数据的改变而引起不必要的界面渲染'的问题。如上文中提到的：无论是对整个feedList做connect还是针对每个feedCell分别做connect，都无法摆脱对userById和organById的依赖。

我个人认为，reselect不太适合与Normalized Data Model一起使用(除非你能忍受因userById或organById中无关数据的改变而导致feed列表的重新渲染)。那么在使用Normalized Data Model的前提下，我们该如何来缓存selector的结果呢？

##### (4) 自定义memoizeSelector

reselect是memoizeSelector的一个具体例子，它主要做了两件事：

1. 判断相关依赖数据是否改变
2. 相关依赖数据不变，读取缓存；相关依赖数据改变，重新计算相关结果

因为reselect将判断相关依赖数据是否改变的逻辑封装在其内部，在面对复杂问题时，缺乏了相应的灵活性。我们可以根据实际需求，自定义这个过程，创造自己的memoizeSelector。在第一章第7节的第4小节“将缓存的思想应用到mapStateToProps中”中，我们借助lruMap自定义了一个memoizeSelector。mapStateToProps部分代码如下：

```javascript
const mapStateToProps = (state, ownProps) => {
  const hash = 'mapStateToProps'
  let feedIdList = memoizeFeedIdListByKey(state, lruMap, 'feedIdList_1')
  let hasChanged = feedIdList.hasChanged
  if (!hasChanged) {
    hasChanged = feedIdList.result.some((feedId) => {
      return memoizeFeedById(state, lruMap, feed).hasChanged
    })
  }
  
  if (!hasChanged && lruMap.has(hash)) {
    return lruMap.get(hash)
  }
  
  let feedIds = feedIdList.result.toJS()
  let feedList = feedIds((feedId) => {
    return memoizeFeedById(state, lruMap, feed).result
  })
  // do some other time consuming calculations here
  let result = {
    feedList: feedList,
    // other data
  }
  lruMap.set(hash, result)
  return result
}
```

与reselect不同的是，reselect是在定义是确定相关依赖数据，然后在执行过程判断相关依赖数据是否改变。而在这个实现中，我们是在selector执行过程中确定其相关依赖数据并同时判断相关依赖数据是否改变的，具有更大的灵活性，也不会存在'因store中无关数据的改变而引起不必要的界面渲染'的问题。

在这种实现中，我们无法避免对相关依赖数据的完全遍历，但相比于界面的重新渲染，这个遍历过程几乎可以忽略不计。

## 三、Per-Component Memoization

#### 1. 这个概念是如何产生的

per-component memoization是react-redux在4.3.0版本中引入的一个新概念。在4.3.0版本之前，当使用reselect在界面上渲染多个相似控件时，例如在界面上同时渲染feedList\_1和feedlist\_2，feedList\_1和feedlist\_2的唯一区别是ownProps中的idListKey不同。上文中有讨论到，reselect的默认缓存大小为1，同时渲染两个feedList会导致feedListSelector不能有效缓存。针对这个问题，git上有许多的讨论，列举一些我发现的相关讨论与资料：[reselect/issues/66](https://github.com/reactjs/reselect/issues/66)，[react-redux/pull/179](https://github.com/reactjs/react-redux/pull/179)，[react-redux/pull/180](https://github.com/reactjs/react-redux/pull/180)，[react-redux/pull/182](https://github.com/reactjs/react-redux/pull/182)，[react-redux/pull/183](https://github.com/reactjs/react-redux/pull/183)，[react-redux/pull/185](https://github.com/reactjs/react-redux/pull/185)，[react-redux/pull/279](https://github.com/reactjs/react-redux/pull/279)，[reselect/issues/79](https://github.com/reactjs/reselect/issues/79)，[reselect#accessing-react-props-in-selectors](https://github.com/reactjs/reselect#accessing-react-props-in-selectors)

对于这些讨论的结果，可简单总结如下：

* 在之前，ComponentClass拥有一个selectorInstance的引用，所以每个componentInstance共享同一个selectorInstance
* 我们想要的，ComponentClass拥有一个对SelectorClass的引用，所以每个componentInstance可以拥有自己的selectorInstance

我们可以通过以下两个途径达到这个目的：

* selectorInstance可以辨认出所依赖的componentInstance，根据相关讨论，这个方法似乎行不通
* 每个componentInstance创建自己的selectorInstance

*注意：这里只是对讨论中已有总结的翻译，查看该总结的来源请点击[这里](https://github.com/reactjs/react-redux/pull/183#issuecomment-165335094)*

#### 2. mapStateToProps

我们知道mapStateToProps可以返回一个object，即计算出的stateProps。但是在一些高级场景中，如果你想对渲染的性能做更多的控制，mapStateToProps可以返回一个函数，这时这个被返回的函数会被作为正常使用场景下的mapStateToProps传递给特定component实例。这样每个component实例都有自己的mapStateToProps，而不再是共用同一个mapStateToProps。在第二章第4节的第2小节："对feedList中的每个feed单独做connect"中，我们就利用了这个特性。

```javascript
const mapStateToProps = (state, ownProps) => {
  const feedSelector = createSelector(
    getFeed, getUserById, getOrganById, calculateFeedCb
  )

  return (state, ownProps) => {
    return feedSelector(state, ownProps.feedId)
  }
}
```

这样，每个feedCell都只属于有自己的mapStateToProps和feedSelector实例，这样就不会由因reselect默认缓存大小为1而引起的不缓存问题了。

#### 3. 修改上文中lruMap例子的实现

在第一章第7节的第4小节的lruMap例子中，我们创建了一个全局的lruMap，并设置key的个数为500。共用一个全局的lruMap难免会出现这样或那样的问题，现在我们可以创建专属于某个component实例的lruMap，也就是所谓的per-component memoization。代码如下：

```javascript
const mapStateToProps = (state, ownProps) => {
  const LRUMap = require('lru_map').LRUMap
  const lruMap = new LRUMap(500)

  return (state, ownProps) => {
    const hash = 'mapStateToProps'
    let feedIdList = memoizeFeedIdListByKey(state, lruMap, 'feedIdList_1')
    let hasChanged = feedIdList.hasChanged
    if (!hasChanged) {
      hasChanged = feedIdList.result.some((feedId) => {
        return memoizeFeedById(state, lruMap, feed).hasChanged
      })
    }
    ...
  }
}
```

#### 4. 缓存某个component实例的接口数据

对于移动端App，大多都会选择在本地缓存某些特定页面的数据，这样即使在离线状态下，再次打开App时，也能够向用户展示一些数据。同时还可以对用户隐藏一些耗时接口的加载过程。使用redux时，可以使用redux-persist这个库将store中的数据缓存到本地存储，每次打开App时，会去读取这些本地存储，并将数据恢复到store中。

使用redux-persist可能会出现下面这几个问题：

* redux-persist一般会在App启动时通过rehydrate事件来恢复store数据，但如果store过大，恢复会比较耗时，这会影响到App的启动速度；
* Android端SQLite中的每条记录有最大为2MB的限制，某个reducer key对应的store数据过大时，可能会报Couldn't read row 0, col 0 from CursorWindow错误，详见[这里](https://github.com/facebook/react-native/issues/12529)和[这里](https://stackoverflow.com/questions/21432556/android-java-lang-illegalstateexception-couldnt-read-row-0-col-0-from-cursorw/21432966#21432966)；
* 虽然可以通过whitelist和transforms来控制缓存的数据的大小，但实现针对特定页面数据的transform会比较困难；

我们可以利用per-component memoization这一特性，来缓存某些特定页面的接口或store数据。同时利用redux-persist来存储诸如登陆状态、配置信息等数据量较少的全局数据。

下面的伪代码以feed详情为例，展示如何利用per-component memoization特性来缓存页面的接口数据，其中ownProps中含有feedId：

```javascript
import { AsyncStorage } from 'react-native'
import { fetchFeedDetail } from 'path-to-action-folder'
import { dispatchFeedDetailApiResult } from 'some-where-else'

const mapStateToProps = (state, ownProps) => {
  const hash = `feedDetail:${ownProps.feedId}`
  let apiResult = AsyncStorage.getItem(hash)
  if (apiResult) {
    dispatchFeedDetailApiResult({ apiResult: apiResult })
  }
  
  return (state, ownProps) => {
    return { ... }
  }
}

const mapDispatchToProps = (dispatch, ownProps) => {
  return {
    fetchDatas: (payload) => {
      return dispatch(fetchDetailWrapper(ownProps, payload))
    },
    ...
  }
}

function fetchDetailWrapper (ownProps, payload) {
  return (dispatch, getState) => {
    let fetchParams = { ... }
    return dispatch(fetchFeedDetail(fetchParams)).then((result) => {
      const hash = `feedDetail:${ownProps.feedId}`
      AsyncStorage.setItem(hash, JSON.stringify(result))
      return result
    })
  }
}

export connect(mapStateToProps, mapDispatchToProps)
```

在mapStateToProps中，先检测本地是否有接口数据，如果有，则通过dispatchFeedDetailApiResult处理该数据并以某种途径存储到store中；在component实例中调用fetchDatas获取文章详情数据时，会将获取到的接口数据缓存在本地。相对于redux-persist，这种实现是在进入某个页面而不是启动App时恢复相关数据，且可以针对特定页面做相关的接口数据的缓存。

## 四、总结

接触react/redux有一段时间了，也利用它们做了一些项目。本文主要是对在实际项目中遇到的问题的一些思考，并给出了一些可能的解决方案。同时本文还涉及到了一些关于在什么地方使用connect，在store采用什么样的数据模型等基本问题的讨论。希望能给大家一些帮助。

本文是我的第一篇技术分享类文章，希望大家能喜欢。出于个人经验和能力的局限性，对于本文中可能存在的问题或错误的观点，还请各位读者大神帮忙指正。

最后，欢迎大家访问我们基于react/redux搭建的网站：[12km.com](https://www.12km.com/)。

</font>