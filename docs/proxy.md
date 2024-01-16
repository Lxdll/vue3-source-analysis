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

```javascript
ah
```