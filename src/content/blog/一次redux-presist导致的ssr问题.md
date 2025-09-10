---
title: 一次redux presist导致的ssr问题
date: 2024-07-18 17:19:46
tags: [前端, ssr, nextjs]
headimg: https://lain.astrack.me/img/blog/8e526a14-8135-4d62-8b5c-e49b0b33e435.png
---

## 问题
在使用`redux presist`的时候，如果这样写`StoreProvider`:
```javascript
"use client"

import { persistor, store } from "@/store"
import { ReactNode } from "react"
import { Provider } from "react-redux"
import { PersistGate } from "redux-persist/integration/react"

export default function StoreProvider({ children }: { children: ReactNode }){
    return (
        <Provider store={store}>
            <PersistGate loading={null} persistor={persistor}>
                {children}
            </PersistGate>
        </Provider>
    )
}
```
会导致`children`无法在服务端被渲染。

## 解决方案
来自[nuriun](https://github.com/nuriun)的[方法](https://github.com/vercel/next.js/issues/8240#issuecomment-647699316)

改写成:
```javascript
"use client"

import { persistor, store } from "@/store"
import { ReactNode } from "react"
import { Provider } from "react-redux"
import { PersistGate } from "redux-persist/integration/react"

export default function StoreProvider({ children }: { children: ReactNode }){
    return (
        <Provider store={store}>
            <PersistGate loading={null} persistor={persistor}>
                {() => children}
            </PersistGate>
        </Provider>
    )
}
```

## 原理
`PresistGate`使用了`bootstrapped`这个`state`，并且在`componentDidMount`中设置。

而`componentDidMount`并不在服务端运行。于是就渲染了`loading`中的内容。

代码:

```javascript
  componentDidMount() {
    this._unsubscribe = this.props.persistor.subscribe(
      this.handlePersistorState
    )
    this.handlePersistorState()
  }

  handlePersistorState = () => {
    const { persistor } = this.props
    let { bootstrapped } = persistor.getState()
    if (bootstrapped) {
      if (this.props.onBeforeLift) {
        Promise.resolve(this.props.onBeforeLift())
          .finally(() => this.setState({ bootstrapped: true }))
      } else {
        this.setState({ bootstrapped: true })
      }
      this._unsubscribe && this._unsubscribe()
    }
  }
```
```javascript
  render() {
    if (process.env.NODE_ENV !== 'production') {
      if (typeof this.props.children === 'function' && this.props.loading)
        console.error(
          'redux-persist: PersistGate expects either a function child or loading prop, but not both. The loading prop will be ignored.'
        )
    }
    if (typeof this.props.children === 'function') {
      return this.props.children(this.state.bootstrapped)
    }

    return this.state.bootstrapped ? this.props.children : this.props.loading
  }cover
```

但是如果`children`是一个`funcion`，就会执行：
```javascript
    if (typeof this.props.children === 'function') {
      return this.props.children(this.state.bootstrapped)
    }
```
这与`this.state.bootstrapped`无关。