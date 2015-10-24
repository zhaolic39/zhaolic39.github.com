title: Office中数学公式用Java解析
comments: true
categories: java
date: 2015-10-24 11:10:06
tags:
---

公司正在做教育类产品，在遇到数学公式时，我们一般会使用`latex`表达式来做保存和渲染。  
在其中一个项目上，遇到一个需求是要从`office`文档（`Word`或`Excel`）中导入题目内容至数据库，题目内容中就有可能包括数学公式，而在文档中编辑希望使用office的公式插件来写公式元素。  
其实公司之前的产品已经使用`.net`实现过此功能，不过现在公司全面转型`Java`，我们也要研究出一个适用`Java`的解决方案。
<!--more-->

## office文档中的公式编辑器
### mathtype插件
[mathtype](http://www.mathtype.cn/)是一个第三方的数学公式插件，它能在`Office`文档中启用编辑，并生成一个带有公式矢量图的`ole`对象插入到文档中。  
原来`.net`的方案就是使用此种方式，使用`mathtype`提供的`c#`库包来解析`ole`对象，抽取`LaTeX`表达式。  
但在纯`Java`环境下就无法做到了。

### office自带公式编辑器
从2007版开始，`Office`也自带了一个公式编辑器。  
在2007版中`Word`与`Excel`之间不同的是，前者插入的公式对象是`Office MathML`节点，后者插入的还是`ole`。  
到了2010版开始，两个产品的公式编辑器插入的都是`Office MathML`节点了，但是两者对公式对象中的默认文字编码处理不同。
这些不同点可以看出就算同样属于`Office`的产品，他们之间也是有很多不统一的地方。

## 公式表达式
### LaTeX
`LaTeX`是一种基于`ΤΕΧ`的排版系统，它非常适用于生成高印刷质量的科技和数学类文档。
例如勾股定理用`LaTeX`表达：
```
a^{2}+b^{2}=c^{2}
```
常用的`LaTeX`渲染组件是[MathJax](https://www.mathjax.org/)。
我们在项目中使用的便是`LaTeX`，所以本次研究就是如何将`Office`中的公式对象转换成`LaTeX`表达式。
### Mathml
全称为数学标记语言（Mathematical Markup Language），是一种基于`XML`的标准，用来在互联网上书写数学符号和公式的置标语言。
例如一个表达式：
```xml
<math xmlns="http://www.w3.org/1998/Math/MathML">
  	<msup>
    	<mi>n</mi>
    <mrow>
   	  <mi>p</mi>
      <mo>-</mo>
    	  <mn>1</mn>
    </mrow>
  </msup>
  <mspace width=".2em"/>
  <mo>&equiv;</mo>
  <mspace width=".2em"/>
  <mn>1</mn>
  <mspace width=".2em"/>
  <mo>(</mo>
  <mi>mod</mi>
  <mspace width=".2em"/>
  <mi>p</mi>
  <mo>)</mo>
</math>
```
### Office MathML (OMML)
在`office2007`之后版本所编辑的公式对象便是`OMML`。`OMML`是`office`为了配合`Office Open Xml`制定的数学标记语言。  
例如：
```xml
<m:oMathPara><!-- mathematical block container used as a paragraph -->
  <m:oMath><!-- mathematical inline formula -->
    <m:f><!-- a fraction -->
      <m:num><m:r><m:t>π</m:t></m:r></m:num><!-- numerator containing a single run of text -->
      <m:den><m:r><m:t>2</m:t></m:r></m:den><!-- denominator containing a single run of text -->
    </m:f>
  </m:oMath>
</m:oMathPara>
```
### 转换关系
我们在项目中使用到的三者之间转换关系是：`OMML` -> `MathML` -> `LaTex`
`Office`在安装目录中提供了将`OMML`转为`MathML`的`xsl`工具：`MML2OMML.XSL`
`MathML`转`LaTex`使用网上找到另一个`xsl`工具[mmltex.xsl](http://sourceforge.net/projects/xsltml/files/xsltml/)。

## Office文档Java解析
### 2007与之前的版本
用过一段`Office`的同学们都知道，`Office`文档分为`word`与`wordx`这两种类型，分别对应着2007之前与之后的版本格式。  
2007之前版本使用的`Office`文档是二进制文件。而之后版本中x代表的意义是`xml`，表明新版的`Office`文档使用[Office Open Xml](https://en.wikipedia.org/wiki/Office_Open_XML)规范定义文件格式。  
如果我们把`wordx`文件的扩展名改为`zip`，就可以正常解压出`Word`文档包含的所有内容。  

### POI
相信用`Java`做过信息系统的同学都遇过生成统计`Excel`文档或解析`Excel`导入数据的功能。这时我们最常使用的开发库就是[Apache POI](http://poi.apache.org/)。  
`POI`支持二进制与`Office Open Xml`文档，可以满足我们大部分的`Office`文档解析需求。  

## 解析公式实例
首先要说明我们的功能限制：只针对`Office2010`及以上的`Office Open Xml`文档，`Word`和`Excel`均可。  其中，`Excel`的公式数学字符需要转为普通字符，否则会出现`Java`无法识别的字符。  
这里用`Excel`文档为例子来说明解析过程。

### 功能实现思路
这个功能的关键点在于如何获得`Office`文档中的公式节点(`OMML`)，得到`OMML`后我们就可以使用上述的两个工具转换为`LaTeX`。

### 获得OMML
既然我们知道`Excel`文档是一个`xml`，那只需要使用`xml`解析工具读出`OMML`节点就行了。  
先用`POI`得到操作的`XSSFSheet`：
```java
String basePath = "f:\\";
FileInputStream fis = new FileInputStream(basePath + "math.xlsx");
OPCPackage pack = OPCPackage.open(fis);
XSSFWorkbook workbook = new XSSFWorkbook(pack);
XSSFSheet sheet = workbook.getSheetAt(0);
```
插入在`Excel`文档中的图片、公式及其他元素，它都是存放在一个叫`drawing`的单独`xml`文件中，其中的节点记录了元素摆放的位置信息。用`POI`得到`drawing`元素：
```java
XSSFDrawing dr = sheet.getDrawingPatriarch();
CTDrawing drawing = dr.getCTDrawing();
CTOneCellAnchor[] oneCells = drawing.getOneCellAnchorArray();   //所有的图片、公式等元素
```
每个`CTOneCellAnchor`的`xml`里包含元素的位置信息，包括X坐标、Y坐标，所在行、所在列等，更重要的是图片或公式的描述节点。`OMML`节点名为`m:oMathPara`，这里我们就使用`dom4j`的`xpath`来获得`OMML`：
```java
CTOneCellAnchor c = oneCells[0];
String xml = c.xmlText();   //得到xml串

//dom4j解析器的初始化
SAXReader reader = reader = new SAXReader(new DocumentFactory());
Map<String, String> map=new HashMap<String, String>();
map.put("xdr","http://schemas.openxmlformats.org/drawingml/2006/spreadsheetDrawing");
map.put("m","http://schemas.openxmlformats.org/officeDocument/2006/math");
reader.getDocumentFactory().setXPathNamespaceURIs(map); //xml文档的namespace设置

InputSource source = new InputSource(new StringReader(xml));
source.setEncoding("utf-8");
Document doc = reader.read(source);
Element root = doc.getRootElement();
Element e = (Element)root.selectSingleNode("//m:oMathPara");    //用xpath得到OMML节点
String omml = e.asXML();    //转为xml
```
### 转换OMML为Mathml及LaTeX
顺利得到`OMML`后，就可以使用`xsl`转换工具得到`Mathml`与`LaTeX`了。  
这里先写一下`xsl`转换工具方法，使用`javax.xml.transform`工具包实现：
```java
/**    
 * <p>Description: xsl转换器</p>
 */
public static String xslConvert(String s, String xslpath, URIResolver uriResolver){
    TransformerFactory tFac = TransformerFactory.newInstance();
    if(uriResolver != null)  tFac.setURIResolver(uriResolver);
    StreamSource xslSource = new StreamSource(MathmlUtils.class.getResourceAsStream(xslpath));
    StringWriter writer = new StringWriter();   
    try {
        Transformer t = tFac.newTransformer(xslSource);
        Source source = new StreamSource(new StringReader(s));
        Result result = new StreamResult(writer);   
        t.transform(source, result);
    } catch (TransformerException e) {
        logger.error(e.getMessage(), e);
    }
    return writer.getBuffer().toString();
}

/**
 * <p>Description: 将mathml转为latx </p>
 * @param mml
 * @return
 */
public static String convertMML2Latex(String mml){
    mml = mml.substring(mml.indexOf("?>")+2, mml.length()); //去掉xml的头节点
    URIResolver r = new URIResolver(){  //设置xls依赖文件的路径
        @Override
        public Source resolve(String href, String base) throws TransformerException {
            InputStream inputStream = MathmlUtils.class.getResourceAsStream("/conventer/mml2tex/" + href);
            return new StreamSource(inputStream);
        }
    };
    String latex = xslConvert(mml, "/conventer/mml2tex/mmltex.xsl", r);
    if(latex != null && latex.length() > 1){
        latex = latex.substring(1, latex.length() - 1);
    }
    return latex;
}

/**
 * <p>Description: office mathml转为mml </p>
 * @param xml
 * @return
 */
public static String convertOMML2MML(String xml){
    String result = xslConvert(xml, "/conventer/OMML2MML.XSL", null);
    return result;
}
```
至此我们就可以将`OMML`转成`Mathml`与`LaTeX`表达式了：
```
String mml = convertOMML2MML(omml);
String latex = convertMML2Latex(mml);
```

## 一些心得体会
实现这个功能的时候，手上真的也没太多直接的资料可以参考，走过好几个弯路，网上查到的信息很多也是过时或者把话说一半的。  
在与同事的交流下，使用不同思路，查阅许多`api`文档，再加上不断的尝试，也算完成了这个不算实用的功能。  
就算你自己本身不够优秀，在一个好的团队也能不断推着你向前走。一个人最终能前行到多远，还是要看与你同行的人。
