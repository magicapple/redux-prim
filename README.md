# redux-prim
redux-prim 弱化了 action 和 reducer ，加强了 action creator。帮助在 react + redux 更好的进行数据契约设计实现代码复用。即使项目里面不需要抽象和复用，使用 redux-prim 也能简化 action 和 reducer，在某些场景非常方便。

## 安装

```shell
npm i redux-prim 
```
## 简单例子
``` javascript
import { createPrimActions, createPrimReducer } from 'redux-prim';

var todoActions = createPrimActions('todo', ({ setState }) => {
	return {
		setTodoVisibility(todoVisible) {
			return setState({ todoVisible });
		}
	}
} );

combineReducer({
	todo: createPrimReducer('todo', function getDefaultState() {
		// return default state
	})
})

```

你可能会奇怪不需要 middleware 配置，甚至不需要定义 action 甚至 reducer。这正是 redux-prim 希望强调的，开发者可以更加聚焦到数据的意义和变化，而不是用考虑 action，reducer 已经数据不变性。

## Namespace
例子中 `createPrimActions` 和 `createPrimReducer` 的第一个参数必填参数 **todo** 是命名空间，同一命名空间的 actions 和 reducers 是配套的，它是 redux-prim 实现其它特性的基础。

## Updater
Updater 是 redux-prim 最重要的特性，他本意是用来表示对数据操作的一层抽象，这些函数与业务无关，方便我们快速在这个抽象基础上实现业务逻辑。比如上述例子的 setState 就是一个内置的 updater，你会发现它生成一个符合 SFA 的 action，并由对应的 reducer 函数处理。

``` javascript
{
    type: '@prim/todo/setState?todoVisible=true'
    payload: { todoVisible: true }
}
```
这个 action 会被对应的内置 reducer 捕获并处理，总共有三个数据操作函数：

- initState(state), 对应处理函数为：
``` javascript
    return Object.assign({}, getDefaultState(), state);
```

- setState(changes), 对应处理函数为：
``` javascript
    return Object.assign({}, state, changes);
```

- mergeState(changes), equal to
``` javascript
return Object.keys(changes).reduce(function (s, key) {
	if (Object.prototype.toString.call(s[key]) === "[object Object]" &&
		Object.prototype.toString.call(changes[key]) === "[object Object]") {
		s[key] = Object.assign({}, s[key], changes[key]);
	} else {
		s[key] = changes;
	}
	return s;
}, Object.assign({}, state));
```

## extendUpdaters 
我们非常谨慎的提供 Updater 及其实现，并允许用户在项目里自定义 `updater`：

``` javascript
import { extendUpdaters } = from 'reduxe-prim';
extendUpdaters({
	pushArray({ state, action, /*getDefaultState*/ }) {
		var { name, value } = action.payload // for simplicity
		return {...state, [name]: [...state[name], value] }
	}
})
```
这样我们就可以有了 4 个 `updater`，`initState`, `setState`, `mergeState`, `pushArray`.

``` javascript
import { createPrimActions, createPrimReducer } from 'redux-prim';
var todoActions = createPrimActions('todo', ({ pushArray }) => {
    return {
        addTodo(todo) {
            return pushArray({ name: 'todoList', value: todo });
        }
    }
} );
combineReducer({
    todo: createPrimReducer('todo', function getDefaultState() {
        return {
			todoList: []
		}
    })
})
```

如果我们打印这个 action 会看到如下类似：
``` javascript
{
    type: '@prim/todo/pushArray?name=[String]&value=[Object]
    payload: { name: 'todoList', value: todo }
}
```
刚才提到，Updater 本身是数据操作的抽象，我们甚至可以覆盖内置 `updater` 并基于 Immutable.js 重写。
``` javascript
import { extendUpdaters } = from 'reduxe-prim';
extendUpdaters({
	setState({ state, action }) { /* use immutable.js */ },
	mergeState({ state, action }) { /* use immutable.js */ }
	//...
})
```

## action 和 reducer
有些和业务相关的复杂数据操作，不适合用 `extendUpdaters` 实心。 我们仍然可以定义 action type 并实现 reducer，参考如下写法：

