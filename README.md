# 小程序测试工具 weapp-test-utils

目前只能测试service层的js代码，对于渲染层的wxml还不能测试，所以测试的原则是比较data数据和检测函数调用的次数
测试分为页面和自定义组件测试

## 安装

npm install weapp-test-utils

因为小程序依赖全局属性Page, Component, Behavior, wx, getApp等，需要在测试框架中进行引入，下面以jest进行引入
在jest配置中添加setupFiles指定的文件，在文件中添加如下代码
```js
import { Page, Component, Behavior, wx } from 'weapp-test-utils'
window.Page = Page
window.Behavior = Behavior
window.Component = Component
window.wx = wx
// 框架暂时没有实现getApp和App，需要用户自己实现getApp
window.getApp = function () {
  return {
    globalData: {},
    getOpenId () {
      return Promise.resolve(123)
    }
  }
}

```

## 注册页面

```js
// 因为页面直接执行Page函数，没有返回值，所以需要手动
// 注册页面而不是通过import引入，目的是能够拿到页面的
// 实例，方便操作
import { register } from 'weapp-test-utils'
// 默认从项目的src目录下查找，可以通过basePath设置根目录，
// register('pages/home/home.js', {basePath: ''})
const page = register('pages/home/home.js')

// 注册页面只是将页面加载进来，获取页面实例，需要通过执行run
// 方法来启动页面，run可以添加参数作为onLoad的参数
// run函数会一次执行onLoad,onShow,onRead
page.run()
```

### 测试技巧

```js
// 页面
Page({
  data: {
    name: 'Weapp'
  },
  onLoad () {},
  setName (name) {
    this.setData({
      name
    })
  }
})

// 下面的测试例子都是基于jest的

it('页面加载', () => {
  // 页面有一个calledTimes属性，用来检测生命周期函数调用的次数
  expect(page.calledTimes['onLoad']).toBe(1)
  expect(page.calledTimes['onShow']).toBe(1)
})

it('修改简单数据', () => {
  expect(page.data.name).toBe('Weapp')
  // 通过call方法调用页面的函数，对于一些dom事件也是类似这样调用
  // 需要注意的是，事件调用需要用户自己封装小程序的参数结构，这里只是
  // 提供了调用的方式
  page.call('setName', 'Jay')
  expect(page.data.name).toBe('Jay')
})

it('重新启动页面', () => {
  // 为了方便测试，提供了reLaunch方法，该方法会重置内部属性，自动执行run方法，
  // 可以添加参数作为onLoad的参数
  page.reLaunch()
  expect(page.calledTimes['onLoad']).toBe(1)
  expect(page.calledTimes['onShow']).toBe(1)
  expect(page.data).toEqual({ name: 'Weapp' })
})

it('页面卸载', () => {
  // pageUnload会执行onHide和onUnload
  page.pageUnload()
  expect(page.calledTimes['onUnload']).toBe(1)
  expect(page.calledTimes['onHide']).toBe(1)
})

it('模拟页面打开：onShow', () => {
  page.enterPage()
})

it('模拟页面打开：onHide', () => {
  page.leavePage()
})

it('模拟页面进入后台：onHide', () => {
  page.background()
})

it('模拟页面进入前台：onShow', () => {
  page.foreground()
})

it('模拟页面刷新：onPullDownRefresh', () => {
  page.pullDownRefresh()
})

it('模拟页面到底：onReachBottom', () => {
  page.reachBottom()
})

it('模拟页面分享：onShareAppMessage', () => {
  page.shareAppMessage()
})

it('模拟页面滚动：onPageScroll', () => {
  page.pageScroll()
})

it('模拟页面tab点击：onTabItemTap', () => {
  page.tabItemTap()
})

// ==== 其它
// 获取页面中的某个属性，需要强调的是这个page只是模拟的页面实例，方便测试，并不是真正的Page的实例
page.get('key')
// 修改页面数据，代理Page的实例的方法
page.setData
```

## 挂载组件

