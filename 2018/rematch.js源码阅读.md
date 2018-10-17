## 介绍
众所周知，redux是react全家桶最佳实践必不可少的一环，但是其模版代码过多，完成一个状态的闭环需要跨多个文件，再加上异步处理方案（如redux-saga）后，模版代码就更多了，开发起来特别痛苦，业内比较有名的解决方案是dva，基于redux-saga做异步处理，且尽量的减少了模版代码。

rematch是一个新的redux Framework[《重新思考redux》](https://hackernoon.com/redesigning-redux-b2baee8b8a38),[中文翻译版](https://segmentfault.com/a/1190000014843214).

下面是官方对rematch的介绍：

Rematch是没有boilerplate的Redux最佳实践。没有多余的action types，action creators，switch 语句或者thunks。

## 基本用法

```js
  //getStore.js
  import { init } from '@rematch/core'
  import { currentTheme } from './models/theme'
  import { currentType } from './models/type'
  import { todos } from './models/todos'
  const store = init({
    models: {
      todos,
      currentTheme,
      currentType
    }
  })
  export default store

  //index.js
  ReactDOM.render(
    <Provider store={store}>
      <App />
    </Provider>,
    document.getElementById('root')
  )

  //container.js
  
  import { dispatch } from '@rematch/core'
  dispatch.todos.addTodo(text)
  ....
  export default connect(mapState)(TodoList)

  //todos 相当于reducer

  export const  todos = {
    state: [],
    //异步action
    effects: {
      async addTodoAsync(payload) {
        await fetchBy()
      }
    },
    //同步action
    reducers: {
      addTodo(state, text) {
        return [newTodo, ...state]
      },
    }
  }
```
## 源码解析

### 工具类

```js
/**
* utils/deprecate.js
* @param {string}
* @return {void}
**/ 
export default (warning: string): void => {console.warn(warning)}


/**
* utils/isListener.js
* @param {string}
* @return {boolean}
* 判断这个action是否在其他model上是一个listener,判断是全局的还是私有的
**/
export default (reducer: string): boolean => reducer.includes('/')



/**
* utils/validate.js
* 如果错误，就抛出的函数，类似invariant
* @param {array[object]} validations 
* @return {void}
**/
export default (validations: Validation[]): void => {
	if (process.env.NODE_ENV !== 'production') {
		for (const validation of validations) {
			const condition = validation[0]
			const errorMessage = validation[1]
			if (condition) {
				throw new Error(errorMessage)
			}
		}
	}
}


/**
* utils/mergeConfig.js
* @param {object} validations 
* @return {object}
* 对init传入的参数进行校验及合并，校验规则这里去掉了
**/

(initConfig: R.InitConfig & { name: string }): R.Config => {
	const config: R.Config = {
		name: initConfig.name,
		models: {},
		plugins: [],
		...initConfig,
		redux: {
			reducers: {},
			rootReducers: {},
			enhancers: [],
			middlewares: [],
			...initConfig.redux,
			devtoolOptions: {
				name: initConfig.name,
				...(initConfig.redux && initConfig.redux.devtoolOptions
					? initConfig.redux.devtoolOptions
					: {}),
			},
		},
	}
	// defaults
	for (const plugin of config.plugins) {
		if (plugin.config) {
			// models
			const models: R.Models = merge(config.models, plugin.config.models)
			config.models = models

			// plugins
			config.plugins = [...config.plugins, ...(plugin.config.plugins || [])]

			// redux
			if (plugin.config.redux) {
				config.redux.initialState = merge(
					config.redux.initialState,
					plugin.config.redux.initialState
				)
				config.redux.reducers = merge(
					config.redux.reducers,
					plugin.config.redux.reducers
				)
				config.redux.rootReducers = merge(
					config.redux.rootReducers,
					plugin.config.redux.reducers
				)
				config.redux.enhancers = [
					...config.redux.enhancers,
					...(plugin.config.redux.enhancers || []),
				]
				config.redux.middlewares = [
					...config.redux.middlewares,
					...(plugin.config.redux.middlewares || []),
				]
				config.redux.combineReducers =
					config.redux.combineReducers || plugin.config.redux.combineReducers
				config.redux.createStore =
					config.redux.createStore || plugin.config.redux.createStore
			}
		}
	}
	return config
}

```

### 内置插件
#### dispatch.ts
用于同步方法reducer生成唯一的dispatch
```js
/**
 * Dispatch Plugin
 *
 * 结合pluginFactory，generates dispatch[modelName][actionName]
 */
const dispatchPlugin: R.Plugin = {
	exposed: {
    //定义storeDispatch、storeGetState来确定onStoreCreated是否按预期执行
    ....
		//就是redux的dispatch
		dispatch(action: R.Action) {
			return this.storeDispatch(action)
		},
    //自动生成dispatch的type,模块名+reducer名
		createDispatcher(modelName: string, reducerName: string) {
			return async (payload?: any, meta?: any): Promise<any> => {
				const action: R.Action = { type: `${modelName}/${reducerName}` }
				if (typeof payload !== 'undefined') {
					action.payload = payload
				}
				if (typeof meta !== 'undefined') {
					action.meta = meta
				}
				return this.dispatch(action)
			}
		},
	},

	// 在store生成后，把getState，dispatch赋值给this
	onStoreCreated(store: any) {
		this.storeDispatch = store.dispatch
		this.storeGetState = store.getState
		//返回修改redux store树上的dispatch，以至于来监听所有的dispatch
		return { dispatch: this.dispatch }
	},

	// 对所有的model的reducers生成一个 action creators
	onModel(model: R.Model) {
		this.dispatch[model.name] = {}
		if (!model.reducers) {
			return
		}
		for (const reducerName of Object.keys(model.reducers)) {
      //类型检查
      ...
      //把每个reducer的方法绑定到this.dispatch上，同时，自动生成唯一到type
			this.dispatch[model.name][reducerName] = this.createDispatcher.apply(
				this,
				[model.name, reducerName]
			)
		}
	},
}

export default dispatchPlugin
```

#### effects.ts
用于异步方法reducer生成唯一的dispatch
```js
const effectsPlugin: R.Plugin = {
	exposed: {
		// expose effects for access from dispatch plugin
		effects: {},
	},

	// 添加effects到dispatch，以至于dispatch[modelName][effectName]可以调用effect
	onModel(model: R.Model): void {
		if (!model.effects) {
			return
		}

		const effects =
			typeof model.effects === 'function'
				? model.effects(this.dispatch)
				: model.effects

		for (const effectName of Object.keys(effects)) {
      ...
      //类型校验
      ...
			this.effects[`${model.name}/${effectName}`] = effects[effectName].bind(
				this.dispatch[model.name]
			)
			//把每个effect的方法绑定到this.dispatch上，同时，自动生成唯一到type
      //this.createDispatcher方法这时已经绑定在this上了
      //插件的加载顺序 [dispatchPlugin, effectsPlugin,...others]
			this.dispatch[model.name][effectName] = this.createDispatcher.apply(
				this,
				[model.name, effectName]
			)
			// 标记effect，区别同步方法
			this.dispatch[model.name][effectName].isEffect = true
		}
	},

	// process async/await actions
  // redux的中间件
	middleware(store) {
		return next => async (action: R.Action) => {
			// async/await acts as promise middleware
			if (action.type in this.effects) {
				await next(action)
				return this.effects[action.type](
					action.payload,
					store.getState(),
					action.meta
				)
			}
			return next(action)
		}
	},
}

export default effectsPlugin
```

### 主流程
#### index.ts

```js

export {
  getState,// 提示全局的getState方法移除，请使用use store.getState
  dispatch,// 提示全局的dispatch方法移除，请使用use store.dispatch
  createModel,// ts检测类型，返回和参数一样的对象
}
// 基于配置生成一个rematch store，
export const init = (initConfig: R.InitConfig = {}): R.RematchStore => {
	const name = initConfig.name || count.toString()
	count += 1
	const config: R.Config = mergeConfig({ ...initConfig, name })
	return new Rematch(config).init()
}

export default {
	init
}

```
#### pluginFactory.ts

```js
export default (config: R.Config) => ({
	config,
  // 工具类中报错的函数，方便后面调用
	validate,
  // 绑定plugin的属性和方法到PluginFactorys的实例上
	create(plugin: R.Plugin): R.Plugin {

    //校验类型这里去掉

    //执行plugin对象的onInit方法
		if (plugin.onInit) {
			plugin.onInit.call(this)
		}

		const result: R.Plugin | any = {}

		if (plugin.exposed) {
			for (const key of Object.keys(plugin.exposed)) {
				this[key] =
					typeof plugin.exposed[key] === 'function'
						? plugin.exposed[key].bind(this) // 绑定函数到plugin class
						: Object.create(plugin.exposed[key]) // 添加属性到plugin class
			}
		}
    //绑定onModel，middleware，onStoreCreated的this指向
		for (const method of ['onModel', 'middleware', 'onStoreCreated']) {
			if (plugin[method]) {
				result[method] = plugin[method].bind(this)
			}
		}
		return result
	},
})

```
#### rematch.ts

```js
  // 默认的两个插件
  // dispatchPlugin -> 同步方法reducer生成唯一的dispatch
  //effectsPlugin   -> 异步方法effects生成唯一的dispatch
  const corePlugins: R.Plugin[] = [dispatchPlugin, effectsPlugin]
  export default class Rematch {
    protected config: R.Config
    protected models: R.Model[]
    private plugins: R.Plugin[] = []
    private pluginFactory: R.PluginFactory

    constructor(config: R.Config) {
      this.config = config
      //绑定一个含'onModel', 'middleware', 'onStoreCreated'的对象到this上
      this.pluginFactory = pluginFactory(config)
      for (const plugin of corePlugins.concat(this.config.plugins)) {
        this.plugins.push(this.pluginFactory.create(plugin))
      }
      // 把plugin里面的middleware放入this.config.redux.middlewares中
      this.forEachPlugin('middleware', (middleware) => {
        this.config.redux.middlewares.push(middleware)
      })
    }
    public forEachPlugin(method: string, fn: (content: any) => void) {
      for (const plugin of this.plugins) {
        if (plugin[method]) {
          fn(plugin[method])
        }
      }
    }
    public getModels(models: R.Models): R.Model[] {
      return Object.keys(models).map((name: string) => ({
        name,
        ...models[name],
        reducers: models[name].reducers || {},
      }))
    }
    public addModel(model: R.Model) {
      ...
      //类型检查...
      ...
      // 把对应的model当作参数，执行plugin的onModel方法，生成唯一的type的dispatch
      this.forEachPlugin('onModel', (onModel) => onModel(model))
    }
    public init() {
      // 保存所有的models
      this.models = this.getModels(this.config.models)
      for (const model of this.models) {
				// 执行所有的plugin里的onModel方法
        this.addModel(model)
      }
      // 基于initialState生成一个redux store树
      // 合并额外的reducers
      const redux = createRedux.call(this, {
        redux: this.config.redux,
        models: this.models,
      })

			//基于redux的store生成一个rematchStore
      const rematchStore = {
        name: this.config.name,
				//浅拷贝redux store里的方法属性
        ...redux.store,
				// 利用replaceReducer动态地加载models，类似dva
        model: (model: R.Model) => {
					//执行plugin的onModel方法
          this.addModel(model)
					//合并新的Reducers
          redux.mergeReducers(redux.createModelReducer(model))
          redux.store.replaceReducer(redux.createRootReducer(this.config.redux.rootReducers))
          redux.store.dispatch({ type: '@@redux/REPLACE '})
        },
      }

      this.forEachPlugin('onStoreCreated', (onStoreCreated) => {
        const returned = onStoreCreated(rematchStore)
        // 如果onStoreCreated返回了一个对象
        // 把对象合并到store树上，来覆盖初始化的属性和方法
        if (returned) {
          Object.keys(returned || {}).forEach((key) => {
            rematchStore[key] = returned[key]
          })
        }
      })
			//返回一个rematch的store树
      return rematchStore
    }
  }
```

#### redux.ts

```js

export default function({
	redux,
	models,
}: {
	redux: R.ConfigRedux,
	models: R.Model[],
}) {
	const combineReducers = redux.combineReducers || Redux.combineReducers
	const createStore: Redux.StoreCreator = redux.createStore || Redux.createStore
	const initialState: any =
		typeof redux.initialState !== 'undefined' ? redux.initialState : {}

	this.reducers = redux.reducers

	// 合并models生成reducers
	this.mergeReducers = (nextReducers: R.ModelReducers = {}) => {
		// 合并新的reducers和老的reducers
		this.reducers = { ...this.reducers, ...nextReducers }
		if (!Object.keys(this.reducers).length) {
			// 如果没有reducers,就直接返回state
			return (state: any) => state
		}
		return combineReducers(this.reducers)
	}

	this.createModelReducer = (model: R.Model) => {
		const modelBaseReducer = model.baseReducer
		const modelReducers = {}
		// 把model里面的所有的educers转化成action
		for (const modelReducer of Object.keys(model.reducers || {})) {
			const action = isListener(modelReducer)
				? modelReducer
				: `${model.name}/${modelReducer}`
			modelReducers[action] = model.reducers[modelReducer]
		}
		//合并reducer
		const combinedReducer = (state: any = model.state, action: R.Action) => {
			// 对异步的effects进行处理
			if (typeof modelReducers[action.type] === 'function') {
				return modelReducers[action.type](state, action.payload, action.meta)
			}
			return state
		}

		this.reducers[model.name] = !modelBaseReducer
			? combinedReducer
			: (state: any, action: R.Action) =>
					combinedReducer(modelBaseReducer(state, action), action)
	}
	// 初始化 model reducers
	for (const model of models) {
		this.createModelReducer(model)
	}
	// 生成一个当前根reducers
	this.createRootReducer = (
		rootReducers: R.RootReducers = {}
	): Redux.Reducer<any, R.Action> => {
		const mergedReducers: Redux.Reducer<any> = this.mergeReducers()
		if (Object.keys(rootReducers).length) {
			return (state, action) => {
				const rootReducerAction = rootReducers[action.type]
				if (rootReducers[action.type]) {
					return mergedReducers(rootReducerAction(state, action), action)
				}
				return mergedReducers(state, action)
			}
		}
		return mergedReducers
	}
	// redux的初始化：生成一个当前根reducers
	const rootReducer = this.createRootReducer(redux.rootReducers)

	// redux的初始化：生成一个当前根middlewares
	const middlewares = Redux.applyMiddleware(...redux.middlewares)


	// redux的初始化：生成一个当前根enhancers，默认加上Devtools
	const enhancers = composeEnhancersWithDevtools(redux.devtoolOptions)(
		...redux.enhancers,
		middlewares
	)

	//redux的初始化：生成一个redux树
	this.store = createStore(rootReducer, initialState, enhancers)

	return this
}

```

