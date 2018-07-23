---
layout:     page
title:      【Direct3D 11】关于DirectX 11 SDK的变动
subtitle:   DirectX 11 SDK移入Windows SDK 8.x
date:       2018-07-16
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

# 解惑：别人的Direct3D 11 demo 可能在自己机子上build失败？  

从Win8开始，DirectX SDK 被包含在了 Windows SDK中。  
微软的解释是：DirectX技术和Windows系统结合得越来越紧密，而Windows SDK是Windows上最主要的SDK。  
如果仅仅只是把以前独立发布的DirectX SDK移入Windows SDK中，可能也不会有上面的问题。
问题在于：微软不再发布单独的DirectX SDK，而且剔除了所有的D3DX库（包括D3DX11）。微软去掉了d3dx*.h和d3dx*.lib等等，把剩下的合并到了Windows SDK中。  
（注：D3DX和D3D并不是一个东西。D3DX，即Direct3D Extension是一套比较上层的API库，用于给微软的Direct3D图形API提供辅助。D3DX已经被微软抛弃。）    
[This post](https://docs.microsoft.com/zh-cn/windows/desktop/directx-sdk--august-2009-) 描述了曾经属于DirectX SDK，而现在属于Windows SDK的工具库。也就是说，这里列出的工具库依然可用。  
如果依然想用被抛弃的D3DX库，一般也是可以的，因为微软已经把很多D3DX开源了。比如FX11、DirectXTex，都已经开源了。  

比较容易混淆的还有D3DXMath。Direct3D 9和Direct3D 10都有D3DXMath库，但是Direct3D 11不包含D3DXMath库，取而代之的是XNAMath。然后，随着DirectX SDK移入Windows SDK, XNAMath也不再存在，取而代之的是DirectXMath。其实基本上只是XNAMath改了名字为DirectXMath。当然，头文件、类型名、API都是有改动的，只是替换起来也容易。另外，DirectXMath增加了命名空间DirectX。  

可以参考：[Living without D3DX]( https://blogs.msdn.microsoft.com/chuckw/2013/08/20/living-without-d3dx/)  
这里讲述了怎么替换老版本的API。  
对于D3DXMath，更详细的还可以参考[Working with D3DXMath]( https://docs.microsoft.com/zh-cn/windows/desktop/dxmath/pg-xnamath-migration-d3dx)  
这里专门讲述了怎么替换老的D3DXMath，包括API，数据结构等等。

#Stack Overflow 上关于DirectX SDK以及Windows SDK的一个比较详细的说明
As the person who did the work to merge the DirectX SDK into the Windows SDK, I've addressed this question many times here and elsewhere.
```
If you have Visual Studio 2012, 2013, or 2015 then you already have the Windows 8.x SDK
which supports development for DirectX 11. With VS 2015, you can optionally install the
Windows 10 SDK which is needed to develop for DirectX 12.
```
The official status of the legacy DirectX SDK is addressed on [MSDN](https://docs.microsoft.com/zh-cn/windows/desktop/directx-sdk--august-2009-).  
More details are covered in this series of blog posts:
1. [Where is the DirectX SDK (2015 Edition)?](https://blogs.msdn.microsoft.com/chuckw/2015/08/05/where-is-the-directx-sdk-2015-edition/)
1. [Where is the DirectX SDK (2013 Edition)?](https://blogs.msdn.microsoft.com/chuckw/2013/07/01/where-is-the-directx-sdk-2013-edition/)
1. [Where is the DirectX SDK?](https://blogs.msdn.microsoft.com/chuckw/2012/03/22/where-is-the-directx-sdk/)

With the transition to the Windows SDK, some stuff was 'left behind' and I've moved a lot of that stuff to various GitHub projects. For a complete survey of what ended up where, see these blog posts:
1. [Living without D3DX](https://blogs.msdn.microsoft.com/chuckw/2013/08/20/living-without-d3dx/)
1. [DirectX SDK Tools Catalog](https://blogs.msdn.microsoft.com/chuckw/2014/10/28/directx-sdk-tools-catalog/)
1. [DirectX SDK Samples Catalog](https://blogs.msdn.microsoft.com/chuckw/2013/09/20/directx-sdk-samples-catalog/)
1. [DirectX SDKs of a certain age](https://blogs.msdn.microsoft.com/chuckw/2012/08/21/directx-sdks-of-a-certain-age/)

At this point in time, there are only really two uses for the legacy DirectX SDK as covered in [The Zombie DirectX SDK](https://blogs.msdn.microsoft.com/chuckw/2015/03/23/the-zombie-directx-sdk/):
1. You are developing for Windows XP. This requires the Windows 7.1 SDK because the Windows 8.x SDK and Windows 10 SDK do not support Windows XP and this predates the merge of DirectX. With VS 2012/2013/2015 the Windows XP toolset includes the Windows 7.1A SDK (see [this post](https://blogs.msdn.microsoft.com/chuckw/2012/11/26/visual-studio-2012-update-1/)). While the basic Direct3D 9 header has been in the Windows SDK for many years, there's really no Direct3D 9 utility code anywhere except in the legacy DirectX SDK in the  D3DX library.
1. You are wanting to use XAudio on Windows 7 which requires XAudio 2.7, which is only available in the legacy DirectX SDK. XAudio 2.8 is included with Windows 8 and Windows 10 and the headers are in the Windows 8.x SDK. See [this post](https://blogs.msdn.microsoft.com/chuckw/2012/04/02/xaudio2-and-windows-8/).

Old tutorials and books for Direct3D 11 use the D3DX11 utility library and xnamath or D3DXmath. You can use the legacy DirectX SDK with the Windows 8.x SDK or Windows 10 SDK, but it's tricky due to the inverted include/lib path order. See MSDN for details. Instead, I recommend using the replacements for D3DX listed above such as the [DirectX Tool Kit](https://github.com/Microsoft/DirectXTK/wiki/Getting-Started) and [DirectXMath](https://blogs.msdn.microsoft.com/chuckw/2012/03/26/introducing-directxmath/).

```
If you are using Direct3D 10.x, then you should upgrade to Direct3D 11. The API is extremely similar,
Direct3D 11 is supported on a broader set of platforms and hardware, and you can use the replacements
for D3DX11 to remove any lingering legacy DirectX dependency.
```


# 附录
[一个中学物理老师翻译的DX11龙书](http://shiba.hpe.sh.cn/jiaoyanzu/WULI/Soft/NotXNA)  

[如何使龙书demo运行起来](https://blog.csdn.net/pobber/article/details/51971939?_t=t)

[大神翻译的OpenGL](https://learnopengl-cn.github.io/)

[Direct3D Tutorial Win32 Sample(有些很重要，讲得也很精彩的内容，如Tutorial 4：3D空间)](https://code.msdn.microsoft.com/Direct3D-Tutorial-Win32-829979ef)

[MSDN上关于YUV格式以及渲染的文档](https://docs.microsoft.com/zh-cn/windows/desktop/medfound/recommended-8-bit-yuv-formats-for-video-rendering)
