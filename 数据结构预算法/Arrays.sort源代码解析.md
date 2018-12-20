**Java Arrays中提供了对所有类型的排序。其中主要分为Primitive(8种基本类型)和Object两大类。**

**基本类型：采用调优的快速排序；**

**对象类型：采用改进的归并排序。**

## 一、对于基本类型源码分析如下（以int[]为例）：

　　**Java对Primitive（int，float等原型数据）数组采用快速排序，对Object对象数组采用归并排序。****对这一区别，sun在<<The Java Tutorial>>中做出的解释如下：**





https://www.cnblogs.com/gw811/archive/2012/10/04/2711746.html