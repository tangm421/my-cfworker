# 1. 记BFCP解析库confiance-v7bfcp-only-05中踩过的坑

在开发sip呼叫辅流的过程中使用了一个bfcp的解析库：
[confiance-v7bfcp-only-05](https://sourceforge.net/projects/confiance/)。

然而在使用过程中发现这个解析库的一些缺陷，列举如下：

## 1.1. `SUPPORTED-PRIMITIVES` 和 `SUPPORTED-ATTRIBUTES` 的字节长度问题
原始库中对这两个字段的处理都是采用的`unsigned short int`存储到网络交互字节中，
根据  
[RFC4582 5.2.10.  SUPPORTED-ATTRIBUTES](https://tools.ietf.org/html/rfc4582#section-5.2.10) 和  
[RFC4582 5.2.11.  SUPPORTED-PRIMITIVES](https://tools.ietf.org/html/rfc4582#section-5.2.11) 的描述，这两个字段都是占用1个字节的。

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |0 0 0 1 0 1 0|M|    Length     | Supp. Attr. |R| Supp. Attr. |R|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     | Supp. Attr. |R| Supp. Attr. |R| Supp. Attr. |R| Supp. Attr. |R|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                                                               |
     /                                                               /
     /                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                               |            Padding            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                  Figure 17: SUPPORTED-ATTRIBUTES format

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |0 0 0 1 0 1 1|M|    Length     |   Primitive   |   Primitive   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |   Primitive   |   Primitive   |   Primitive   |   Primitive   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                                                               |
     /                                                               /
     /                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                               |            Padding            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                  Figure 18: SUPPORTED-PRIMITIVES format

## 1.2. Payload Length 的字节长度及单位问题

根据 [RFC4582 5.1.  COMMON-HEADER Format](https://tools.ietf.org/html/rfc4582#section-5.1) 描述
> Payload Length: This 16-bit field contains the length of the message in 4-octet units, excluding the common header

由于bfcp要求网络交互字节块必须要以4字节对齐，凡是不足4字节的都需要填充。因此称为`in 4-octet units`，

*`octet` 是指八个比特（bit）为一组的单位，通常是指一个字节（byte）的意思。*

也就是说使用抓包工具查看bfcp的package时，Payload Length字段显示的值应该为整个BFCP包的长度减去BFCP通用头长度再除以4（即 `(BFCPPackageLength - 12) / 4`）；相反的，在解析bfcp的package时也要根据Payload Length字段的值乘以4再加上通用头长度即为整个包的长度（即 `PayloadLength * 4 + 12`） ，且应该对负载长度以4求余来校验字节块是否满足这一条件。

但原始库中对于这块的处理是有问题的，包括判断逻辑也有问题。

## 1.3. 关于va_arg的错误使用
在修改中引入了的这个问题，linux上使用解析库时引发异常

在使用 `va_arg` 时传入的可变参数会经历 `默认参数提升` 的隐式转换过程，因此在使用过程中ap的下一个参数类型不应该指定为以下类型：

`char` / `signed char` / `unsigned char`  
`short` / `signed short` / `unsigned short`  
`short int` / `signed short int` / `unsigned short int`

有关c语言 [默认参数提升](https://zh.cppreference.com/w/c/language/conversion#.E9.BB.98.E8.AE.A4.E5.8F.82.E6.95.B0.E6.8F.90.E5.8D.87) 的规则描述如下：

> **默认参数提升**  
  在函数调用表达式中，当调用下列函数时  
    &emsp;&emsp;1) 无原型函数  
    &emsp;&emsp;2) 变参数函数，其中参数表达式是匹配省略号参数的尾随参数之一  
  每个整数类型的参数都会经历整数提升（见后述），而每个 `float` 类型参数都隐式转换为 `double` 类型

> **整数提升**  
整数提升是任何等级小于或等于 `int` 等级的整数类型，或是 _Bool 、 signed int 、 unsigned int 类型的位域类型的值到 `int` 或 `unsigned int` 类型值的隐式转换。  
若 `int` 能表示原类型的整个值域（或原位域的值域），则值转换成 `int` 类型。否则值转化成 `unsigned int` 类型。  
整数提升保持值，包含符号：

更详细的信息可参考：[可变长参数列表误区与陷阱——va_arg不可接受的类型](http://www.cppblog.com/ownwaterloo/archive/2009/04/21/unacceptable_type_in_va_arg.html)


## 1.4. 修改清单
详情参考：[fix confiance-v7bfcp-only-05](https://github.com/tangm421/confiance-v7bfcp-only-05/commit/ddb84a2a827dac6027a034b8fe10f6b4b64086b0)

左侧文件： 原始文件

右侧文件： 修改后


### 1.4.1. bfcp_messages.h

<table class="fc" cellspacing="0" cellpadding="0">
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">111</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">typedef</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span> <span class="TextSegElement_230_179_168_233_135_138">/* FLOOR-ID list, to manage the multiple FLOOR-ID attributes */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">111</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">typedef</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span> <span class="TextSegElement_230_179_168_233_135_138">/* FLOOR-ID list, to manage the multiple FLOOR-ID attributes */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">112</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ID</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* FLOOR-ID */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">112</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ID</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* FLOOR-ID */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">113</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Pointer to next FLOOR-ID instance */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">113</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Pointer to next FLOOR-ID instance */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">114</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">114</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">115</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">115</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">116</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">typedef</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* list to manage all the supported attributes and primitives */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">116</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">typedef</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* list to manage all the supported attributes and primitives */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">117</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; unsigned <span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span> element;&nbsp; &nbsp;&nbsp; /* Element (Attribute/Primitive) */</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">117</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; unsigned <span class="TextSegSigDiff">char</span> element;&nbsp; &nbsp; &nbsp; /* Element (Attribute/Primitive) */</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">118</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Pointer to next supported element instance */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">118</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Pointer to next supported element instance */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">119</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">119</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">120</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">120</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">121</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">typedef</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_request_status</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">121</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">typedef</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">struct</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_request_status</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">122</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">rs</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Request Status */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">122</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">rs</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Request Status */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">123</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">qp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Queue Position */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">123</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">qp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Queue Position */</span></td>
</tr>
<tr class="SectionGap"><td colspan="5">&nbsp;</td></tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">243</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Add IDs to an existing Floor ID list (last argument MUST be 0) */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">243</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Add IDs to an existing Floor ID list (last argument MUST be 0) */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">244</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_add_floor_id_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">fID</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">.</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">.</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">.</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">244</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_add_floor_id_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">fID</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">.</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">.</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">.</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">245</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Floor ID list */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">245</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Floor ID list */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">246</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_free_floor_id_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">246</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_free_floor_id_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_floor_id_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">247</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">247</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">248</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Create a new Supported (Primitives/Attributes) list (last argument MUST be 0) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">248</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Create a new Supported (Primitives/Attributes) list (last argument MUST be 0) */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">249</td>
<td class="TextItemSigDiffMod">bfcp_supported_list *bfcp_new_supported_list(unsigned <span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span> element, ...);</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">249</td>
<td class="TextItemSigDiffMod">bfcp_supported_list *bfcp_new_supported_list(unsigned <span class="TextSegSigDiff">char</span> element, ...);</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">250</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Supported (Primitives/Attributes) list */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">250</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Supported (Primitives/Attributes) list */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">251</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_free_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">251</td>
<td class="TextItemSame"><span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_free_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">252</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">252</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">253</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Create a New Request Status (RequestStatus/QueuePosition) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">253</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Create a New Request Status (RequestStatus/QueuePosition) */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">254</td>
<td class="TextItemSame"><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_request_status</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_new_request_status</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">rs</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">qp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">254</td>
<td class="TextItemSame"><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_request_status</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_new_request_status</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">rs</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">qp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">255</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Request Status (RequestStatus/QueuePosition) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">255</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Request Status (RequestStatus/QueuePosition) */</span></td>
</tr>
</table>

### 1.4.2. bfcp_messages.c

<table class="fc" cellspacing="0" cellpadding="0">
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">195</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">195</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">196</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">196</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">197</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">197</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">198</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">198</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">199</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">199</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">200</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Create a new Supported (Primitives/Attributes) list (last argument MUST be 0) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">200</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Create a new Supported (Primitives/Attributes) list (last argument MUST be 0) */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">201</td>
<td class="TextItemSigDiffMod">bfcp_supported_list *bfcp_new_supported_list(unsigned <span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span> element, ...)</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">201</td>
<td class="TextItemSigDiffMod">bfcp_supported_list *bfcp_new_supported_list(unsigned <span class="TextSegSigDiff">char</span> element, ...)</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">202</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">202</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">203</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">203</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">204</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">va_list</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ap</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">204</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">va_list</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ap</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">205</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">va_start</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ap</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">205</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">va_start</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ap</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">206</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">206</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">207</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We could not allocate the memory, return a with failure */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">207</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We could not allocate the memory, return a with failure */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">208</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">208</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">209</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">209</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">210</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">210</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">211</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; element = va_arg(ap, int);</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">211</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; element = <span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">char</span><span class="TextSegSigDiff">)</span>va_arg(ap, int);</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">212</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">while</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">212</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">while</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">213</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">213</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">214</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We could not allocate the memory, return a with failure */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">214</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We could not allocate the memory, return a with failure */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">215</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">215</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">216</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">216</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">element</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">217</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">217</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">218</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">218</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">219</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; element = va_arg(ap, int);</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">219</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; element = <span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">char</span><span class="TextSegSigDiff">)</span>va_arg(ap, int);</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">220</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">220</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">221</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">va_end</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ap</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">221</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">va_end</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ap</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">222</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">222</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">223</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">223</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">224</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">224</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">225</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Supported (Primitives/Attributes) list */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">225</td>
<td class="TextItemSame"><span class="TextSegElement_230_179_168_233_135_138">/* Free a Supported (Primitives/Attributes) list */</span></td>
</tr>
</table>

### 1.4.3. bfcp_messages_build.c

<table class="fc" cellspacing="0" cellpadding="0">
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">45</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* 32 bits */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">45</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* 32 bits */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">46</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch16</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* 16 bits */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">46</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">short</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch16</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* 16 bits */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">47</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">47</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">48</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0xE0000000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">|</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span> <span class="TextSegElement_230_149_176_229_173_151">29</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span>&nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* First the Version (3 bits, set to 001) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">48</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0xE0000000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">|</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span> <span class="TextSegElement_230_149_176_229_173_151">29</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span>&nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* First the Version (3 bits, set to 001) */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">49</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0x1F000000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">|</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span> <span class="TextSegElement_230_149_176_229_173_151">24</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span>&nbsp; &nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* then the Reserved (5 bits, ignored) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">49</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0x1F000000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">|</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span> <span class="TextSegElement_230_149_176_229_173_151">24</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span>&nbsp; &nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* then the Reserved (5 bits, ignored) */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">50</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0x00FF0000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">|</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitive</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span> <span class="TextSegElement_230_149_176_229_173_151">16</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* the Primitive (8 bits) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">50</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">0x00FF0000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">|</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitive</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span> <span class="TextSegElement_230_149_176_229_173_151">16</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* the Primitive (8 bits) */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">51</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; ((ch32 &amp; !(0x0000FFFF)) | (message-&gt;length - 12));&nbsp; /* and the payload length (16 bits) */</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">51</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; ((ch32 &amp; !(0x0000FFFF)) | (<span class="TextSegSigDiff">(</span>message-&gt;length - 12<span class="TextSegSigDiff">)</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">/</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">4</span>));&nbsp; &nbsp; /* and the payload length (16 bits)<span class="TextSegInsigDiff">, contains the length of the message in 4-octet units</span> */</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">52</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">htonl</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We want all protocol values in network-byte-order */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">52</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">htonl</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We want all protocol values in network-byte-order */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">53</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memcpy</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">53</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memcpy</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">54</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">54</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">55</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">htonl</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">entity</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">conferenceID</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">55</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">htonl</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">entity</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">conferenceID</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">56</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memcpy</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">56</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memcpy</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">57</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">57</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionGap"><td colspan="5">&nbsp;</td></tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">497</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attributes</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The supported attributes list is empty, return with a failure */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">497</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attributes</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The supported attributes list is empty, return with a failure */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">498</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">498</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">499</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* The Lenght of the attribute (starting from the TLV) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">499</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* The Lenght of the attribute (starting from the TLV) */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">500</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Number of bytes of padding */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">500</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Number of bytes of padding */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">501</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We keep track of where the TLV will have to be */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">501</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We keep track of where the TLV will have to be */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">502</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We skip the TLV bytes */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">502</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We skip the TLV bytes */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">503</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">;</span><span class="TextSegInsigDiff">&nbsp; &nbsp; </span><span class="TextSegInsigDiff">/* 16 bits */</span></td>
<td class="AlignCenter">+-</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">504</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">attributes</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">503</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">attributes</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">505</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">while</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Fill all supported attributes */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">504</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">while</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Fill all supported attributes */</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">506</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span>ch<span class="TextSegSigDiff">16</span> = <span class="TextSegSigDiff">htons</span><span class="TextSegSigDiff">(</span>temp-&gt;element<span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">505</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">char</span><span class="TextSegInsigDiff"> </span>ch = temp-&gt;element;</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">507</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span><span class="TextSegSigDiff">memcpy</span><span class="TextSegSigDiff">(</span>buffer<span class="TextSegSigDiff">,</span> <span class="TextSegSigDiff">&amp;</span>ch<span class="TextSegSigDiff">16</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">2</span><span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">506</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span>buffer<span class="TextSegSigDiff">[</span><span class="TextSegSigDiff">0</span><span class="TextSegSigDiff">]</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">=</span> ch<span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">&lt;&lt;</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">1</span>;</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">508</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; buffer = buffer+<span class="TextSegSigDiff">2</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">507</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; buffer = buffer+<span class="TextSegSigDiff">1</span>;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">509</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; attrlen = attrlen+<span class="TextSegSigDiff">2</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">508</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; attrlen = attrlen+<span class="TextSegSigDiff">1</span>;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">510</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">509</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">511</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">510</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">512</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We need padding */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">511</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We need padding */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">513</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">512</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">514</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memset</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">513</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memset</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">515</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">514</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionGap"><td colspan="5">&nbsp;</td></tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">524</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitives</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The supported attributes list is empty, return with a failure */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">523</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitives</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The supported attributes list is empty, return with a failure */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">525</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">524</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">526</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* The Lenght of the attribute (starting from the TLV) */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">525</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* The Lenght of the attribute (starting from the TLV) */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">527</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Number of bytes of padding */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">526</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Number of bytes of padding */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">528</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We keep track of where the TLV will have to be */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">527</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We keep track of where the TLV will have to be */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">529</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We skip the TLV bytes */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">528</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We skip the TLV bytes */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">530</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">;</span><span class="TextSegInsigDiff">&nbsp; &nbsp; </span><span class="TextSegInsigDiff">/* 16 bits */</span></td>
<td class="AlignCenter">+-</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">531</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitives</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">529</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitives</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">532</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">while</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Fill all supported primitives */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">530</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">while</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Fill all supported primitives */</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">533</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span>ch<span class="TextSegSigDiff">16</span> = <span class="TextSegSigDiff">htons</span><span class="TextSegSigDiff">(</span>temp-&gt;element<span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">531</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">char</span><span class="TextSegInsigDiff"> </span>ch = temp-&gt;element;</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">534</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span>memcpy(buffer, &amp;ch<span class="TextSegSigDiff">16</span>, <span class="TextSegSigDiff">2</span>);</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">532</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; </span>memcpy(buffer, &amp;ch, <span class="TextSegSigDiff">1</span>);</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">535</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; buffer = buffer+<span class="TextSegSigDiff">2</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">533</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; buffer = buffer+<span class="TextSegSigDiff">1</span>;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">536</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; attrlen = attrlen+<span class="TextSegSigDiff">2</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">534</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; attrlen = attrlen+<span class="TextSegSigDiff">1</span>;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">537</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">535</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">temp</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">538</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">536</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">539</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We need padding */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">537</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* We need padding */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">540</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">538</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">attrlen</span>%<span class="TextSegElement_230_149_176_229_173_151">4</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">541</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memset</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">539</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">memset</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">padding</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">542</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">540</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
</table>

### 1.4.4. bfcp_messages_parse.c

<table class="fc" cellspacing="0" cellpadding="0">
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">190</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">reserved</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Reserved bits are not 0, return with an error */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">190</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">reserved</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Reserved bits are not 0, return with an error */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">191</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_message_add_error</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">BFCP_RESERVED_NOT_ZERO</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">191</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_message_add_error</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">BFCP_RESERVED_NOT_ZERO</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">192</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">192</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">193</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred while recording the error, return with failure */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">193</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred while recording the error, return with failure */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">194</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">194</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">195</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitive</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_230_149_176_229_173_151">0x00FF0000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span> <span class="TextSegElement_230_149_176_229_173_151">16</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span> <span class="TextSegElement_230_179_168_233_135_138">/* Primitive identifier */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">195</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">primitive</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">ch32</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&amp;</span> <span class="TextSegElement_230_149_176_229_173_151">0x00FF0000</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span> <span class="TextSegElement_230_149_176_229_173_151">16</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span> <span class="TextSegElement_230_179_168_233_135_138">/* Primitive identifier */</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">196</td>
<td class="TextItemInsigRightMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegInsigDiff">/* cause the Payload Lenght contains the length of the message in 4-octet units, here we need to multiple with 4 */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">196</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; recvM-&gt;length = (ch32 &amp; 0x0000FFFF) + 12;&nbsp;&nbsp; /* Payload Lenght of the message + 12 (Common Header) */</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">197</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; recvM-&gt;length = (ch32 &amp; 0x0000FFFF) <span class="TextSegSigDiff">*</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">4</span><span class="TextSegInsigDiff"> </span>+ 12;&nbsp;&nbsp; /* Payload Lenght of the message + 12 (Common Header) */</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">197</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; if((<span class="TextSegSigDiff">(</span>recvM-&gt;length<span class="TextSegSigDiff">)</span> != message-&gt;length) || <span class="TextSegSigDiff">!</span>(recvM-&gt;length<span class="TextSegSigDiff">)</span>%4) {&nbsp; &nbsp; /* The message length is wrong */</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">198</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; if((recvM-&gt;length != message-&gt;length) || (recvM-&gt;length<span class="TextSegInsigDiff"> </span>%<span class="TextSegInsigDiff"> </span>4<span class="TextSegSigDiff">)</span>) { /* The message length is wrong */</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">198</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Either the length in the header is different from the length of the buffer... */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">199</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Either the length in the header is different from the length of the buffer... */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">199</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/*&nbsp;&nbsp; ...or the length is not a multiple of 4, meaning it's surely not aligned */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">200</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/*&nbsp;&nbsp; ...or the length is not a multiple of 4, meaning it's surely not aligned */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">200</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_message_add_error</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">BFCP_WRONG_LENGTH</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">201</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_message_add_error</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_149_176_229_173_151">0</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">BFCP_WRONG_LENGTH</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">201</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">202</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvM</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">errors</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">202</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred while recording the error, return with failure */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">203</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred while recording the error, return with failure */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">203</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">204</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionGap"><td colspan="5">&nbsp;</td></tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">621</td>
<td class="TextItemSame"><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_parse_attribute_SUPPORTED_ATTRIBUTES</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_message</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_attribute</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">622</td>
<td class="TextItemSame"><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_parse_attribute_SUPPORTED_ATTRIBUTES</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_message</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_attribute</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">622</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">623</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">623</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">length</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_149_176_229_173_151">3</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The length of this attribute is wrong */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">624</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">length</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_149_176_229_173_151">3</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The length of this attribute is wrong */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">624</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">625</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">625</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">626</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">626</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">627</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">627</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">;</span><span class="TextSegInsigDiff">&nbsp; &nbsp; </span><span class="TextSegInsigDiff">/* 16 bits */</span></td>
<td class="AlignCenter">+-</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">628</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Skip the Header */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">628</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Skip the Header */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">629</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; int number = (recvA-&gt;length-2)<span class="TextSegSigDiff">/</span><span class="TextSegSigDiff">2</span>;&nbsp;&nbsp; /* Each supported attribute takes <span class="TextSegInsigDiff">2</span> bytes */</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">629</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; int number = (recvA-&gt;length-2); /* Each supported attribute takes <span class="TextSegInsigDiff">1</span> bytes */</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">630</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">630</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">631</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* No supported attributes? */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">631</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* No supported attributes? */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">632</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">632</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">633</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">633</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">634</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">634</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">635</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegSigDiff">memcpy</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">&amp;</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">buffer</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">2</span><span class="TextSegSigDiff">)</span><span class="TextSegSigDiff">;</span></td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">636</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span>first-&gt;element = <span class="TextSegSigDiff">ntohs</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">635</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span>first-&gt;element = <span class="TextSegSigDiff">buffer</span><span class="TextSegSigDiff">[</span><span class="TextSegSigDiff">0</span><span class="TextSegSigDiff">]</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">&amp;</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">0xFE</span>;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">637</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">636</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">638</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Let's parse each other supported attribute we find */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">637</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Let's parse each other supported attribute we find */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">639</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">for</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">638</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">for</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">640</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">639</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">641</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">640</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">642</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">641</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">643</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>buffer = buffer+<span class="TextSegSigDiff">2</span>;&nbsp; /* Skip to the next supported attribute */</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">642</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>buffer = buffer<span class="TextSegInsigDiff"> </span>+<span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">1</span>;&nbsp; &nbsp; /* Skip to the next supported attribute */</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">644</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>me<span class="TextSegSigDiff">mcpy</span><span class="TextSegSigDiff">(&amp;</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">,</span> buffer<span class="TextSegSigDiff">,</span> <span class="TextSegSigDiff">2</span><span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">643</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span><span class="TextSegSigDiff">next</span><span class="TextSegSigDiff">-&gt;</span><span class="TextSegSigDiff">ele</span>me<span class="TextSegSigDiff">nt</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">=</span> buffer<span class="TextSegSigDiff">[</span><span class="TextSegSigDiff">0</span><span class="TextSegSigDiff">]</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">&amp;</span> <span class="TextSegSigDiff">0xFE</span>;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">645</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span><span class="TextSegSigDiff">next</span><span class="TextSegSigDiff">-</span><span class="TextSegSigDiff">&gt;</span><span class="TextSegSigDiff">element</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">=</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">ntohs</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">)</span><span class="TextSegSigDiff">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">646</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">644</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">647</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">645</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">648</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">646</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">649</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">647</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">650</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">648</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">651</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">649</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionGap"><td colspan="5">&nbsp;</td></tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">653</td>
<td class="TextItemSame"><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_parse_attribute_SUPPORTED_PRIMITIVES</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_message</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_attribute</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">651</td>
<td class="TextItemSame"><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_parse_attribute_SUPPORTED_PRIMITIVES</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_message</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_received_attribute</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">654</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">652</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">655</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">length</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_149_176_229_173_151">3</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The length of this attribute is wrong */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">653</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">length</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_149_176_229_173_151">3</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_230_179_168_233_135_138">/* The length of this attribute is wrong */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">656</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">654</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">657</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">655</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">int</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">658</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">656</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">659</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegSigDiff">unsigned</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">short</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">int</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">;</span><span class="TextSegInsigDiff">&nbsp; &nbsp; </span><span class="TextSegInsigDiff">/* 16 bits */</span></td>
<td class="AlignCenter">+-</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">660</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Skip the Header */</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">657</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">unsigned</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">char</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">*</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">message</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">buffer</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">recvA</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">position</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_230_149_176_229_173_151">2</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Skip the Header */</span></td>
</tr>
<tr class="SectionAll">
<td class="TextItemNum AlignRight">661</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; int number = (recvA-&gt;length-2)<span class="TextSegSigDiff">/</span><span class="TextSegSigDiff">2</span>;&nbsp;&nbsp; /* Each supported primitive takes <span class="TextSegInsigDiff">2</span> bytes */</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">658</td>
<td class="TextItemSigDiffMod">&nbsp;&nbsp;&nbsp; int number = (recvA-&gt;length-2); /* Each supported primitive takes <span class="TextSegInsigDiff">1</span> bytes */</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">662</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">659</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">663</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* No supported primitives? */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">660</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span>&nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* No supported primitives? */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">664</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">661</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">665</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">662</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">666</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">663</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">667</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span><span class="TextSegSigDiff">memcpy</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">&amp;</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">buffer</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">2</span><span class="TextSegSigDiff">)</span><span class="TextSegSigDiff">;</span></td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">668</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span>first-&gt;element = <span class="TextSegSigDiff">ntohs</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">664</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp;&nbsp; </span>first-&gt;element = <span class="TextSegSigDiff">buffer</span><span class="TextSegSigDiff">[</span><span class="TextSegSigDiff">0</span><span class="TextSegSigDiff">]</span>;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">669</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">665</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">670</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Let's parse each other supported primitive we find */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">666</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span>&nbsp; &nbsp; &nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* Let's parse each other supported primitive we find */</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">671</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">for</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">667</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">for</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&lt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">number</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">i</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">+</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">{</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">672</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">668</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">calloc</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_149_176_229_173_151">1</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">,</span> <span class="TextSegElement_229_133_179_233_148_174_229_173_151">sizeof</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">bfcp_supported_list</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">673</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">669</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">if</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">(</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">!</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">)</span>&nbsp;&nbsp; <span class="TextSegElement_230_179_168_233_135_138">/* An error occurred in creating a new Supported Attributes list */</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">674</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">670</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">NULL</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">675</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>buffer = buffer+<span class="TextSegSigDiff">2</span>;&nbsp; /* Skip to the next supported primitive */</td>
<td class="AlignCenter">&lt;&gt;</td>
<td class="TextItemNum AlignRight">671</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>buffer = buffer<span class="TextSegInsigDiff"> </span>+<span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">1</span>;&nbsp; &nbsp; /* Skip to the next supported primitive */</td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">676</td>
<td class="TextItemSigLeftMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span><span class="TextSegSigDiff">memcpy</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">&amp;</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">buffer</span><span class="TextSegSigDiff">,</span><span class="TextSegInsigDiff"> </span><span class="TextSegSigDiff">2</span><span class="TextSegSigDiff">)</span><span class="TextSegSigDiff">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">&nbsp;</td>
<td class="TextItemSame">&nbsp;</td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">677</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>next-&gt;element = <span class="TextSegSigDiff">ntohs</span><span class="TextSegSigDiff">(</span><span class="TextSegSigDiff">ch16</span><span class="TextSegSigDiff">)</span>;</td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">672</td>
<td class="TextItemSigDiffMod"><span class="TextSegInsigDiff">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; </span>next-&gt;element = <span class="TextSegSigDiff">buffer</span><span class="TextSegSigDiff">[</span><span class="TextSegSigDiff">0</span><span class="TextSegSigDiff">]</span>;</td>
</tr>
<tr class="SectionBegin">
<td class="TextItemNum AlignRight">678</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">=</td>
<td class="TextItemNum AlignRight">673</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">-</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">&gt;</span><span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">679</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">674</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_230_160_135_232_175_134_231_172_166">previous</span> <span class="TextSegElement_232_191_144_231_174_151_231_172_166">=</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">next</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">680</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">675</td>
<td class="TextItemSame">&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">681</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">676</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
<tr class="SectionMiddle">
<td class="TextItemNum AlignRight">682</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">677</td>
<td class="TextItemSame">&nbsp;&nbsp;&nbsp; <span class="TextSegElement_229_133_179_233_148_174_229_173_151">return</span> <span class="TextSegElement_230_160_135_232_175_134_231_172_166">first</span><span class="TextSegElement_232_191_144_231_174_151_231_172_166">;</span></td>
</tr>
<tr class="SectionEnd">
<td class="TextItemNum AlignRight">683</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
<td class="AlignCenter">&nbsp;</td>
<td class="TextItemNum AlignRight">678</td>
<td class="TextItemSame"><span class="TextSegElement_232_191_144_231_174_151_231_172_166">}</span></td>
</tr>
</table>


<style>
.AlignLeft { text-align: left; }
.AlignCenter { text-align: center; }
.AlignRight { text-align: right; }
body { font-family: sans-serif; font-size: 11pt; }
td { vertical-align: top; padding-left: 4px; padding-right: 4px; }

tr.SectionGap td { font-size: 4px; border-left: none; border-top: none; border-bottom: 1px solid Black; border-right: 1px solid Black; }
tr.SectionAll td { border-left: none; border-top: none; border-bottom: 1px solid Black; border-right: 1px solid Black; }
tr.SectionBegin td { border-left: none; border-top: none; border-right: 1px solid Black; }
tr.SectionEnd td { border-left: none; border-top: none; border-bottom: 1px solid Black; border-right: 1px solid Black; }
tr.SectionMiddle td { border-left: none; border-top: none; border-right: 1px solid Black; }
tr.SubsectionAll td { border-left: none; border-top: none; border-bottom: 1px solid Gray; border-right: 1px solid Black; }
tr.SubsectionEnd td { border-left: none; border-top: none; border-bottom: 1px solid Gray; border-right: 1px solid Black; }
table.fc { border-top: 1px solid Black; border-left: 1px solid Black; width: 100%; font-family: monospace; font-size: 10pt; }
td.TextItemInsigAdd { color: #000000; background-color: #EFEFFF; }
td.TextItemInsigDel { color: #000000; background-color: #EFEFFF; text-decoration: line-through; }
td.TextItemInsigDiffMod { color: #000000; background-color: #EFEFFF; }
td.TextItemInsigLeftMod { color: #000000; background-color: #EFEFFF; }
td.TextItemInsigRightMod { color: #000000; background-color: #EFEFFF; }
td.TextItemNum { color: #827357; background-color: #F2F2F2; }
td.TextItemSame { color: #000000; background-color: #FFFFFF; }
td.TextItemSigAdd { color: #000000; background-color: #FFE3E3; }
td.TextItemSigDel { color: #000000; background-color: #FFE3E3; text-decoration: line-through; }
td.TextItemSigDiffMod { color: #000000; background-color: #FFE3E3; }
td.TextItemSigLeftMod { color: #000000; background-color: #FFE3E3; }
td.TextItemSigRightMod { color: #000000; background-color: #FFE3E3; }
.TextSegInsigDiff { color: #0000FF; }
.TextSegReplacedDiff { color: #0000FF; font-style: italic; }
.TextSegSigDiff { color: #FF0000; }
.TextSegElement_229_133_179_233_148_174_229_173_151 { font-weight: bold; }
.TextSegElement_230_160_135_232_175_134_231_172_166 { }
.TextSegElement_230_149_176_229_173_151 { color: #2E9269; }
.TextSegElement_230_179_168_233_135_138 { color: #786A41; }
.TextSegElement_232_191_144_231_174_151_231_172_166 { }
</style>