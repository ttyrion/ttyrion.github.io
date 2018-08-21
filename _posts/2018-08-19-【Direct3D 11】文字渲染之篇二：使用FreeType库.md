---
layout:     page
title:      【Direct3D 11】文字渲染之篇二：使用FreeType库
subtitle:   FreeType字体库
date:       2018-08-5
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---
## 前言

>FreeType是一个免费的可用于渲染字体的库，官网地址是[FreeType](https://www.freetype.org/index.html)。由于DirectX 11 没有了文字渲染接口，很多开发者推荐FreeType方式来渲染字体。  

## FreeType渲染字体的大体流程
1. 通过FreeType载入字体文件
2. 通过字符的utf-32编码值(我这里选择的字符映射表是Unicode字符映射表,后面有相关代码)获取到字符的位图
3. 将字符的位图渲染出来

## FreeType头文件
包含FreeType的头文件方式如下：  
```cpp
#include <ft2build.h>
#include FT_FREETYPE_H
```  
从FreeType 2.1.6开始，旧式的头文件包含模式将不会再被支持也就是说，下面的头文件包含方式会出错：  
```cpp
#include <freetype/freetype.h>
#include <freetype/ftglyph.h>
```
如果代码中需要包含FreeType其他的头文件，也一样需要用这种包含方式，比如我这里需要包含的 outline、glyph相关的头文件：  
```cpp
#include FT_OUTLINE_H
#include FT_GLYPH_H
```
**相关的头文件宏定义在 ftheader.h 中。**

## FreeType初始化  
这里我定义了一个字体类FreeTypeFont，来负责处理字体加载的工作。  
初始化FreeType，调用FT_Init_FreeType即可。因为库只需初始化一次，我这里定义了两个static函数：  
```cpp
public:
        static bool Init();
        static void UnInit();
        bool LoadFont();
        bool GetTextData(UINT char_utf32_code, TextData& data);
private:
        static std::wstring system_fonts_dir_;
        static FT_Library lib_;
        FT_Face face_ = nullptr;
        std::wstring font_name_;
        UINT font_size_ = 0;
```
外部代码在初始化和退出时调用FreeTypeFont::Init() 和 FreeTypeFont::UnInit() 即可。实现如下：  
```cpp
  bool FreeTypeFont::Init() {
      if (!lib_) {
            FT_Error err = FT_Init_FreeType(&lib_);
            if (err) {
                return false;
            }
        }

        WCHAR buffer[MAX_PATH] = { 0 };
        if (!::GetWindowsDirectory(buffer, MAX_PATH)) {
            return false;
        }
        system_fonts_dir_ = buffer;
        system_fonts_dir_ += L"\\Fonts";

        return true;
    }

    void FreeTypeFont::UnInit() {
        if (lib_) {
            FT_Done_FreeType(lib_);
        }

        lib_ = nullptr;
    }
```

## 载入字体文件，设置字体大小
使用FT_New_Face可以从一个指定文件载入字体，也可以使用FT_New_Memory_Face从一个内存地址处载入字体。  
有些字体格式的字体文件可能若干个字体外观(FT_Face)，一般情况下，在载入字体文件的时候，总是选择第一个(index == 0)字体外观。然后根据字体外观的num_faces来获取该文件中的外观数。
一个face包含了一个字体外观的相关信息，比如 face_flags 和 style_flags 描述了一个字体外观对象的属性和风格，我们可以根据要求的效果来遍历以及选择合适的face。这里简单处理，选择
第一个face。
```cpp
  bool FreeTypeFont::LoadFont() {
        std::wstring file = system_fonts_dir_ + L"\\msyh.ttc";

        if (!::PathFileExists(file.c_str())) {
            return false;
        }

        std::string font_file = CW2AEX<>(file.c_str(), CP_UTF8);
        if (face_) {
            FT_Done_Face(face_);
            face_ = nullptr;
        }
        FT_Error err = FT_New_Face(lib_, font_file.c_str(), 0, &face_);

        if (err == FT_Err_Unknown_File_Format) {
            //font file accessed successfully, but it has an unknown format

            return false;
        }
        else if (err) {
            //failed on other reasons

            return false;
        }

        //Actually by default, when a new face object is created, FreeType tries to select a Unicode charmap

        FT_Select_Charmap(face_, FT_ENCODING_UNICODE);
        if (!face_->charmap) {
            FT_Done_Face(face_);
            face_ = nullptr;

            //the current font does not have a Unicode charmap

            return false;
        }

        FT_Set_Pixel_Sizes(face_, 0, font_size_);

        return true;
    }
```
**注意：**  
1. 上面的代码中，调用 FT_Set_Pixel_Sizes 设置字体大小。也可以用 FT_Set_Char_Size 来设置字体大小，那就需要以point为单位（A point is a physical distance, equaling 1/72th of an inch.），
还要考虑DPI, FreeType根据这些信息来计算字体的像素大小。很明显，这比直接用 FT_Set_Pixel_Sizes 复杂得多。
2. 通常一个字体文件会包含多个字符映射表，以提供对多种常用的字符编码的支持。因为要显示汉字，我上面选择了Unicode字符映射表，因此后面获取字形索引的时候要用字符的utf-32编码。
3. 另外，face对象使用完后需要调用 FT_Done_Face 释放，我这里在FreeTypeFont析构的时候释放face：
```cpp
FreeTypeFont::~FreeTypeFont() {
        if (face_) {
            FT_Done_Face(face_);
            face_ = nullptr;
        }
    }
```

## 获取字符位图： 根据字形索引来装载字形
字形可以理解为就是字符的位图。
1. 字形索引： 我们可以通过 FT_Get_Char_Index 来获取一个字符的字形索引，参数是face和字符的utf-32编码。获取了字形索引之后，就可以装载字形了。
2. 装载字形： 可以通过 FT_Load_Glyph 来装载字形。对于固定尺寸字体格式，每个字形都是一个位图。对于可伸缩字体格式，则使用名为轮廓的矢量形状来描述每一个字形。**字形位图存储在字形槽中，一个FT_Face对象只有一个字形槽。所以每次只能获取一个字符串中的一个字符对应的字形。**
对于固定尺寸的字体格式，由于获取到的字形是位图，所以可以直接使用，而对于可伸缩格式的字体，装载的是一个轮廓，因此还必须通过 FT_Render_Glyph 函数将轮廓渲染成位图，方可使用。
3. 字形位图： 获取到位图之后，可以通过face->glyph->bitmap来访问位图数据。

代码如下：
```cpp
bool FreeTypeFont::GetTextData(UINT char_utf32_code, TextData& data) {
        if (!face_) {
            return false;
        }

        FT_UInt glyph_index = FT_Get_Char_Index(face_, char_utf32_code);
        FT_Error err = FT_Load_Glyph(face_, glyph_index, FT_LOAD_DEFAULT);
        if (err) {
            return false;
        }

        if (face_->glyph->format != FT_GLYPH_FORMAT_BITMAP) {
            err = FT_Render_Glyph(face_->glyph, FT_RENDER_MODE_NORMAL);
            if (err || face_->glyph->format != FT_GLYPH_FORMAT_BITMAP) {
                return false;
            }
        }

        data.bitmap.width = face_->glyph->bitmap.width;
        data.bitmap.height = face_->glyph->bitmap.rows;
        data.metrics.width = face_->glyph->metrics.width / 64;
        data.metrics.height = face_->glyph->metrics.height / 64;
        data.metrics.bearingX = face_->glyph->metrics.horiBearingX / 64;
        data.metrics.bearingY = face_->glyph->metrics.horiBearingY / 64;
        data.metrics.advance = face_->glyph->metrics.horiAdvance / 64;

        switch (face_->glyph->bitmap.pixel_mode) {
        case FT_PIXEL_MODE_BGRA: {

        }
                                 break;
        case FT_PIXEL_MODE_GRAY: {
            data.bitmap.format = IMAGE_FORMAT_GRAY;
            data.bitmap.buffer.append((char*)face_->glyph->bitmap.buffer, data.bitmap.width * data.bitmap.height);
        }
                                 break;
        default:
            return false;
        }

        return true;
    }
```
我上面代码中，使用std::string来管理字形位图数据。另外，也记下了字形的bearingX、bearingY、advance等等，这些数据在排版字体的时候要用到。对于这些参数的意义，
可以看FreeType官网上的示意图和说明（[查看这里](https://www.freetype.org/freetype2/docs/tutorial/step2.html)）。  

至此，一个字符的位图已经加载到了，剩下就是渲染，比如我用的是Direct3D的纹理贴图。顺便提一下，对于上面的 FT_PIXEL_MODE_GRAY 格式的灰度图，在像素着色器里面把r,g,b设置为一个值即可：
```cpp
    float r = texture_resource.Sample(sample_state, input.tex).r;
    float g = r;
    float b = r;
```

### utf-16 转 utf-32
因为vs工程中wstring是默认采用utf-16编码的，而FreeType需要的是utf-32编码。因此还需要一个转码的工作：
```cpp
        auto utf16_utf32 = [](const WORD* in, DWORD& utf32) -> UINT {
            if (!in || *in == 0) {
                utf32 = 0;
                return 0;
            }

            WORD w1 = in[0];
            UINT units = 0;

            if (w1 >= 0xD800 && w1 <= 0xDFFF) {
                if (w1 < 0xDC00) {
                    WORD w2 = in[1];
                    if (w2 >= 0xDC00 && w2 <= 0xDFFF) {
                        utf32 = (w2 & 0x03FF) + (((w1 & 0x03FF) + 0x40) << 10);
                        units = 2;
                    }
                }
                else {
                    //invalid in

                    utf32 = 0;
                    units = 0;
                }
            }
            else {
                utf32 = w1;
                units = 1;
            }

            return units;
        };
```

## 问题
到这里，很明显可以看到使用FreeType的问题，比如需要知道某个字体对应的字体文件路径，还要知道字符的utf-32编码，并且要一次一个地获取字形。一般我们都会想到用一个缓存来存已经获取的字形位图，或者把常用汉字
的位图先放到一个缓存中（我这里存在一个纹理资源中）。但是如果改变字体大小，就需要重新获取字形位图。这给缓存的方式也带来麻烦。因此，这种方式虽然被很多开发者推荐，但我觉得并不方便。后面想办法解决看看。