```js
// 组件通过mount进行挂载，挂载的方式和页面一样
import { mount } from 'weapp-test-utils'
const wrapper = mount('components/toast/toast.js')
// 组件启动时，可以添加一些属性，作为使用组件时的参数
// 这些参数可以不传，看实际使用情况
// is表示引入组件的路径
// id组件的id
// dataset在组件中的data-*
// props给properties传值
wrapper.run({
  is: 'components/test/test.js',
  id: 'app',
  dataset: {
    name: 'test'
  },
  props: {
    mask: true
  }
})
```

### 测试技巧

```js
it('组件加载', () => {
  // calledTimes检测生命周期执行次数
  expect(wrapper.calledTimes['ready']).toBe(1)
  // get获取组件的属性
  expect(wrapper.get('is')).toBe('components/test/test.js')
  expect(wrapper.get('id')).toBe('app')
  expect(wrapper.get('dataset').name).toBe('test')
  // 获取组件数据包括properties
  expect(wrapper.data.mask).toBe(true)
})

it('修改properties', () => {
  expect(wrapper.data.duration).toBe(2000)
  // 可以直接通过data修改properties和data的值
  wrapper.setData({
    duration: 1000
  })
  expect(wrapper.data.duration).toBe(1000)
})

it('调用组件方法', () => {
  // 通过call调用组件的方法
  wrapper.call('onAnimationEnd')
})

it('监听组件触发的事件', () => {
  const callback = jest.fn()
  // 通过on监听事件回调
  wrapper.on('close', callback)
  // 组件内部 this.triggerEvent('close')
  expect(callback).toBeCalled()
})

it('组件重新初始化', () => {
  wrapper.reLaunch()
})

it('组件销毁', () => {
  wrapper.destroy()
})
```

## 测试Behavior

Behavior 一般和组件一起测试，也可以单独测试，单独测试需要把 Behavior 当成组件来使用

```js
import feedback from '@/behaviors/feedback'

// 将behaviors封装成组件，测试和组件一样，Component是全局变量
const wrapper = Component(feedback)
```

## API的测试

如果在代码中使用到了wx的api，weapp-test-utils对wx进行了重写（此处有坑），可以直接使用，微信的api基本满足
success，fail，complete的规则。在实际测试中不会真的执行对应的api，需要用户手动执行。如当执行wx.request时，
获取success的方式是wx.request.success({success: true, data: {}}), 获取fail的方式是
wx.request.fail('失败了')，成功失败都会执行complete。

api方法众多，没有全部覆盖，没有覆盖的api主要有websocket相关, 音频视频相关，蓝牙，iBeacon，NFC, Wi-Fi, canvas, wxml节点信息等
后续会陆续加入

```js
// 异步接口测试
// 异步接口如果使用了promise,推荐使用 async 和 flush-promises
// 这里演示的是wx.request使用promise进行了改写
import flushPromises from 'flush-promises'
it('onLoad加载数据', async () => {
  // 进入页面加载数据
  page.run({ ticketId: 1 })

  const data = {
    bizName: '烧烤',
    ticketName: '优惠券',
    ticketId: 1,
    ticketCode: '3849494',
    beginTime: 1524901214895,
    endTime: 1524901214895,
    minConsumption: '200',
    poiInfos: [{ addr: 'abc', name: 'cdf', id: 12 }]
  }
  // 先清空所有promise
  await flushPromises()
  // 执行success会执行wx.request的options的success回调，在
  // 回调函数中会执行promise的resolve
  // 需要强调的是wx.request.success本身是同步的，如果没有使用
  // promise，可以直接比较，不需要async函数
  wx.request.success({
    data: {
      success: true,
      data
    }
  })
  // 等待promise执行完毕
  await flushPromises()
  // 比较处理后的结果
  expect(page.data).toEqual({
    errMsg: '',
    bizName: '烧烤',
    ticketName: '优惠券',
    ticketId: 1,
    ticketCode: '3849 494',
    beginTime: '2018.04.28',
    endTime: '2018.04.28',
    minConsumption: '200',
    poiInfos: [{ addr: 'abc', name: 'cdf', id: 12 }]
  })
})
```

## 后续
1. wx实现的比较粗糙
2. 自定义组件并不完善
3. App和getApp没有实现
4. 无法测试wxml
5. 测试框架本身bug
6. ...