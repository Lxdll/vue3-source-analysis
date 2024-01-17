# 响应式原理

## 公共小函数
### isReadonly()
```javascript
/**
 * 判断对象身上有没有 __v_isReadonly 这个属性
 *  - 如果有这个属性，那么就是只读属性
 *  - 如果没有，就不是只读
 */
export function isReadonly(value: unknown): boolean {
  // ReactiveFlags.IS_READONLY = "__v_isReadonly"
  return !!(value && (value as Target)[ReactiveFlags.IS_READONLY])
}
```

### isObject()
```javascript
/**
 * 使用 typeof 判断是否是 object，且不是 null
 */
export const isObject = (val: unknown): val is Record<any, any> =>
  val !== null && typeof val === 'object'
```

## Ref

## Reactive
```javascript
/**
 * 如果是只读的属性，那么直接 return
 * 否则使用 createReactiveObject 函数创建响应式对象
 */
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (isReadonly(target)) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap,
  )
}
```
### createReactiveObject()
::: tip
1. 首先判断 `target` 是否是一个对象类型的（即使用 typeof 判断 `target` 的类型是否是 object，如果不是的话在 DEV 环境给出对应的警告）<br>
2. 判断当前 `target` 是否已经有对应的 proxy 了，即当前 `target` 已经是一个响应式对象了，如果有的话，直接返回<br>
3. 使用 `Object.prototype.toString.call(target)` 拿到 targetType<br>
\- 如果 `target` 被标记了 SKIP（即存在 ReactiveFlags.SKIP 属性 - [ReactiveFlags.SKIP = "__v_skip"]）<br>
\- 或者 `!Object.isExtensible(target)` 即当前的 `target` 不可以扩展<br>
4. 不同的 targetType 使用不同的 handlers<br>
\- `Object` 和 `Array` 使用 baseHandlers<br>
\- `Map`、`Set`、`WeakMap`、`WeakSet` 使用 collectionHandlers<br>
:::

::: details 源码实现
```javascript
function createReactiveObject(
  // 需要 proxy 的 target
  target: Target,
  // 是否是只读的
  isReadonly: boolean,
  // 对象(Object)和数组(Array)的处理器
  baseHandlers: ProxyHandler<any>,
  // 集合类型的处理器
  // Map
  // Set
  // WeakMap
  // WeakSet
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>,
) {
  /**
   * 使用 typeof 判断 target 是否为 object
   * 不是的话，在开发环境要给出警告信息
   */
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }

  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  /**
   * 判断当前的 target 是否存在对应的 proxy
   * 如果已经存在了，直接返回
   */
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }

  /**
   * 获取当前 target 的类型
   *  - 使用 Object.prototype.toString.call(target)
   *  - 类型是截取 "[object targetType]" 中的 targetType
   */
  const targetType = getTargetType(target)
  /**
   * 如果 target 上标记了 SKIP，即存在 ReactiveFlags.SKIP 属性 - [ReactiveFlags.SKIP = "__v_skip"]
   * 或者 !Object.isExtensible(target) 即当前的 target 不可以扩展
   * 
   * 那么 targetType 就是 INVALID 的
   */
  if (targetType === TargetType.INVALID) {
    return target
  }

  /**
   * 对于不同的 targetType 做不同的 handler
   * Object 和 Array 类型使用 baseHandlers
   * Map、Set、WeakMap 和 WeakSet 类型使用 collectionHandlers
   */
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers,
  )
  // 缓存一份当前 target 和 proxy 的 map
  proxyMap.set(target, proxy)
  return proxy
}
```
:::

### baseHandlers
#### get 拦截函数
如果读取的是数组的下列方法，那么返回 Vue 框架重写的方法 <br>
1. `includes`、`indexOf`、`lastIndexOf` <br>
`TODO`: Vue 的自实现逻辑
2. `push`、`pop`、`shift`、`unshift`、`splice` <br>
`TODO`: Vue 的自实现逻辑

如果是读取的 hasOwnProperty 属性，也是返回 Vue 框架重写的方法
### collectionHandlers