``` javascript
import { createPrimActions, createPrimReducer } from 'redux-prim';

var todoActions = createPrimActions('todo', ({ primAction /*, setState*/ }) => {
	return {
		complexAction(data) {
		    // 用 primAction 包裹，才能在下面的 reducer 里面捕获
			return primAction({
				type: 'complex-action',
				payload: data
			})
		}
	}
} );

combineReducer({
	todo: createPrimReducer('todo', getDefaultState, function reducer(state, action) {
		if(action.type === 'complex-action') {
			// return a new state
		}
		return state;
	})
})
```
可以发现 action 签名如下
``` javascript
{
    type: '@prim/todo/complex-action
    payload: data
}
```

## 使用middleware
redux-prim 与大部分 redux-middleware 并没有冲突，action 的创建也符合 SFA 规范。假如我们配置了 redux-thunk 中间件，可以正常在 createPrimAction 里面使用：
``` javascript
var todoActions = createPrimActions('todo', ({ initState }) => ({
	loadPage(todo) {
		return async (dispatch, getState) => {
		    var pageState = await loadPageState();
			dispatch(initState(pageState));
		}
	}
}));
```

# 数据契约设计
redux-prim 最开始是 redux 架构下一套最佳实践的工具，这套最佳实践最后演变成为数据契约设计，类似 bertrond myer 在 OOP 下的契约式设计，数据契约设计可以认为是在 FP 下实现抽象设计的方法。

## 何时使用
在一些大型的云系统、CMS、ERP 等，我们会遇到大量相似的场景，比如表格或者表单页。需要把这些页面的抽象并复用代码实现，同时遵循 react +  redux 的模式。



## 解决了什么问题
redux 是一个通过 action 和 reducer 管理 state 的类库，它规定 state 是不可变的单一对象树，state 的修改必须通过 dispatch 一个 action 来声明意图，并通过纯函数 reducer 返回新的 state 来达到修改的目的。这样可以让应用更可预测，同时更好支持热重载，时间旅行和服务端渲染等功能。但是带来的问题也很明显：

- **过多的模式代码**

在一个拥有几十上百个列表增删改查页的应用中，假设一个页面需要10+ actions，1个reducer 以及若干的容器组件，我们需要实现维护数千个甚至更多的 actions 和 actionCreator，数百个 reducer，以及更多的容器组件（容器组件通常会细分已获得更高的性能）。使用 redux-prim 这样的页面不管多少，只需要与一个页面差不多甚至更少的 actions，reducer 和容器组件就能完成。大大增加的开发速度和维护成本。

