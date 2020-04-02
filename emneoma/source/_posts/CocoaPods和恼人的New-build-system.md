---
title: CocoaPods和恼人的New build system
date: 2020-03-09 01:36:52
tags: [iOS,CocoaPods,NewBuildSystem]
---

<div class="note primary"><p>New build system 把我坑惨了啊 TAT</p></div>

<!-- more -->

# CocoaPods对New build system水土不服

如果您使用CocoaPods与新的Xcode构建系统(New build system)一起为您的应用程序开发库，会有一个对新手极不友好的**BUG** —— 即Xcode在构建pods库时不会获取上次*pod install*之后修改的最新代码。

参见[radar 41126633](https://openradar.appspot.com/41126633)。

 

也就是说，开发过程中每次修改了库代码，你就要重新clean -> build -> run，否则跑的还是旧代码！

“如果你是第一次接触库开发，第一次接触组件化开发，而且你的Xcode刚好升级到了10以上有了新的编译系统，那么你总是会莫名其妙踩进这个怪坑！其次，每次clean之后重新编译，对于比较大的项目就要消耗一堆时间，而且每次等待也是打断了开发的思路，这是非常严重的问题啊！”



最简单的方式就是改回旧的build system：

**File » Workspace Settings » Use Legacy Build**.

> Tips: 记得在每次*pod install*之后，都得去改，因为xcworkspace默认也会使用新编译系统！



# 为什么水土不服？

长话短说：看一下项目的**Build Phases**。 有个**Embed Pods Frameworks**，可将构建的框架复制到项目中。 此功能使用一种称为*Input/Output files*的功能，该功能仅在受影响的文件已更改时才运行脚本。

[![8SknEt.md.png](https://s2.ax1x.com/2020/03/09/8SknEt.md.png)](https://imgchr.com/i/8SknEt)

这似乎在新的构建系统中被打破了，看看 [Xcode bug](https://openradar.appspot.com/41126633) 所说的。



# 更好的解决办法

1. 升级你的CocoaPods版本至>=1.6；

   ```shell
   gem update cocoapods
   ```

2. 然后，您可以将*installation option*（安装选项）添加到**Podfile**中，以禁用*Input/Output files*的使用：

   ```ruby
   use_frameworks!
   
   install! 'cocoapods', :disable_input_output_paths => true
   
   target 'HelloPod_Example' do
     # ...
   end
   ```

3. 现在，当您再次运行pod install时，构建阶段将不再使用输入/输出文件，Cocoapods脚本将始终运行，以保证您的库代码始终是最新的。