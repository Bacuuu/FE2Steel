### 记录一次难忘的useState使用



#### 需求

​		在一次`Taro`小程序开发过程中，我需要在当前页面切换到其他页面返回后，去判断当前用户是否改变，以调用不同的接口，当用户未改变时则不需要进行接口的重新调用。



#### 实现与调试

​		不假思索的，我写下如下代码

```javascript
  	const [req, setReq] = useState(new Function())
    // 切换至当前页面时会执行
	useDidShow(() => {
    let _req = user.userInfo ? getUserRecommendSheet : getRecommendSheet
    // 如果有所改变
    if (req !== _req) {
      setReq(_req)
      _req({ limit: 6 }).then(r => {
        setSongSheet(r.result || r.recommend.slice(0, 6))
      })
    }
  })
```

​		我的思路是，当第一次加载后，如果我不去改变`user.userInfo`，由于`req === _req`，所以`if`后的语句一直不会执行。但是运行之后发现每次都会去执行。那只有使用`console`大法调试了。

​		在`setReq(_req)`后，我添加了`console.log(req)`，以查看值发生了变化了吗，是否和`_req`一致。打印结果如下图。

![](https://cdn.jsdelivr.net/gh/Bacuuu/sleeping-image@master/uPic/ISZhLT.jpg)

​		**为什么会是一个具有`cookie`属性的函数？**

​		这里就涉及到`setState`的使用，`react`[官方文档](https://zh-hans.reactjs.org/docs/hooks-reference.html#usestate)中是这么描述的

> `setState` 函数用于更新 state。它接收一个新的 state 值并将组件的一次重新渲染加入队列。
>
> ```javascript
> setState(newState);
> ```
>
> 
>
> **函数式更新**
>
> 如果新的 state 需要通过使用先前的 state 计算得出，那么可以将函数传递给 `setState`。该函数将接收先前的 state，并返回一个更新后的值。

​		原来我们的本意是将函数传入`setState`作为一个值进行更新，但是实际上是进行了函数式更新。我们顺着函数式更新往下分析。传入`setState`的函数如下

```javascript
const getUserRecommendSheet:any = (params ?:Object) => http.post('/recommend/resource', params)
```

​		首先分析为什么会增加一个`cookie`属性，这里我们可以看到，我们将`params`，也就是请求参数作为上一次的值传入`setState`。根据接口来看，该请求是没有请求参数的，但是在请求拦截器中有这样的逻辑：

```javascript
function requestInterceptor (chain) {
  const { requestParams } = chain
  // 将cookie放入请求参数中
  requestParams.data.cookie = Taro.getStorageSync('cookie')
  changeLoading('on')
  return chain.proceed(requestParams)
    .then(responseSuccessInterceptor)
    .catch(responseErrInterceptor)
}
```

​		这里的`requestParams.data`就是请求参数，也就是在这里将`cookie`属性挂到了`state`上。接着往下分析，请求函数最终返回的是一个`Promise`。

​		**为什么控制台打印的是一个函数？**

​		其实很简单，我们再切出页面再返回就能发现打印结果变了，这是因为`setState`函数式更新是异步的，这里的函数是我们初始化时的函数

![](https://cdn.jsdelivr.net/gh/Bacuuu/sleeping-image@master/uPic/DJczXt.jpg)

​		现在的打印结果是一个`Promise`了，但是

**为什么也会有`cookie`属性？**

​		注意我们的操作步骤，我们是第二次进入这个页面，所以第二次我们能看到这个`state`是`Promise`，但是此时又去执行了`setReq(_req)`，重复上面的思路可以知道，这会将当前值在请求拦截器中增加`cookie`字段，这里我们看到的`cookie`是在第二次更新时添加上去的。



#### 结论

​		**如果需要通过`setState`对变量赋函数类型值，切记要通过函数式更新方法去处理。**

```
setState(() => newFunction)	
```



