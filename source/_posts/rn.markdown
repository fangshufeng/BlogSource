---
layout: post
title: "React-Native - FlexBox布局"
date: 2017-07-09 22:01:27 +0800
comments: true
categories: 
tag:
    React-Native
---

弹性盒模型（The Flexible Box Module）,又叫Flexbox，意为“弹性布局”，旨在通过弹性的方式来对齐和分布容器中内容的空间，使其能适应不同屏幕，为盒装模型提供最大的灵活性，Flexbox可以在不同屏幕尺寸上提供一致的布局结构。

<!--more-->
 
在`React-Native`使用`FlexBox`布局能够大大提升我们的开发效率`FlexBox`能做的有
1. 适配屏幕；
2. 快速垂直和水平居中
3. 自动分配宽度和高度。

###一、主轴和侧轴，标志着在主轴和侧轴上的布局方式，在`React-Native`中默认的主轴方式是竖直的，当然，也可以根据需要选择合适的对齐方式。

![](http://upload-images.jianshu.io/upload_images/3279997-6e0f4f74dfa1388e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###二、常见的属性
   
（1）.  `flexDirection`：（row | column）
![](http://upload-images.jianshu.io/upload_images/3279997-af2d833236d15d81.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
import React, { Component } from 'react';
import {
    AppRegistry,
    StyleSheet,
    Text,
    View
} from 'react-native';

export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red'}} >测试一</Text>
              <Text style={{backgroundColor:'yellow'}}>测试二</Text>
              <Text style={{backgroundColor:'gray'}}>测试三</Text>
              <Text style={{backgroundColor:'blue'}}>测试四</Text>

            </View>
        );
    }
}

const styles = StyleSheet.create({
    container: {
          backgroundColor:'skyblue',
      marginTop:100,
    }
});

AppRegistry.registerComponent('untitled', () => untitled);
```

看结果可知默认是竖直的

![](http://upload-images.jianshu.io/upload_images/3279997-e8447df0f7433b97.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

把样式代码改一下

```
const styles = StyleSheet.create({
    container: {
      backgroundColor:'skyblue',
      marginTop:100,
        flexDirection:'row', //添加了这行
    }
});
```


![](http://upload-images.jianshu.io/upload_images/3279997-621b4292a24283e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


（2）.   `justifyContent`：（`flex-start`、`center`、`flex-end`、`space-around`以及`space-between`）来控制主轴的布局样式

     我们分别来试试这个属性

`flex-start`  默认就是这个样式

```
const styles = StyleSheet.create({
    container: {
      backgroundColor:'skyblue',
      marginTop:100,
      flexDirection:'row',
    justifyContent:'flex-start',
    }
});
```


![](http://upload-images.jianshu.io/upload_images/3279997-8f3fce4774122cc0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



`center`居中

![](http://upload-images.jianshu.io/upload_images/3279997-458ebdcacaa8c797.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`flex-end`


![](http://upload-images.jianshu.io/upload_images/3279997-3930894b856f421d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`space-around` 表示单个元素周边的间距 注意对比`space-between`


![](http://upload-images.jianshu.io/upload_images/3279997-b3546770c8ec8814.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`space-between`表示两个元素之间 注意对比`space-around`


![](http://upload-images.jianshu.io/upload_images/3279997-c12c7dc5b4f68206.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


（3）. `alignItems`（`flex-start`、`center`、`flex-end`以及`stretch`）这个是用来控制侧轴方向的对齐方式的，此时的侧轴是Y轴


我们再来吧代码改一下

```
export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red',height:80}} >测试一</Text>
              <Text style={{backgroundColor:'yellow',height:90}}>测试二</Text>
              <Text style={{backgroundColor:'gray',height:50}}>测试三</Text>
              <Text style={{backgroundColor:'blue',height:100}}>测试四</Text>

            </View>
        );
    }
}
```

给每个item都加上一个高度 也就是默认`flex-start`的显示




![](http://upload-images.jianshu.io/upload_images/3279997-48300ece95f53af1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`center` 


![](http://upload-images.jianshu.io/upload_images/3279997-dc887eff67c90304.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`flex-end`


![](http://upload-images.jianshu.io/upload_images/3279997-8c247ac5768387d3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`stretch`

> 注意：要使stretch选项生效的话，子元素在次轴方向上不能有固定的尺寸。以下面的代码为例：只有将子元素样式中的width: 50去掉之后，alignItems: 'stretch'才能生效。


```
export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red'}} >测试一</Text>
              <Text style={{backgroundColor:'yellow'}}>测试二</Text>
              <Text style={{backgroundColor:'gray'}}>测试三</Text>
              <Text style={{backgroundColor:'blue'}}>测试四</Text>

            </View>
        );
    }
}

