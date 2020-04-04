---
title: iOS-镂空效果遮罩
date: 2020-04-04 10:23:01
tags: [iOS,开发之术,镂空,BezierPath,CAShapeLayer]
Categories: iOS
---

{% cq %}

做新手指引/扫描二维码等的需要的镂空效果。

{% endcq %}

<!-- more -->

# 效果图

(额，毛玻璃是我后期加的为了保护隐私实际是没有这一层的)

[![G0XpX6.png](https://s1.ax1x.com/2020/04/05/G0XpX6.png)](https://imgchr.com/i/G0XpX6)

# 核心代码

```swift
override func draw(_ rect: CGRect) {
        super.draw(rect)
        
        clearBgView.layer.sublayers?.removeAll()
        
        // 全路径
        let bgPath = UIBezierPath(rect: self.bounds)
        // 镂空路径
        let hollowPath = UIBezierPath(roundedRect: hollowRects[idx], cornerRadius: 4)
        bgPath.append(hollowPath)
  
  /*
        // 虚线部分（需求，可以不写）
        let line = CAShapeLayer()
        line.fillColor = UIColor.clear.cgColor
        line.strokeColor = UIColor.white.cgColor
        line.lineWidth = 1
        line.lineDashPattern = [3, 3]
        line.path = hollowPath.cgPath
        clearBgView.layer.addSublayer(line)
  */
        // 遮罩
        let mask = CAShapeLayer()
        mask.path = bgPath.cgPath
        mask.fillRule = kCAFillRuleEvenOdd
        //mask.fillColor = UIColor.black.cgColor
        //mask.opacity = 0.7
        clearBgView.layer.addSublayer(mask)
        

        DispatchQueue.main.async {
            //... 在这里添加或修改 image/label
        }
    }
```

- 每次点击，就调用**`self.setNeedsDisplay()`**，唤起`drawRect`方法重绘视图
- {% label info@hollowPath %} ：镂空路径，每次点击就根据传入的hollowRects数组修改path的rect，并且重新将hollowPath append 到 bgPath，设置了EvenOdd属性之后，则重叠部分路径才会渲染（奇偶原则，偶数路径内视为外部，不填充）

- {% label warning@clearBgView %}：背景，如果clearBgView是半透明黑色，那么直接给他设置mask遮罩就行了；如果 clearBgView是透明的，那么遮罩就加上fillColor和opacity属性。
- {% label success@maskView %}：maskView要设置 **`usesEvenOddFillRule`** 为true

# 补充

>`even-odd rule`与`non-zero rule`
>
>- 对于奇偶规则(even-odd rule)，如果路径交叉的总数为奇数，则认为该点在路径内部，并且填充了相应的区域。如果交叉点的数量为偶数，则认为该点在路径之外，并且该区域未被填充。
>- 对于非零规则(non-zero rule)，从左到右路径的交叉点算作+1，从右到左路径的交叉点算作-1。如果交叉的总和不为零，则认为该点在路径内，并且相应的区域被填充。如果总和为0，则该点在路径之外，并且该区域未被填充。

因此如果镂空部分是圆形或者椭圆形，则可以直接使clockwise属性为false，不修改usesEvenOddFillRule属性，默认是非零规则！也可达到镂空效果噢。