---
title: EasyExcel 简单的Excel 处理工具
tags:
  - Java
categories:
  - 技术
toc: false
date: 2020-08-14 14:32:43
---

> 项目里经常会用到报表导出功能或者导入功能，最近发现阿里也出了开源的Excel组件 EasyExcel，写一个DEMO尝试用下

### EasyExcel 
先附上 Github 开源地址 :https://github.com/alibaba/easyexcel，以maven项目为列引入
``` xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>easyexcel</artifactId>
  <version>${easyexcel.version}</version>
</dependency>
```

### 写 Excel 
``` java
/**
  * 最简单的读
  * <p>1. 创建excel对应的实体对象 参照{@link DemoData}
  * <p>2. 由于默认一行行的读取excel，所以需要创建excel一行一行的回调监听器，参照{@link DemoDataListener}
  * <p>3. 直接读即可
  */
@Test
public void simpleRead() {
   String fileName = TestFileUtil.getPath() + "demo" + File.separator + "demo.xlsx";
   // 这里 需要指定读用哪个class去读，然后读取第一个sheet 文件流会自动关闭
   EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).sheet().doRead();
}
```

``` java
@Data
public class DemoData {
    @ExcelProperty("字符串标题")
    private String string;
    @ExcelProperty("日期标题")
    private Date date;
    @ExcelProperty("数字标题")
    private Double doubleData;
    /**
     * 忽略这个字段
     */
    @ExcelIgnore
    private String ignore;
}
```

#### @ExcelProperty
|属性|类型|用处|
|-|-|-|
|value|String[]|表格中的列名，当为空时默认使用字段名，多个列名可以组合复杂的表头|
|index|int|写入表格中指定列|
|order|int|写入顺序，默认按照字段顺序|
|converter|Class<? extends Converter>|转化器用于对字段进行自定义处理，EasyExcel 提供了大量默认转化器|

#### @ContentFontStyle @HeadFontStyle
用于设置单元格和表头内容字体格式的注解
|参数|类型|含义|
|-|-|-|
|fontName|string|字体名称|
|fontHeightInPoints|short|字体高度|
|italic|boolean|是否斜体|
|strikeout|boolean|是否设置删除水平线|
|color|	short|字体颜色|
|typeOffset|short|偏移量|
|underline|byte|下划线|
|bold|boolean|是否加粗|
|charset|int|编码格式|

#### @ContentStyle @HeadStyle
用于设置单元格和表头样式注解
|参数|类型|含义|
|-|-|-|
|dataFormat|short|数据格式化|
|hidden|boolean|样式隐藏|
|locked|boolean|样式锁定|
|quotePrefix|boolean|在单元格前面增加`符号，数字或公式将以字符串形式展示|
|horizontalAlignment|HorizontalAlignment|设置是否水平居中|
|wrapped|boolean|设置文本是否应换行。将此标志设置为true通过在多行上显示使单元格中的所有内容可见|
|verticalAlignment|boolean|设置是否垂直居中|
|rotation|short|设置单元格中文本旋转角度。03版本的Excel旋转角度区间为-90°~90°，07版本的Excel旋转角度区间为0°~180°|
|indent|short|设置单元格中缩进文本的空格数|
|borderLeft|BorderStyle|设置左边框的样式|
|borderRight|BorderStyle|设置右边框样式|
|borderTop|BorderStyle|设置上边框样式|
|borderBottom|BorderStyle|设置下边框样式|
|leftBorderColor|short|设置左边框颜色|
|rightBorderColor|short|设置右边框颜色|
|topBorderColor|short|设置上边框颜色|
|bottomBorderColor|short|设置下边框颜色|
|fillPatternType|FillPatternType|设置填充类型|
|fillBackgroundColor|short|设置背景色|
|fillForegroundColor|short|设置前景色|
|shrinkToFit|boolean|设置自动单元格自动大小|

#### @ContentRowHeight @HeadRowHeight
用于设置单元和和表头行高注解
|参数|类型|含义|
|-|-|-|
|value|short|行高|

#### @ColumnWidth 
用于设置单元格的列宽，作用类属于全局，当某字段作用覆盖全局设置，最大值为255
|参数|类型|含义|
|-|-|-|
|value|short|列宽|

#### @ExcelIgnore
用于屏蔽不需要写入Excel的字段标记
