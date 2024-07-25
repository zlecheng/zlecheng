# 「AntV」X6 自定义vue节点(vue3)

[官方文档](https://x6.antv.antgroup.com/tutorial/intermediate/vue)
> 本篇文档只讲解vue3中如何使用，vue2的可以参考下官方文档

## 安装插件
`@antv/x6-vue-shape`
## 添加vue组件
> 既然使用vue节点，那么我们就需要准备一个vue的组件，这个组件就是节点的一些样式，根据你们的ui自行写代码即可

```vue
<template>
  <div>节点名称</div>
  <div>节点描述</div>
  ……
</template>
```
## 注册vue节点

1. 导入vue节点注册插件

`import { register, getTeleport } from '@antv/x6-vue-shape';`

2. 注册节点
```javascript
register({
  shape: 'custom-vue-node',
  width: 'auto',
  height: 104,
  component: vueNode // 这个就是你定义的vue组件
});
```

3. 添加传送门
```javascript
import { getTeleport } from '@antv/x6-vue-shape';
const TeleportContainer = defineComponent(getTeleport());

// template 中添加标签，和你的画布容器平级
<div id="graphDom"></div>
<TeleportContainer />
```

4. 使用
```javascript
  const node = graph.createNode({
    shape: 'custom-vue-node',
    width: 100,
    height: 104,
    label: data?.name,
    id: data?.id,
    // 所有节点的数据源头都在这里设置，需要哪些字段自行添加即可
    data: {
      name: data?.name, // 节点的名称
      img: data?.img || remoteImgUrl.value, // 图标
      desc: data?.dataNum || 0, // 总数据描述
      ……
    },
    /**
     * 连接桩位置判断逻辑
     * 1、数据源类型的连接桩只显示右侧
     * 2、算子类型的连接桩显示左右两侧
     * 3、算子类型-关联回填的连接桩显示左侧
     */
    ports: {
      ...port
    }
  });
```
## 节点内部监听数据变化
```javascript
const getNodeData = inject('getNode');
onMounted(() => {
  const currentNode = getNodeData();
    // 监听当前节点数据发生了变化
  currentNode.on('change:data', ({ current }) => {
    console.log('节点数据是否发生变化了 >>>', current);
  });
})
```
## vue节点拖拽的时候报错？
![image.png](https://cdn.nlark.com/yuque/0/2024/png/2578582/1715161582677-71762bbd-ddeb-425c-870f-da649b480e2d.png#averageHue=%23fbf0d9&clientId=ueef1645c-9ec3-4&from=paste&height=494&id=u030e6c88&originHeight=741&originWidth=1422&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=164899&status=done&style=none&taskId=u92889cff-a972-4350-8e4f-53cb48f31b2&title=&width=948)
检查你的vue组件是否是这种结构
```html
<template>
  <div>内容：{{ dataNode.name }}</div>
  <n-badge>
    <n-avatar :src="vueIco"></n-avatar>
  </n-badge>
</template>
```
需要改成下面这种的结构(需要用根节点进行包裹)
```html
<div>
  <div>内容：{{ dataNode.name }}</div>
  <n-badge>
    <n-avatar :src="vueIco"></n-avatar>
  </n-badge>
</div>
```
## 节点事件和vue节点内的click事件冲突问题
### 场景
> 因为我用的是vue类型的节点，所以这里就按照vue节点来进行讲解，其它的节点(React、Angular、Html)这些都是通用的。

> 在vue节点内部的某个元素上需要执行一个点击事件，但是在执行本事件的时候不能去触发`node:click`的事件、在执行`node:click`事件的时候不能触发vue节点的点击事件，也就是两边的事件都是独立的，谁也不能影响谁，而且vue节点内的点击事件在点击的时候还得获取当前节点信息

### 踩坑方案1
直接给vue的点击事件添加`stop`修饰符，阻止事件传递，然后在`node:click`的时候再阻止下，但是结果下来确是不行……
```javascript
// vue节点的事件
@click.stop = test

// 父页面的节点节点事件
graph.on('node:click',{e} => {
  e.stopPropagation()
})
```
### 踩坑方案2
采用群里小伙伴的方案，阻止节点鼠标按下或者鼠标抬起的事件，这样可以实现在点击vue节点的时候不触发节点本身的`node:click`事件，虽然可以实现阻止的功能，但是不好操作节点的数据，我是需要获取当前节点的数据的
### 终极解决方案
通过获取click事件的点击区域进行判断，如果是点击了vue节点内的点击事件区域，就直接在node:click的时候阻止掉就行了
```javascript
graph.on('node:click',{e} => {
  // 判断target的className或者id，或者你定义的一些自定义属性，
  // 反正只要你能知道当前点击的区域是属于谁的就行
  // 我在vue节点点击事件的标签上加了个class
  if(e.target.className == 'cu-class') return
})
```

---

## vue节点数据如何反向传递给父组件(vue3)？
提的issues：[https://github.com/antvis/X6/issues/4323](https://github.com/antvis/X6/issues/4323) （这里面有vue2的解决方案，下面这个图片是vue2的解决方法）
![image.png](https://cdn.nlark.com/yuque/0/2024/png/2578582/1720599671114-64a461f5-83e1-4cf1-98bf-064ccf82e136.png#averageHue=%2324211f&clientId=u1a0881ec-08f0-4&from=paste&height=735&id=uab3f81a0&originHeight=1103&originWidth=1271&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=264223&status=done&style=none&taskId=u97f2f055-44e7-48bf-8c89-de764bbf4e9&title=&width=847.3333333333334)
### 场景
vue节点内部有一个复选框，用于勾选节点，选中后要给当前节点添加一个是否选中的属性，由于节点的数据更新只能在父页面进行更新，所以必须得把复选框绑定的值传递给父页面
### 解决方案1
> 这个方案属于野路子，不是很灵活，如果不是复选框那基本凉凉了

```javascript
// vue节点内正常写复选框绑定的逻辑
const checked2 = ref(false);
<el-checkbox v-model="checked2" size="large" @change="checkChange"></el-checkbox>



// 父组件监听节点的点击事件
graph.on('node:click',({e,node}) => {
    let state = node.data.checkState ?? false;
  // 这个判断是为了解决复选框的点击事件和节点的点击事件冲突的问题
    if (e.target.className == 'el-checkbox__inner') {
      // 给节点添加一个checkState属性，标识是否选中
      node.updateData({ checkState: !state }, { ignoreHistory: true });
      return;
    }
})


// 最后点击保存按钮的时候获取下节点checkState为true的数据
const save = () => {
  const allNodes = graph.getNodes();
  // 我这里是取的id属性，如果你们需要其它的可以自行组装
  checkedOps.value = allNodes.filter(item => item.data.checkState).map(item => item.id);
  console.log('checkedOps >>>', checkedOps.value); 
}
```
### 解决方案2
> 这个方案就可以随便玩了，不再局限于我自己的需求，如果还要在节点上加其它的控件都可以完美的把数据传递到父组件，其灵感来源于github的小伙伴[qw123gz](https://github.com/qw123gz)，问官方交流群的群主，问了半天也没有给出方案……

#### 子组件添加emit事件
```javascript
<el-checkbox v-model="checked2" size="large" @change="checkChange"></el-checkbox>


const checked2 = ref(false);
const emits = defineEmits(['getCheckVal']);
const checkChange = val => {
  emits('getCheckVal', val);
};
```
#### 父组件改造注册vue节点的代码
```javascript
register({
  shape: 'custom-vue-node',
  width: 'auto',
  height: 104,
  // component: vueNode   这个是官方提供的注册方式
  component: {
    // 使用vue3的render渲染组件，并添加自定义事件
    render() {
      return h(vueNode, {
        // 事件名称前面必须添加 `on`
        onGetCheckVal: val => getMyCheckVal(val)
      });
    }
  }
});
```
至此，数据反向传递就完成了，至于怎么使用传递过来的数据就看你们的业务需求了
## vue节点重复渲染问题
> 今天在封装画布节点预览组件的时候，发现从这个组件跳转到编辑页面的时候，节点重复渲染了，也就是原来有两个节点，现在变成了4个节点，原本以为是我本地缓存，但是刷新页面也还是会重复渲染……，那这就有点摸不着头脑了，于是开始去社区找小伙伴们的帮助，确定了问题的根源在于`TeleportContainer`，最后社区的小伙伴也是给了个下面的解决方案

### 解决方案1
由于是两个毫不相关的父页面，而且两个父页面都使用了`TeleportContainer`，解决的方法就是如何只保留一个，也就是在路由跳转的时候携带个参数过来，告知当前父页面是否需要删除 `TeleportContainer`
```javascript
// 父页面A
route.push('/fb?delTeleport=true');

// 父页面B
<TeleportContainer v-if="!delTeleport"/> 
```
### 解决方案2
推荐第二种写法，毕竟眼不见心不烦，既然只能保留一个`TeleportContainer`，那就把它放在所有页面的共同父组件里面，也就是根组件 `App.vue`里面
```vue
<template>
    <TeleportContainer />
</template>


import { getTeleport } from '@antv/x6-vue-shape';
// x6的vue节点数据传送门
const TeleportContainer = defineComponent(getTeleport());
```

![7b49173b21c936862b568f83c1dae9a.jpg](https://cdn.nlark.com/yuque/0/2023/jpeg/2578582/1686119457435-59195b9d-534e-47c0-9d4b-266bccc5d638.jpeg#averageHue=%23fbfbfa&clientId=u86232ec9-518c-4&from=paste&height=23&id=v5dhq&originHeight=34&originWidth=1280&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=4234&status=done&style=none&taskId=ua82c9a7e-a8b4-4064-bf5b-c50965b1dee&title=&width=853.3333333333334)

#####    🚀 官方链接

- [Dnd插件](https://x6.antv.antgroup.com/tutorial/plugins/dnd)
- [Stencil插件](https://x6.antv.antgroup.com/tutorial/plugins/stencil)
- [图片导出](https://x6.antv.antgroup.com/tutorial/plugins/export)
- [1.x常见问题](https://antv-x6.gitee.io/zh/docs/tutorial/epilog#%E5%AF%BC%E5%87%BA%E5%9B%BE%E7%89%87%E6%A0%B7%E5%BC%8F%E7%BC%BA%E5%A4%B1)
- [坐标转换](https://antv-x6.gitee.io/zh/docs/tutorial/log#19x)
- [Transform](https://antv-x6.gitee.io/zh/docs/api/graph/transform/#translating)
- [Model](https://antv-x6.gitee.io/zh/docs/api/graph/model/#getconnectededges)
- [getsuccessors](https://antv-x6.gitee.io/zh/docs/api/graph/model/#getsuccessors)
- [X6 Vue3 Components](https://daizhigitee.gitee.io/x6-vue3-components/)
- [微信收录文章](https://mp.weixin.qq.com/s/IGIjKjMMxaLvnimXbX5YDg)
#####    ✈️ demo体验

- [关系图谱(国内镜像)](https://sxdpanda.gitee.io/graph-visualization/#/index)
- [关系图谱(备用地址)](https://zlecheng.github.io/atlas-pages/#/index)
- [拖拽demo(国内镜像)](https://sxdpanda.gitee.io/antv-admin/)
- [自定义拖拽(备用地址)](https://zlecheng.github.io/antv-page/#/home)
#####     🌍 关联文章

- [自定义拖拽](https://www.yuque.com/sxd_panda/antv/dnd)
- [框选添加右键菜单](https://www.yuque.com/sxd_panda/antv/selection)
- [图片导出问题汇总](https://www.yuque.com/sxd_panda/antv/export)
- [自定义html节点](https://www.yuque.com/sxd_panda/antv/html-node)
- [自定义vue节点(vue3)](https://www.yuque.com/sxd_panda/antv/vue-node)

![111.webp](https://cdn.nlark.com/yuque/0/2023/webp/2578582/1702559928278-65460ac2-56fb-4629-a341-a9f10e76e8ec.webp#averageHue=%23181614&clientId=ub724a9d6-9de2-4&from=paste&height=102&id=V5qKo&originHeight=127&originWidth=341&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=6318&status=done&style=none&taskId=uc8e7c8c7-462b-4546-bd47-5cd60f690d1&title=&width=272.8)
