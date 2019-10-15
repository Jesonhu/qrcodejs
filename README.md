# QRCode.js
QRCode.js is javascript library for making QRCode. QRCode.js supports Cross-browser with HTML5 Canvas and table tag in DOM.
QRCode.js has no dependencies.

## Basic Usages
```
<div id="qrcode"></div>
<script type="text/javascript">
new QRCode(document.getElementById("qrcode"), "http://jindo.dev.naver.com/collie");
</script>
```

or with some options

```
<div id="qrcode"></div>
<script type="text/javascript">
var qrcode = new QRCode(document.getElementById("qrcode"), {
	text: "http://jindo.dev.naver.com/collie",
	width: 128,
	height: 128,
	colorDark : "#000000",
	colorLight : "#ffffff",
	correctLevel : QRCode.CorrectLevel.H,
	done: function () {
        console.log('draw done');
    },
    fail: function (err) {
        console.log('draw fail', err);
    }
});
</script>
```

and you can use some methods

```
qrcode.clear(); // clear the code.
qrcode.makeCode("http://naver.com"); // make another code.
```
## modify

最近有一个需求，需要将连接转换为二维码。

直接找了github最多star的库 [https://github.com/noteScript/qrcodejs](https://github.com/noteScript/qrcodejs)。

在使用中遇到了一些小问题，并且这个库很久不维护了，加上遇到的问题网上并没有很好的解决方案，所以fork一下修改源码，方便个人后续使用。

### problem

在使用问题中发现有下面的几个问题。

1. 不支持模块化

2. 虽然支持npm安装，但是没办法在项目中使用

3. 在某些安卓机（小米）下二维码没有生成

4. 二维码绘制结束或者报错没有提供api来进行通知

### code

粗略地将源码看了一下，简单说明一下运行流程。

1. 将实例化QRcode参数options.text使用qrcode算法转换为矩阵（就是一个二维数组）。

2. 判断当前文件是否是svg文件，是则调用svg api绘制矩阵。不是则根据参数及浏览器支持情况选择绘制方案。

3. 根据options.useSVG选择是否使用svg渲染，如果不是则看浏览器是否支持canvas，不支持则用table绘制。

4. 如果使用canvas绘制矩阵，调用canvas.toDataUrl获得一个base64格式的图片，之后创建img标签插入实例化QRcode时传入参数的dom中。

### resolve

1. 将源码用umd包装

```
(function (root, factory) {
	if (typeof define == 'function' && define.amd) {
		define('qrcode', factory);
	} else if (typeof exports == 'object') {
		module.exports = factory();
	} else {
		root.QRCode = factory();
	}
}(this, function () {
	// ...code

	return QRcode;
}));
```

2. 这个不用说了，修改完代码自己推送一下npm喽～

3. 当遇到二维码不生成这个问题的时候，当然是先<del>百度</del>谷歌一下喽，网上针对这个问题方案有 MutationObserve，setTimeout，甚至还有轮询检测等，基本都是基于观察dom中img的生成和img的src属性是否存在来解决。

比起问题是怎么解决的，我更想知道问题是怎么来的。

查看源码后发现是由于源码中_getAndroid获取版本信息的正则写的有问题，而导致QRCode.prototype.makeImage中本该执行的绘制方法没有执行。

```
QRCode.prototype.makeImage = function () {
	// this._android 在有些安卓机型下为false
	if (typeof this._oDrawing.makeImage == "function" && (!this._android || this._android >= 3)) {
		this._oDrawing.makeImage();
	}
};
```
```
function _getAndroid() {
	var android = false;
	var sAgent = navigator.userAgent;
	
	if (/android/i.test(sAgent)) { // android
		android = true;
		var aMat = sAgent.toString().match(/android ([0-9]\.[0-9])/i);
		if (aMat && aMat[1]) {
			android = parseFloat(aMat[1]);
		}
	}
	
	return android;
}
```

我们可以看到 这个安卓版本正则的获取是有问题的：
1. ua版本必须有两位以上（国内很多安卓机型ua中 ua版本返回的是一位，看来国外友人并不了解国内的手机厂商啊～）
2. 只能匹配10以下版本，并且版本第二位只能匹配一位（听说安卓10已经发布了 233333）

所以需要将正则改写为如下形式

```
/android (\d+(\.\d+)?)/i
```

4. 绘制完成通知：可以在实例化QRcode时传入回调函数，然后根据各种渲染方案调用即可。

### supplement

1. 在测试时发现下面this._android会报错。

```
if (this._android && this._android <= 2.1) {
	var factor = 1 / window.devicePixelRatio;
	var drawImage = CanvasRenderingContext2D.prototype.drawImage; 
	CanvasRenderingContext2D.prototype.drawImage = function (image, sx, sy, sw, sh, dx, dy, dw, dh) {
		if (("nodeName" in image) && /img/i.test(image.nodeName)) {
			for (var i = arguments.length - 1; i >= 1; i--) {
				arguments[i] = arguments[i] * factor;
			}
		} else if (typeof dw == "undefined") {
			arguments[1] *= factor;
			arguments[2] *= factor;
			arguments[3] *= factor;
			arguments[4] *= factor;
		}
		
		drawImage.apply(this, arguments); 
	};
}
```
在cmd中this指向undefined，而将QRcode挂载到window上或者使用amd时this指向则是window。

```
// 修改
var _android = _getAndroid();
if (_android && _android <= 2.1) {
	// ...code
}
```

2. 降级

	虽然源码中做了多种方案，但是因为一些产品和视觉上的限制，可能绘制的二维码并不可用（依赖客户端识别二维码的实现），二维码本意是方便传播，我们自己的需求中查看了数据预估了一下，对一些低版本直接进行了url的展示让用户自己复制。

3. 测试

	源码中有很多关于系统版本的判断，可以使用chrome的调试工具去修改ua测试。

### use

webpack
```
npm install @roronoalee/qrcodejs
```
```
import QRcode from '@roronoalee/qrcodejs'
```

amd 

```
require(['qrcode'], function (QRcode) {
	// ... your code
});
```

cdn

```
<script src="{{YOUR_CND_DOMAIN}}/qrcode.js"></script>
```

## Browser Compatibility
IE6~10, Chrome, Firefox, Safari, Opera, Mobile Safari, Android, Windows Mobile, ETC.

## License
MIT License

## Contact
twitter @davidshimjs

[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/davidshimjs/qrcodejs/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

