# Table of contents

* [ReadMe](README.md)
* [前言](qian-yan.md)

## 第一部分：良好的代码 <a href="#good code" id="good code"></a>

* [第一章：安全性](di-yi-bu-fen-liang-hao-de-dai-ma/page-1/README.md)
  * [第1条：限制可变性](di-yi-bu-fen-liang-hao-de-dai-ma/page-1/di-1-tiao-xian-zhi-ke-bian-xing.md)
  * [第2条：最小化变量的作用域](di-yi-bu-fen-liang-hao-de-dai-ma/page-1/di-2-tiao-zui-xiao-hua-bian-liang-de-zuo-yong-yu.md)
  * [第3条：尽可能消除平台类型](<good code/page-1/di-3-tiao-jin-ke-neng-xiao-chu-ping-tai-lei-xing.md>)
  * [第4条： 不要暴露需要推断的类型](<good code/page-1/di-4-tiao-bu-yao-bao-lou-xu-yao-tui-duan-de-lei-xing.md>)
  * [第5条：指明你期望的参数和状态](<good code/page-1/di-5-tiao-zhi-ming-ni-qi-wang-de-can-shu-he-zhuang-tai.md>)
  * [第 6 条： 优先使用标准错误，而不是自定错误](<good code/page-1/di-6-tiao.md>)
  * [第7条：当返回结果可能缺失时，优先使 null 或 Failure](<good code/page-1/di-7-tiao-dang-fan-hui-jie-guo-ke-neng-que-shi-shi-you-xian-shi-null-huo-failure.md>)
  * [第8条：妥善处理空值](<good code/page-1/di-8-tiao-tuo-shan-chu-li-kong-zhi.md>)
  * [第9条： 使用 use 来关闭资源](<good code/page-1/di-9-tiao-shi-yong-use-lai-guan-bi-zi-yuan.md>)
  * [第10条：编写单元测试](<good code/page-1/di-10-tiao-bian-xie-dan-yuan-ce-shi.md>)
* [第二章：可读性](di-yi-bu-fen-liang-hao-de-dai-ma/di-er-zhang-ke-du-xing.md)
  * [第11条：为了可读性设计代码](<good code/di-er-zhang-ke-du-xing/di-11-tiao-wei-le-ke-du-xing-she-ji-dai-ma.md>)
  * [第12条：操作符的行为应该与其名称一致](<good code/di-er-zhang-ke-du-xing/di-12-tiao-cao-zuo-fu-de-hang-wei-ying-gai-yu-qi-ming-cheng-yi-zhi.md>)
  * [第13条：避免返回或操作 Unit?](<good code/di-er-zhang-ke-du-xing/di-13-tiao-bi-mian-fan-hui-huo-cao-zuo-unit.md>)
  * [第14条： 在变量不清晰时指定其类型](<good code/di-er-zhang-ke-du-xing/di-14-tiao-zai-bian-liang-bu-qing-xi-shi-zhi-ding-qi-lei-xing.md>)
  * [第15条：考虑显式引用接收者](<good code/di-er-zhang-ke-du-xing/page-2.md>)
  * [第16：属性应该代表状态，而非行为](<good code/di-er-zhang-ke-du-xing/di-16-shu-xing-ying-gai-dai-biao-zhuang-tai-er-fei-hang-wei.md>)
  * [第17条：考虑使用具名参数](<good code/di-er-zhang-ke-du-xing/di-17-tiao-kao-lv-shi-yong-ju-ming-can-shu.md>)
  * [第18条：遵守编程惯例](<good code/di-er-zhang-ke-du-xing/di-18-tiao-zun-shou-bian-cheng-guan-li.md>)

## 第二部分：良好的设计 <a href="#good design" id="good design"></a>

* [第三章：可重用性](<good design/di-san-zhang-ke-zhong-yong-xing/README.md>)
  * [第19条：不要重复知识](<good design/di-san-zhang-ke-zhong-yong-xing/di-19-tiao-bu-yao-zhong-fu-zhi-shi.md>)
  * [第20条：不要重复实现常用算法](<good design/di-san-zhang-ke-zhong-yong-xing/di-20-tiao-bu-yao-zhong-fu-shi-xian-chang-yong-suan-fa.md>)
  * [第21条](<good design/di-san-zhang-ke-zhong-yong-xing/di-21-tiao.md>)
  * [第22条：当实现公共算法时使用泛型](<good design/di-san-zhang-ke-zhong-yong-xing/di-22-tiao-dang-shi-xian-gong-gong-suan-fa-shi-shi-yong-fan-xing.md>)
  * [第23条：避免隐藏类型参数](<good design/di-san-zhang-ke-zhong-yong-xing/di-23-tiao-bi-mian-yin-cang-lei-xing-can-shu.md>)
  * [第24条：在使用泛型时考虑型变](<good design/di-san-zhang-ke-zhong-yong-xing/di-24-tiao-zai-shi-yong-fan-xing-shi-kao-lv-xing-bian.md>)
  * [第25条](<good design/di-san-zhang-ke-zhong-yong-xing/di-25-tiao.md>)
* [第四章](<good design/di-si-zhang.md>)
* [第五章](<good design/di-wu-zhang.md>)
* [第六章](<good design/di-liu-zhang.md>)

## 第三部分：性能 <a href="#performance" id="performance"></a>

* [第七章：](performance/di-qi-zhang.md)
* [第八章：高效的集合处理](performance/di-ba-zhang-gao-xiao-de-ji-he-chu-li/README.md)
  * [第49条：在具有多个处理步骤的大型集合上，优先使用 Sequence](performance/di-ba-zhang-gao-xiao-de-ji-he-chu-li/di-49-tiao-zai-ju-you-duo-ge-chu-li-bu-zhou-de-da-xing-ji-he-shang-you-xian-shi-yong-sequence.md)
  * [第50条：限制操作步骤的数量](performance/di-ba-zhang-gao-xiao-de-ji-he-chu-li/di-50-tiao-xian-zhi-cao-zuo-bu-zhou-de-shu-liang.md)
  * [第51条：性能关键处考虑使用原语的数组](performance/di-ba-zhang-gao-xiao-de-ji-he-chu-li/di-51-tiao-xing-neng-guan-jian-chu-kao-lv-shi-yong-yuan-yu-de-shu-zu.md)
  * [第52条：考虑使用可变集合](performance/di-ba-zhang-gao-xiao-de-ji-he-chu-li/di-52-tiao-kao-lv-shi-yong-ke-bian-ji-he.md)
