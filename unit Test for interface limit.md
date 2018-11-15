# 限制接口重复提交与单元测试

## 故事背景

        我司网站的订单提交页面出现间隔1ms~2min不等的的订单提交接口重复调用警告，并且频率很高。

```javascript
<!-- bug代码片段示意 -->
    function doUpload(){
            return new Promise((resolve, reject) => {
                setTimeout(() => {
                    // ...
                    resolve()
                }, 1000);
            }).catch(rej =>console.log(rej))
        }
```

## bug 定位

1. 时间间隔 1ms，猜测是由于连续点击造成的重复提交。
1. 时间间隔 2min，猜测是由于缺少必要提示造成的用户乱点。

## debug

1. 增加逻辑判断
1. 增加 UI 提示，loading 动画以及遮罩

### 初次修改

```javascript
let isUploading = false;
function doUpload() {
    if (isUploading) return; //增加状态校验
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            // ...
            resolve();
        }, 1000);
    }).catch(rej => console.log(rej));
}
```

bug 并没有解决，仍然复现

### 再次修改

```javascript
let isUploading = false;
function doUpload() {
    if (isUploading) return; //增加状态校验
    isUploading = true; //改变状态
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            // ...
            resolve();
        }, 1000);
    }).catch(rej => console.log(rej));
}
```

只是定义了跳出的条件，但是并没有改变全局状态，所以补上，debug 完成。

## 如何排查类似错误？

    当业务代码比较多的时候这样一行改变状态的代码找起来难度不小，有没有什么办法解决这个需求呢？
    引入单元测试。

## 单元测试

> mocha
```javascript
/** test script for mocha  */
const assert = require('assert');
let upload = require('../mockAjax.js')();
//多次调用接口
describe('multi ajax check', () => {
    it('init', () => {
        assert.equal(upload.getState(), false)
    })
    it('1 time and has result', () => {
        assert.equal(upload.doUpload() instanceof Promise, true)
    })
    it('2 time  false', () => {
        assert.equal(upload.doUpload() instanceof Promise, false)
        assert.equal(upload.getState(), true)
    })
})
```
通过判断返回类型以及闭包状态来测试拦截功能是否正常

## 总结
    单元测试的引入，不光是方便debug,某种程度上，为了代码能够被测试，还倒逼了代码的书写方式。显然耦合越大，嵌套越多的代码越难被测试。
    越是容易被测试的代码，函数就越优美，作用域污染的情况就越少。
>There is no such thing as painless test/debug.

引用一句话作为结尾