const styles = StyleSheet.create({
    container: {
      backgroundColor:'skyblue',
      marginTop:100,
      flexDirection:'row',
        alignItems:'stretch',
        height:100,

    }
});
```

![](http://upload-images.jianshu.io/upload_images/3279997-a4b98b315fdb19f2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（4）. `flexWrap`（`nowrap | wrap `）,这个是用来当主轴显示不下的时候该怎么显示的


`nowrap`，默认不换行

`wrap`，显示不下换行

```
export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red',width:90}} >测试一</Text>
              <Text style={{backgroundColor:'yellow',width:100}}>测试二</Text>
              <Text style={{backgroundColor:'gray',width:200}}>测试三</Text>
              <Text style={{backgroundColor:'blue',width:300}}>测试四</Text>

            </View>
        );
    }
}

const styles = StyleSheet.create({
    container: {
      backgroundColor:'skyblue',
      marginTop:100,
      flexDirection:'row',
        alignItems:'stretch',
        flexWrap:'wrap',

    }
});
```


![](http://upload-images.jianshu.io/upload_images/3279997-ea85c495cda3aad8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


（5）. `flex`控制显示范围的

宽度 ＝ 弹性宽度 * ( flexGrow / sum( flexGorw ) )

`flexGrow`是占的份额，`sum( flexGorw )`是总的份额，也就是对应元素在父控件位置所占的比例

```
export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red',flex:1}} >测试一</Text>
              <Text style={{backgroundColor:'yellow',flex:2}}>测试二</Text>
              <Text style={{backgroundColor:'gray',flex:1}}>测试三</Text>
              <Text style={{backgroundColor:'blue',flex:4}}>测试四</Text>

            </View>
        );
    }
}
```

sum( flexGorw ) = 1 + 2 + 1 + 4 = 8
所以:
`测试一` 1 / 8;
`测试二` 2 / 8;
`测试三` 1 / 8;
`测试四` 4 / 8;

![](http://upload-images.jianshu.io/upload_images/3279997-a04949652caf2a8b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（6）. `alignSelf`，给某个item添加例外的

```
export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red',flex:1,height:100}} >测试一</Text>
              <Text style={{backgroundColor:'yellow',flex:2,height:50}}>测试二</Text>
              <Text style={{backgroundColor:'gray',flex:1,height:80}}>测试三</Text>
              <Text style={{backgroundColor:'blue',flex:4,height:30}}>测试四</Text>

            </View>
        );
    }
}

const styles = StyleSheet.create({
    container: {
      backgroundColor:'skyblue',
      marginTop:100,
      flexDirection:'row',
        alignItems:'flex-end',



    }
});
```


![](http://upload-images.jianshu.io/upload_images/3279997-64f42c49ccbc1235.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

像上面的在侧轴上是`flex-end`，如果只想让`测试四`居中怎么办呢，就可以使用`alignSelf`属性


```
export default class untitled extends Component {
    render() {
        return (
            <View style={styles.container}>

              <Text style={{backgroundColor:'red',flex:1,height:100}} >测试一</Text>
              <Text style={{backgroundColor:'yellow',flex:2,height:50}}>测试二</Text>
              <Text style={{backgroundColor:'gray',flex:1,height:80}}>测试三</Text>
              <Text style={{backgroundColor:'blue',flex:4,height:30,alignSelf:'center'}}>测试四</Text>

            </View>
        );
    }
}
````

![](http://upload-images.jianshu.io/upload_images/3279997-d111067851b15549.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

