---
layout: post
title: 拷贝Source Insight中的彩色代码的方法
tags: SourceInsight CodeColor
---

拷贝Source Insight中的彩色代码的方法
---

将Source Insight 的代码着色拷贝出来(到Word里面) /copy the colorful code in Source Insight

the only method is

1. open the file with Source Insight

2. make sure you have installed a PDF maker tool ,such as Adobe Acrobat 7.0 Professional or JAWS PDF Creator or PDF Factory ,then you can get a virtual PDF Printer . ^_^

so ,Select File -> Print ,then select the "Adobe PDF" as the printer ,click OK .then you will get a PDF format file with the colorful code ! so you can copy them as you want ! ^_^

but in the end ,I find some problem of this method.

the outcome is that we can see colorful code in PDF file,but when we copy them to other place ,such as Word or the current clipboard ,the color is disapear !!! feel sad ….

【将Source Insight里面着色代码拷贝到Word里面的方法】

其实方法很简单，我的是英文版的Source Insight，

用Source Insight打开文件后，去File->Print，然后在 常规->选择打印机中，选择“Foxit Reader PDF Printer”，然后就可以确定，就可以输出一个pdf文件了，比如此处的part_dos.c变成了part_dos.pdf,

然后，去pdf文件里面复制代码，粘贴到word里面，就是可以保留颜色的代码了。

原文链接：
http://www.crifan.com/source_insight_copy_the_color_code_in_the_method_%7C_source_insight_will_copy_the_color_code_out_to_the_word_inside_approach/