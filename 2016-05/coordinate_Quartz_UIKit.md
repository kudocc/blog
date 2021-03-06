# Coordinate Quartz UIKit

坐标系一直是我头痛的问题，在多年以前看Quartz 2D Programming Guide的时候就比较懵逼，当时貌似研究了一阵子似乎是懂了但是其实应该还是云里雾里吧，至于我后来都忘了当时的思考与结论。我觉得混乱是因为UIKit和Quartz中的API使用的是不同的坐标系。UIKit零点在左上角，Y轴向下增长；Quartz的零点是在左下角，Y轴向上增长，所以某些情况下要做一些坐标坐标变换，今天我想详细说说这些内容。

## Current Transform Matrix

无论是UIKit还是Quartz的坐标系其实都是user space坐标系，就是我们自己定义的坐标系。为什么不直接使用一个自己定义的坐标系而不使用设备的坐标系呢？
我们首先说说这个user space坐标系的作用：

1. 它为我们增加了一层抽象，我们可以使用统一的单位来开发，但是在不同的屏幕上可能产生不同的像素数。我们平时开发的时候坐标单位都是点(point)，实际苹果绘制图片的时候都是以像素为单位，但是因为设备不同，一个点对应的像素是不同的，iPhone3gs是一个点对应1个像素，而iPhone4是2*2个像素，如果我们写代码的时候使用的是像素，那就麻烦很多，比如我们要画一条线到屏幕中间，那在3gs上就是画到160位置，而4上就是320的位置，6Plus上就是480...总之使用point隐藏了设备的分辨率，虽然不同分辨率的设备上的宽高比也不同，但是起码不会差别太大。

2. 第二个是我们可以通过对坐标系做变换，达到让绘制的图形变换的效果。比如可以通过让坐标系转45度，让用相同代码绘制的图片旋转45读。

图像真正展示在设备上，我们需要将user space坐标转换成设备坐标，这个工作就是通过current transform matrix (CTM)来做的：CTM是一个`CGAffineTransform`类型的值，其是一个3*3的矩阵，其最后一列是固定的，所以我这里不会表示出来，我们这样表示这个`CGAffineTransform`:[x00, x01, x10, x11, x20, x21]，下标第一位表示行号，第二位表示列号。

在我们做绘制时，先要搞清楚的是所在的上下文，即CGContextRef，或者更具体的说是CTM。UIKit的CTM是[scale, 0, 0, -scale, 0, height]，scale就是方法`[UIScreen mainScreen].scale`的值，height就是这个context的高度(其实UIKit中的CTM是被处理过的，如果我们手动创建一个位图context，然后查看其CTM，会发现其值为`CGAffineTransformIdentity`([1, 0, 0, 1, 0, 0]))。正是因为这个CTM，所以我们的绘制时使用的位置可以用UIKit的坐标系来解释。

为什么CTM是这个值就能将UIKit的坐标正确转换成设备相关的坐标，比如我们绘制一个UIView在iPhone6上，宽高是100*200，使用UIKit坐标系的(0，10)上画一个点，实际上应该在(0, 380)的地方绘制，你可以自己手动将(0, 10, 1)应用[2, 0, 2, -2, 0, 200]，看一下结果是否是(0, 380, 1)。

举一个例子，在iPhone6设备上，覆盖一个UIView的`-drawRect`方法，UIView的长*宽是100*200，绘制一个矩形`CGContextFillRect(context, CGRectMake(0, 0, 10, 10));`，我们认为这个图会被绘制在`UIView`的左上角验证x轴向右，y轴向下分别20个像素的位置。我们知道`CGContextXXX`方法属于Quartz，他们使用的坐标系是y轴向上的，如果此时的CTM是值为`CGAffineTransformIdentity`，那么这个操作会绘制在UIView的左下角，沿着x轴向右，y轴向上10个像素的位置。因为有CTM，`CGRectMake(0, 0, 10, 10)`变成了`CGRectMake(0, 400, 20, 20)`，这个值就是以像素为单位的真实的设备相关坐标。

## 学什么要知道这些

知道这些有屌用？当然，至少对我来说有用，之前做Core Text的时候，要使用UIKit的context绘制文本，但是Core Text底层使用的是Core Graphics来绘制的，也就是与Quartz一样，使用的是y轴向上的坐标系，在计算排版的时候，得到的数值都是在这个坐标系上的，所以绘制时使用这个坐标系就比较方便。将UIKit的坐标系变成了Core Text的坐标系。将UIKit的点变成Core Text的点可以考虑如此变换：(x,y)->(x,height-y)，通过修改CTM，可以达到这个效果：

```
    CGContextTranslateCTM(context, 0, size.height);
    CGContextScaleCTM(context, 1.0, -1.0);
```

在使用Core Text的时候不可避免需要绘制图片，我们拿到的图片一般都是UIImage而不是CGImageRef，UIImage中带有图片的方向(imageOrientation)，如果使用CGContextDrawImage来绘制的话就比较麻烦，因为它的参数是CGImageRef，是没有图片方向信息的，所以绘制出的图片方向可能是错的。所以我就考虑使用UIKit(UIImage)的方法(`-drawInRect:`)，此时又要把CTM变换回UIKit的CTM，使用的公式还是(x,y)->(x,height-y)，所以仍然可以应用之前提到的变换来达到目的，还有一点需要注意，Core Text中计算得到的图片位置是在Core Text坐标系中的位置，如果要传给drawInRect，还需要将这个CGRect从Core Text的坐标系变换成UIKit，代码如下：

```
    CGAffineTransform transform = CGAffineTransformMakeTranslation(0, size.height);
    transform = CGAffineTransformScale(transform, 1, -1);
    frame = CGRectApplyAffineTransform(frame, transform);
```

## 总结

可能有点乱，但其实只需要掌握一条原则，就是调用CGContextXXX的方法时，需要明白当前的CTM是什么，然后就以这个坐标系来计算绘制图形的位置；而调用UIKit的方法时，因为我们不知道其在内部做过什么处理，最稳妥的方式就是将CTM变成UIKit的CTM。

文中提到的Core Text的使用可以在[这里](https://github.com/kudocc/CCKit/blob/master/CCKit/text/CCTextLayout.m)找到。
