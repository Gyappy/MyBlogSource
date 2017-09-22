---
title: QRCode专题--编解码原理
date: 2017-09-21 00:50:33
tags: QRCode
---

![](http://jietu-10024907.file.myqcloud.com/ktarmrsfgrqdomiihyewsqoumwvzpsah.jpg)

## 目录
###### 背景与前言
###### 前置知识
###### encode
###### decode


## 背景与前言
  开篇先给看官道个歉，对不起，这是标题党，没有抉奥阐幽，也没有天地一指，万物一马的梗概与剖析。本文是笔者在维护公司二维码扫描库时，被叫成二维码master，但大都局限于应用层一些UI改动和接口调整，是兔丝燕麦般的master.于是不堪受辱，业余时间研究学习了二维码相关的知识，然后总结出的一些粗浅认识.顺便说一句,本文篇幅短小,看完它的时间只比你闭气不呼吸能坚持的时间稍微长一点。
  
  有任何问题和指导建议,可以联系我讨论:416676773@qq.com 或者 guchenhui1993@gmail.com
## 前置知识
* ### 不同码制:
  Data Matrix,MaxiCode, Aztec,QR Code, Vericode,PDF417,Ultracode,Code 49,Code 16K等
* ### 根据原理分类:
  堆叠式/行排式以及矩阵式。本文讲的是日常生活最常见的矩阵式QR Code.
  * #### 堆叠式/行排式:
  编码原理建立在一维条码基础之上，按需要堆积成二行或多行。代表的码制有:
  Code 16K、Code 49、PDF417、MicroPDF417等.
  * #### 矩阵式:
  在一个矩形空间通过黑、白像素在矩阵中的不同分布进行编码。有占位表示1,无占位表示0.代表的码制有:
  Code One、MaxiCode、QR Code、 Data Matrix、Han Xin Code、Grid Matrix等.
* ### 尺寸:
  二维码一共有40个尺寸(Version)。Version 1是21 x 21的矩阵，Version 2是 25 x25的矩阵，每增加一个version，就会增加4的尺寸，公式:(V-1)*4+21.
  
  看下图:
  ![](http://jietu-10024907.file.myqcloud.com/bdiwoekyagmtelmofdmivptfyoikhhnz.jpg)
  
  Version 40是这样的,真是令人害怕,来五个这样的二维码即可存下《浮生六记》的闺房记乐:
  ![](http://jietu-10024907.file.myqcloud.com/plfaamtpnacfrgkxagthkcrbgjqfujko.jpg)
  
* ### 信息容积:
  * 数字:最多7089字符 
  * 字母:最多4296字符
  * 8位二进制数:最多2953字符
  * 日文汉字:最多1817字符
* ### 格式信息:
  * 表示形式是V-E,V是前面提到的version,E是容错级别,分别是:L(7%),M(15%),Q(25%),R(35%).
* ### 容错：
  允许存储的二维码信息出现重复部分，级别越高，重复信息所占比例越高。
* ### 结构图:
![](http://jietu-10024907.file.myqcloud.com/uupujkplcrdtzajggopdevacojavigso.jpg)

  * Position Detection Patterns:
  
  三个相同的探测图形,用于定位视图和方向,位于左上角、右上角、左下角.符号中遇到类似图形的可能性极小,可以在有效识别到二维码。
  
  ![](http://jietu-10024907.file.myqcloud.com/tdutmwhypepjadbxeiiqpharbnecwosp.jpg)
  * Separators:
  
  每个探测图形和编码区域之间有一条1单位宽度的分隔符.由白色块组成.
  
  * Timing Pattern:
  
  黑白色相间交替组成的一行一列两条位于横纵的两两探测图形之间,用于确定符号的密度和版本,提供决定模块坐标的基准位置.
  
  * Alignment Patterns:
  
  类似小号的探测图形,中心矩形边框变为1单位,这种校正图形的数量由version来定,大于version 1的都有该校正图形.
  
  * Encoding region:
  
  包含数据码字，纠错码字，版本信息，格式信息，符号字符等数据。
  
  * Quiet zone:
  
  环绕整个二维码四周的4单位宽的区域.白色.

## encode

数据分析-->数据编码-->纠错编码-->构造最终数据-->构造矩阵-->打上掩码-->填充格式与版本信息

* ### 数据分析:
  分析输入的数据流,确定编码的字符类型，按相应的字符集转换成符号字符;选择纠错等级;如果没有指定version,会采用能表示目标信息的最小version.
  
  字符类型的编号(中文是1101,终止符是0000):
  
  ![](http://jietu-10024907.file.myqcloud.com/wbawqdwizpboivhxkuaihtfgniyzzdyw.jpg)
  
  各尺寸的数据组成细节:
  
  ![](http://jietu-10024907.file.myqcloud.com/jspjxvtdaasysuwybtymmtqaeulpkbxu.jpg)
  
* ### 数据编码: 
  将数据字符转换为位流，每8位一个码字，整体构成一个数据的码字序列.必要的时候需加入填充字符以填满按照版本要求的数据码字数.
  
  先看看不同version下数据长度对应的编码的位数约定:
  
  ![](http://jietu-10024907.file.myqcloud.com/vkjytdumvleazllykrqiznaacybbuuxz.jpg)
  支持如下编码:
  
  * ##### Numeric mode|数字编码:
  
    数据类型编码(4位)+数据长度编码(根据version转成10或12或14位二进制数)+从左到右,
    每3个数字为一组,转成10位二进制数,剩下不满三位的转成4或7位二进制数。
    
    比如1314520:
      * 131 452 0
      * --> 131(0010000011) 452(0111000100) 0(0000)
      * --> 数字长度7 -->0000000111
      * --> 由上面数据分析的图,数字的编码是0001
      * --> 0001 0000000111 0010000011 0111000100 0000 0000(结束符)
    
    公式B=4+C+10(D DIV 3)+R.
    
  * ##### Alphanumeriv mode|字符编码:0~9,A~Z,以及一些符号和空格。
  
    编码方式和数字编码类似,区别是以2个字符为一组,转成11位二进制数,不满2个字符的转成6位。(a,b)-->a*45+b-->binary
    
    公式B = 4 + C + 11(D DIV 2) + 6(D MOD 2).
  
  ![](http://jietu-10024907.file.myqcloud.com/qjyzvplczaelpabyobajdvdeuqdrilso.jpg)
  
  * ##### Byte mode|字节编码:
  
    公式B = 4 + C + 8D.
  
  ![](http://jietu-10024907.file.myqcloud.com/sawgmkdfbpnsfkdstcdxujkvxpefvrlv.jpg)
  
  * ##### Mixing mode|混合编码
    
    各种类型的组合编码。
   
  ![](http://jietu-10024907.file.myqcloud.com/evlvagdjgzfdtzlvqgivhwlxqzaylvvo.jpg)
   
  * Kanji mode|日文编码(不具体展开)
  * ECI mode|特殊字符集(不具体展开)
  * Structured Append mode|附加结构编码(不具体展开)
  * FNC1 mode|特殊工业或行业使用(不具体展开)
  
  
  8bit重排:接下来会对二进制比特流按照8位重排,编码总位数如果不是8的倍数,需要在末尾补0.

  补齐码:每个字码是8位,重排补0以后,如果字码的长度没有到达该version的容量,需要在后面加上补齐码。
  具体姿势是不断重复11101100和00010001这两个码字。
  
* ### 纠错编码
  为了能被纠错算法处理,将解析出的码字序列分块。然后把纠错码字拼到数据码字后面。有上文提到的L,M,Q,H四种纠错级别.
  * ##### 纠错算法:
  
    可纠正两种类型的错误,拒读错误(错误码字的位置已知)和替代错误(错误码字的位置未知).拒读错误是没有扫描到或者无法译码的字符,替代错误是错误译码的字符。一个替代错误需要两个纠错码字来纠正。
    
    这里有个公式：e + 2t <= d - p
    
    e:拒读错误数 t:替代错误数 d:纠错码字数 p:错误译码保护码字
    
    主要通过[里德-所罗门纠错算法](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction "里德-所罗门纠错算法") 来实现。
    涉及到伽罗瓦域中的多项式算法等数学理论。这边还需要深入学习。写个//TODO_NEXT_TIME
    
    官方的除法电路图示是这样的:
    
    ![](http://jietu-10024907.file.myqcloud.com/lqcuilltjvbmxvxqkvkolrouzixstnyi.jpg)
   
* ### 构造最终数据信息
  将数据码字的块和纠错码字的块以行列交织的方式排列。
  
  经过上一步计算出纠错码以后得到类似这样的结果:
  
  ![](http://jietu-10024907.file.myqcloud.com/grxflqfyxadcrmljazpkopfzbcmsigpj.jpg) 
  
  左边是数据码字,右边是纠错码字。然后按列排列成:
    D1, D12, D23, D35, D2, D13, D24, D36, ... D11, D22, D33, D45, D34, D46, E1, E23, E45, E67, E2, E24, E46, E68, ... E22, E44, E66, E88.
  
* ### 构造矩阵
  将码字模块以及探测图形、分隔符、定位图形、校正图形填充到矩阵相应位置中。具体位置如图:
  
  ![](http://jietu-10024907.file.myqcloud.com/uuswirjwkyczsrgkdbiyacqvrzwgclqy.jpg)
  
  ![](http://jietu-10024907.file.myqcloud.com/gcgepwotbrrboufztsdvtcroytbwdnag.jpg)
  
  以及校验图形的位置:
  
  ![](http://jietu-10024907.file.myqcloud.com/cjlhjlgarcltojtyifmswhguimxfoqms.jpg)
  
  QR Code中,符号字符以2个单位宽的纵列从符号的右下角开始布置,并且自右向左,自底向上。每个码字最高位应放在第一个可用的位置,中途遇到障碍就跳过,在障碍物上面或者下面继续填充。
  
  ![](http://jietu-10024907.file.myqcloud.com/fmkvtapizuuigitqakiwvtengsuldkrp.jpg)
  ![](http://jietu-10024907.file.myqcloud.com/hkujssasjqvcwqavutqwjlbwqfbbztjw.jpg)
  ![](http://jietu-10024907.file.myqcloud.com/pwxdkinbaoqyzvjkpfzqdrgurphdnuey.jpg)
* ### 打上掩码
  为了可靠性,使用掩码图形让矩阵中的黑白色块比例接近1:1并均匀分布,尽可能避免探测图形出现在别的区域.所以每个二维码咋一看,都是一模一样。
  
  有几点需要说明
  
  * 不会影响功能区(Function Patterns)
  * 对编码区域(Encoding Region)进行XOR运算(按照掩码的黑色部分颜色取反)
  * 对结果图不符合人意的部分计分
  * 评估后选择最佳的掩码(分数最低的那个)
  
  关于掩码方案优劣的评分标准(N1=3, N2=3, N3=40, N4=10):
  ![](http://jietu-10024907.file.myqcloud.com/ajvljthgqzwzvkrelrxvxrrmjzuzoowf.jpg)
  
  掩码标识码:
  
  ![](http://jietu-10024907.file.myqcloud.com/nqzzyyynuzzuxabnzyudevzappyficow.jpg)
  
  八种掩码:
  ![](http://jietu-10024907.file.myqcloud.com/kbdtvhrtjpfpphgphbelmfynwsppfttn.jpg)
  
  大致步骤如下:
  ![](http://jietu-10024907.file.myqcloud.com/eieoxszdetxpbvannhtydfwjxkcqbdbg.jpg)
  
* ### 填充格式与版本信息
  * 生成格式和版本信息填入矩阵相应的区域上。
    * 格式信息:格式信息15位,5个数据位10个纠错位。数据位中前两位是纠错等级:
    
    ![](http://jietu-10024907.file.myqcloud.com/cedzxtrqxehriviwawuqqcnivwgkxlwe.jpg)
    
    第三到第五位是掩码标识。然后根据纠错算法算出10位纠错码,加载五位数据位后面。
    * 版本信息:版本信息18位,6位数据位12位纠错位。version 7~40时才有版本信息,没有任何版本信息的结果全是0,不用对版本信息做掩码。
  
  
  ![](http://jietu-10024907.file.myqcloud.com/jtehimrjtfkdiwmpgcdsrxpbwbvgqxwd.jpg)
  
  ![](http://jietu-10024907.file.myqcloud.com/mvinyesqryqbyxwteywgieqmzkkozigb.jpg)
  
  ![](http://jietu-10024907.file.myqcloud.com/tjmfumatsahmqwfymxbzbjnypwwavnsk.jpg)


## decode

encode的逆向流程:

![](http://jietu-10024907.file.myqcloud.com/czwssktmwulmuxwelamqwrikffpobpff.jpg)

* 定位并获取图形,根据黑白色块转成0,1组成的数组,确定一个阈值,用该值将图像转化为一系列深色和浅色像素
* 识别格式信息
* 识别version信息
* 去掉掩码,即从格式信息中得到编码区的摆位图进行异或处理消除掩码
* 恢复数据码字和纠错码字
* 使用纠错码字进行错误检查,并纠错
* 解码数据码字

## 下节预告
* ### 里德-所罗门纠错算法
* ### zxing源码