## 代码实现步骤
一个比较典型的页面如下（截图自 antd-pro）：
![](http://p406.qhimgs4.com/t01958b0e04e22a2fb1.png)
页面比如用户，订单，地址等大致类似,我们暂定这种场景叫 page。要实现一个 page 数据契约，首先：

### 抽象 state 的共性
在系统中我们有许多类似的页面，比如 todo， user，admin，order 等。一般在redux 我们会把页面的数据划分到不同的域（data domains）。比如：
``` JavaScript
  const app = combineReducers({
     todo: todoReducer,
	 user: userReducer,
	 admin: adminReducer,
	 order: orderReducer
  })
```
这种方式会导致功能一致的 container 组件在不同页面的重复定义，因为 connect 的数据域不一致。所以我们制定一个契约（契约强调如果不实现责任，就获取没有利益），所有类似的页面使用同一个数据域 page，并且同一时间这个数据域只有一个场景在使用：

### createContractReducer

``` JavaScript
	import { createContractReducer } from 'reduc-prim';
	const app = combineReducers({
		page:  createContractReducer('page', getDefaultState)
	})
```
我们继续制定契约，约定没个页面都有
	- list：数组，根据不同页面，可以是任意的对象。
	- criteria：一个代表 `key` `value` 的搜索条件，不同页面可以随意设置这个对象
	- pagination：分页, 包含 `page`, `pageSize` 和 `total`。


``` JavaScript
function getDefaultState() {
	return  {
		// an array of domain object
		list: [], 
		// key value pairs use to do build api query
		criteria: {},
		pagination: { page, pageSize, total }
	}
}
```
因为有了契约，没个功能对应的数据的名称，结构以及所在数据域都被固定下来，代码抽象变得更加容易和彻底。

由于 action 和 reducer 以及被 redux-prim 弱化，我们接下来抽象契约数据的行为，对应的事 action creator：
 
### createContractActions

``` JavaScript
import { createContractActions } from 'redux-prim';
const pageContractActions =
  createContractActions('page', ({ primAction, initState, setState, mergeState }, { getListApi }) => {
    var actions = {
      initState(state) {
	    // 页面组件在 componentDidMount 必须调用 initState，除了提供各自
		// 的初始状态，更重要是给 page 数据域施加一个排它锁。
        return initState(state);
      }
      setCriteria(criteria) {
        return mergeState({ criteria })
      },
      resetCriteria() {
        return setState({ criteria: {} });
      },
      getList() {
        return (dispatch, getState) => {
          const { criteria, pagination } = getState().page;
          return getListApi(criteria, pagination).then(data => {
            dispatch(setState({data}));
          });
        }
      },
      changePagination(page, pageSize) {
        return dispatch => {
          dispatch(setState({ page, pageSize }));
          dispatch(actions.getList());
        }
      }
    };
    return actions;
  }
```
因为每一个页面获取列表数据的接口都不一样，所以 pageContractActions 是一个高阶函数，它接受两个参数，name 和 options。
``` javascript
// todo actions
var todoActions = pageContractActions ('todo', { getListApi: api.getTodoList });

// orderActions
var orderActions = pageContractActions ('order', { getListApi: api.getOrderList });

``` 

这里面我们对 getListApi 制定了契约，规定其函数签名为：
``` javascript
    function getListApi(
		criteria: Object, 
		pagination: {page: number, pageSize: number}
    ): Promise<{
		list: Array,
		page: number,
		pageSize: number
		}>
```
`问题：如果我们项目的后端接口不一致怎么办：
	可以用高阶函数做适配，否则就不要接入这套实现。
`
### 实现通用容器组件
现在我们可以针对契约数据 `list` 实现一个通用的表格组件：
``` JavaScript
connect(
	state=>({list: state.page.list})
)
class PageTable extends React.Component {
	static propsType = {
		howToRender: PropTypes.any.isRequired
	}
}
```
这个容器组件只是知道怎么获取数据，但是对数据怎么显示一无所知，所以需要在场景页面配置如何显示：
``` JavaScript
// User 页面
import PageTable from './PageTable';
import UserActions from './UserActions';
connect(
	()=>({}) // 只是为了 dispach
)
User extends Component {
	componentDidMount() {
		this.dispatch(UserActions.initState());
	}
	renderColumns = [
		{
			renderHeader = () => '名字',
			renderContent = item => item.name
		}, {
			renderHeader = () => '性别',
			renderContent = item => <Gender value={item.gender}/>
		}
	]
	render() {
		return <PageTable howToRender={this.renderColumns}/>
	}
}
```
## 最后
契约的概念的特点强调了设计是一种权衡，设计者应该牺牲哪些通用性，来获取哪些利益。

通常在大型项目中破坏抽象的元凶就是无节制的添加功能或者说防御式兼容。而数据契约设计保护核心数据的功能和意义保持不变，迫使开发者在别的层面解决问题，降低了系统的耦合性，同时保护了抽象和已经复用的代码。这一点我在几个项目的实践中反复的被验证，印象深刻。

同样的场景如果不用 redux，不用数据契约设计，是否也能达到类似的抽象效果？答案是肯定的，其实我就是用 OOP 的方式实现之后，探索如何在 FP 这一种范式里面达到类似的效果。要说最终的区别就是，OOP 的方式更加好理解，而 FP 的方式提供了更多的可预测性和组合性。这目前只是我的一种感觉，或许可以用数学来证明？

这里我们列出 Bertrand Meyer 在 OOP 契约式设计几个原则和数据契约设计对比，会发现有惊人相似：

-  Command–query separation (CQS)
	对比 redux 的 action 和 reducer。 

-  Uniform access principle (UAP) 
	里面提出派生数据的概念
	
-  Single Choice Principle (SCP)
	单一数据源，dry
	
-  Open/Closed Principle (OCP)
	开发封闭原则

虽然类似，但是编程范式都不一样，因为抽象从数据开始，所以我命名为`数据契约设计`，这是为了沟通的方便，如果被更多人认可，慢慢就会被接受和提出，目前先不要喷我造新词。

关于数据契约设计，我会在中国首届React开发者大会上进行分享，后续会有更加系统的文章发出。

最后在啰嗦几句，虽然目前这套理论还不完善，希望看到大家更多的参与进来，尤其是FP的大牛们，帮助这种设计理论的演化，哪怕是推翻并出现更好的方法论。














