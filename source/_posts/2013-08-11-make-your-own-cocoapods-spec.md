---
title: 制作自己的CocoaPods spec
author: lancy
layout: post
comments: true
categories:
  - iOS
---
## 前言
关于CocoaPods，相信不用我介绍更多了。本文主要介绍如何制作自己的CocoaPods spec。

## 步骤
1. 首先你要会用git，还要有一个托管在云端的repo，本文以Github为例，Git和Github的使用方式参照[Github Help](http://github.com/help)
2. 在你的repo下面，使用Git的tag功能，给你的某个commit添加一个tag(比如1.1.0），并push到Github.
    
        // 本地添加一个标签:
        $ git tag -a 1.1.0 -m "Version 1.1.0 Stable"
        // Push tag to GitHub:
        $ git push --tags

3. Folk [CocoaPods/Specs](https://github.com/CocoaPods/Specs) 并 Clone 到本地。
4. 在Clone下来的Specs/创建一个自己的spec的目录，再创建一个版本目录。比如：

        Specs/CYHelper/1.1.0
5. 在该目录下创建一个spec档案，并编辑：

        $ pod spec create CYHelper
        $ vi CYHelper.podspec

    pod创建模板会有相关的说明，按指引一步一步填即可。例如，CYHelper的spec配置如下:

        Pod::Spec.new do |s|
          s.name         = "CYHelper"
          s.version      = "1.1.0"
          s.summary      = "CYHelper is an Objective-C library for iOS developers."
          s.homepage     = "https://github.com/lancy/CYHelper"
          s.license      = 'MIT (LICENSE)'
          s.author       = { "lancy" => "lancy1014@gmail.com" }
          s.source       = { :git => "https://github.com/lancy/CYHelper.git", :tag => "1.1.0" }
          s.platform     = :ios, '5.0'
        
          s.source_files = 'CYHelper', 'CYHelper/**/*.{h,m}'
          s.exclude_files = 'CYHelperDemo'
        
          s.frameworks = 'Foundation', 'CoreGraphics', 'UIKit'
          s.requires_arc = true
        end

6. 验证podspec

        pod spec lint CYHelper.podspec        
        
    如果验证成功的话，会有这样的提示

        Analyzed 1 podspec.
        
        CYHelper.podspec passed validation.

7. 最后去Github上发一个PullRequest，等待一段时间的审核和Merge，之后就可以像别的pod那样用CocoaPods来管理了：
        
        // Podfile
        platform :ios, '6.0'
        pod 'CYHelper' 
        
        $ pod install       
  
      

Have Fun!

## 后注
* [CYHelper在这里，欢迎试用](https://github.com/lancy/cyhelper)
* [顺便求fo我的github](https://github.com/lancy)

这里有唐巧和王轲写的两篇相关的文章，可以作为扩展阅读：

 * [使用CocoaPods来做iOS程序的包依赖管理](http://blog.devtang.com/blog/2012/12/02/use-cocoapod-to-manage-ios-lib-dependency/)
* [CocoaPods进阶：本地包管理](http://www.iwangke.me/2013/04/18/advanced-cocoapods/)