---
title: iOS-动态加载字体（给Cocoapod库添加自定义字体文件）
date: 2020-03-31 20:20:21
tags: [iOS,开发之术,字体,Cocoapods]
---

{% cq %}

单想给该库增添一个自定义的字体又不想到主工程去做修改，保持主工程干净并且让子库可有自己独有的字体，该怎么做？ 

{% endcq %}

<!-- more -->



# 注意事项

> 字体文件名不代表字体的名字，在向info.plist 文件中添加字体的时候添加的是字体文件的名字。
>
> 但是使用的时候我们需要字体真正的名字，查找方法可以：右键点击文件 -> show in finder -> 文件标题显示的就是字体的真名；
>
> 或者通过遍历打印所有系统安装的字体文件查找：
>
> ```objective-c
> //获得字体族的名字
> NSArray *arr = [UIFont familyNames];
> for (NSString *family in arr) {
>    //打印字体族名
>    NSLog(@"FamilyName:%@", family);
>    NSArray *arr2 = [UIFont fontNamesForFamilyName:family];
>    for (NSString *name in arr2) {
>        //打印字体名
>        NSLog(@"FontName:%@", name);
>     }
> }
> ```





# 常规添加字体步骤

1. 获取到一个字体文件比如 "DS-DIGI-1.ttf"

2. 在info.plist中的"Fonts provided by application"栏新增一个item，填入你要添加的字体文件的文件名"DS-DIGI-1.ttf"

3. 查找到正确的字体名，在项目中调用UIFont.fontWithName使用

   ```objc
   UIFont(name: "DS-Digital", size: 22)
   ```

   



# 私有库添加字体

当开发私有pod库时，单想给该库增添一个自定义的字体又想保持主工程干净独立，并且让其它项目引用子库也不用手动导入字体，子库也能有自己单独管理的字体，该怎么做？ 



### CTFontManager动态添加

0. 将字体文件放到pod库对应的resource（资源）文件夹下

1. 查找库的bundleA，再到bundleA查找字体文件路径URL

2. 运用CTFontManagerRegisterFontsForURL动态加载字体

3. 写成一个UIFont分类，加载库前调用

   ```swift
   extension UIFont {
       static func registerFont(bundle: Bundle, fontName: String, fontExtension: String) -> Bool {
           guard let bundleURL = bundle.url(forResource: "MYComponent", withExtension: "bundle") else {
               fatalError("Couldn't find bundle MYComponent")
           }
           
           let myBundle = Bundle(url: bundleURL)
           guard let fontURL = myBundle?.url(forResource: fontName, withExtension: fontExtension) else {
               fatalError("Couldn't find font \(fontName)")
           }
           
           if let version = Float(UIDevice.current.systemVersion), version >= 7.0 {
               var error: Unmanaged<CFError>?
               if !CTFontManagerRegisterFontsForURL(fontURL as CFURL, .process, &error) {
                   debugPrint("Error registering font: maybe it was already registered. :%@", error.debugDescription)
                   return false
               }
               
           } else {
               guard let fontDataProvider = CGDataProvider(url: fontURL as CFURL) else {
                   fatalError("Couldn't load data from the font \(fontName)")
               }
               guard let font = CGFont(fontDataProvider) else {
                   fatalError("Couldn't create font from data")
               }
               
               var error: Unmanaged<CFError>?
               // 内存泄漏?
               if !CTFontManagerRegisterGraphicsFont(font, &error) {
                   debugPrint("Error registering font: maybe it was already registered. :%@", error.debugDescription)
                   return false
               }
               
           }
           
           return true
       }
   }
   ```

使用方法还是fontWithName就可以了。



### podSpec动态添加

网上还有用podSpec文件，编译脚本进行hook的方法，通过在resources文件中搜索到字体文件，然后在编译期写入到plist文件，方法写起来比较复杂，且对ruby不熟，笔者用这个方法失败了希望有**大神赐教**。

```ruby
Pod::Spec.new do |s|
  # ...
  s.resources = "Resources/*.otf"
  # ...
  s.post_install do |library_representation|
    require 'rexml/document'

    library = library_representation.library
    proj_path = library.user_project_path
    proj = Xcodeproj::Project.new(proj_path)
    target = proj.targets.first # good guess for simple projects

    info_plists = target.build_configurations.inject([]) do |memo, item|
      memo << item.build_settings['INFOPLIST_FILE']
    end.uniq
    info_plists = info_plists.map { |plist| File.join(File.dirname(proj_path), plist) }

    resources = library.file_accessors.collect(&:resources).flatten
    fonts = resources.find_all { |file| File.extname(file) == '.otf' || File.extname(file) == '.ttf' }
    fonts = fonts.map { |f| File.basename(f) }

    info_plists.each do |plist|
      doc = REXML::Document.new(File.open(plist))
      main_dict = doc.elements["plist"].elements["dict"]
      app_fonts = main_dict.get_elements("key[text()='UIAppFonts']").first
      if app_fonts.nil?
        elem = REXML::Element.new 'key'
        elem.text = 'UIAppFonts'
        main_dict.add_element(elem)
        font_array = REXML::Element.new 'array'
        main_dict.add_element(font_array)
      else
        font_array = app_fonts.next_element
      end

      fonts.each do |font|
        if font_array.get_elements("string[text()='#{font}']").empty?
          font_elem = REXML::Element.new 'string'
          font_elem.text = font
          font_array.add_element(font_elem)
        end
      end

      doc.write(File.open(plist, 'wb'))
    end
  end
```

