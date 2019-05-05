# 如何高性能的给UIImageView 加个圆角?

不好的解决方案：使用下面的方式会`强制Core Animation提前渲染屏幕的离屏绘制, 而离屏绘制就会给性能带来负面影响`，会有卡顿的现象出现。
 

```
self.view.layer.cornerRadius = 5.0f; 
self.view.layer.masksToBounds = YES; 
```


正确的解决方案：使用绘图技术



```
- (UIImage *)circleImage { 
    // NO代表透明 
        UIGraphicsBeginImageContextWithOptions(self.size, NO, 0.0);
        // 获得上下文 
        CGContextRef ctx = UIGraphicsGetCurrentContext(); 
        // 添加一个圆 
        CGRect rect = CGRectMake(0, 0, self.size.width, self.size.height);
        CGContextAddEllipseInRect(ctx, rect); 
        // 裁剪 
        CGContextClip(ctx); 
        // 将图片画上去 
        [self drawInRect:rect]; 
        UIImage *image = UIGraphicsGetImageFromCurrentImageContext(); 
        // 关闭上下文
        UIGraphicsEndImageContext(); 
        return image;
 } 
```
  
* 还有一种方案：使用了贝塞尔曲线"切割"个这个图片, 给UIImageView 添加了的圆角，其实也是通过绘图技术来实现的。* 


```
UIImageView *imageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
imageView.center = CGPointMake(200, 300); 

UIImage *anotherImage = [UIImage imageNamed:@"image"];
UIGraphicsBeginImageContextWithOptions(imageView.bounds.size, NO, 1.0);
[[UIBezierPath bezierPathWithRoundedRect:imageView.bounds cornerRadius:50] addClip];
[anotherImage drawInRect:imageView.bounds];
imageView.image = UIGraphicsGetImageFromCurrentImageContext();UIGraphicsEndImageContext();
[self.view addSubview:imageView];
```



