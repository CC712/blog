# 组件分隔最佳实践探索

以 cms 最经典的三大段页面为例

```javascript
    <filter/>
    <list/>
    <page/>
```
    
好了，这是一个标准的cms页面，我们现在粗略的分隔了组件，靠的是天然的语义以及功能分隔。



```<filter @filterChange="search" :onChange="search"/>```  
1. emit到父组件,数据的处理过程放到父组件。
2. 传进去一个回调函数，状态维护在filter内部，函数的传参在filter内部完成 
```<list :list="list"/>``` 这种基本上就是纯展示界面，完全依赖外部的props

```<page :page="page" @pageChange="pageChangeHandler"/>``` 
分页插件，需要外部传递一个page,内部会改变page 所以会emit改变了的参数回来

有一个比较复杂的功能，比如说我filter的某个改变，需要引起页数的变化.
```javascript
const search(opt) {
    searchListByOption({
        ...opt
    }).then(
}

```

组件的一些构建原则
---
| |内部状态| handler在内部| 
|--|----|-----|
|是|props 初始化| emit 数据|
|否|props 依赖| 回调函数
---