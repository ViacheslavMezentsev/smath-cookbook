# Описание и XSD-схема документа SMath Studio

## Общие положения

- **Формат документа**: `xml`, соответствующий схеме XSD (XML Schema Definition).
- **Структура документа**: Документ представляет собой корневой элемент `worksheet`, содержащий `settings` и `regions`.
- **Пространство имён**: Все элементы документа принадлежат пространству имён `http://smath.info/schemas/worksheet/1.0`.
- **Обратная польская запись (RPN)**: Математические выражения в секциях `math` и `mathcadblock` используют RPN. Их текстовый синтаксис для преобразования в RPN описан в документе `SMathAI.md`. XSD описывает только структуру XML-контейнеров (`<input>`, `<result>`, `<e>`), но не валидирует логику RPN.
- **Корректность структуры**: Обеспечивается соответствием XSD-схеме, которая определяет типы данных, обязательные элементы, атрибуты и их последовательность.
- **Непротиворечивость**: Идентификаторы (`id`, `guid`) должны быть уникальными в пределах документа.
- **Регионы**: Все интерактивные элементы (`text`, `math`, `xyplot` и т.д.) являются дочерними элементами контейнера `<region>`, который определяет их положение и базовые стили. 
- Во всех текстовых регионах `<text>` для атрибута `fontFamily` должно быть установлено значение "Consolas". Дополнительно, внутри элемента `<content>`, основной тег `<p>` должен включать атрибут стиля `style="font-family: Consolas;"`, чтобы гарантировать корректное отображение.
- **Стандартное форматирование ответа:** использовать регион `<text>` для пояснительного предложения, а затем отдельный регион `<math>` для отображения значения вычисленной переменной.
- **Никаких предположений:** не изобретай, не предполагай и не используй какие-либо функции (например, str(), concat() и т.д.), которые не перечислены в списке стандартных функций. Если возможность не описана, она считается недоступной.
- **Строгое соблюдение:** используй только те функции, синтаксис и структуры, которые явно определены в описании синтаксиса (SMathAI.md) и этом документе.

## Генерация идентификатора документа (`<id>`)

- При каждом запросе на создание нового документа SMath Studio ты обязан сгенерировать новый, уникальный идентификатор в формате GUID (версия 4).
- Этот сгенерированный GUID должен быть вставлен в тег `<id>` внутри секции `<settings><identity>...</identity></settings>`.
- **Запрещено** использовать один и тот же GUID для разных документов.
- **Запрещено** использовать статичные или "пустые" GUID, такие как `00000000-0000-0000-0000-000000000000`, за исключением случаев, когда это явно требуется для шаблона.

---

## 1. Общая XSD-схема структуры документа

XSD определяет корневой элемент `worksheet` и его основные дочерние элементы: `settings` и `regions`.

```xml
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="http://smath.info/schemas/worksheet/1.0"
           xmlns="http://smath.info/schemas/worksheet/1.0"
           elementFormDefault="qualified">
  <xs:element name="worksheet">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="settings" type="SettingsType"/>
        <xs:element name="regions" type="RegionsType"/>
      </xs:sequence>
    </xs:complexType>
  </xs:element> 
</xs:schema>
```

---

## 2.1. Секция `settings`

Секция содержит все метаданные, настройки вычислений, страницы и зависимости документа.

```xml
<xs:simpleType name="GuidType">
  <xs:restriction base="xs:string">
    <xs:pattern value="[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}"/>
  </xs:restriction>
</xs:simpleType>

<xs:complexType name="SettingsType">
  <xs:sequence>
    <xs:element name="identity">
      <xs:complexType>
        <xs:sequence>
          <xs:element name="id" type="tns:GuidType"/>
          <xs:element name="revision" type="xs:int"/>
        </xs:sequence>
      </xs:complexType>
    </xs:element>
    <xs:element name="metadata">
      <xs:complexType>
        <xs:sequence>
          <xs:element name="author" type="xs:string" minOccurs="0"/>
        </xs:sequence>
        <xs:attribute name="lang" type="xs:string" use="required"/>
      </xs:complexType>
    </xs:element>
    <xs:element name="calculation">
      <xs:complexType>
        <xs:all>
          <xs:element name="precision" type="xs:int"/>
          <xs:element name="exponentialThreshold" type="xs:int"/>
          <xs:element name="trailingZeros" type="xs:boolean"/>
          <xs:element name="significantDigitsMode" type="xs:boolean"/>
          <xs:element name="mixedNumbers" type="xs:boolean"/>
          <xs:element name="roundingMode" type="xs:int"/>
          <xs:element name="approximateEqualAccuracy" type="xs:int"/>
          <xs:element name="fractions">
            <xs:simpleType>
              <xs:restriction base="xs:string">
                <xs:enumeration value="decimal"/>
                <xs:enumeration value="fraction"/>
              </xs:restriction>
            </xs:simpleType>
          </xs:element>
        </xs:all>
      </xs:complexType>
    </xs:element>
    <xs:element name="pageModel">
      <!-- ... сложное определение, см. ниже ... -->
    </xs:element>
    <xs:element ref="tns:dependencies"/>   
  </xs:sequence>
  <xs:attribute name="ppi" type="xs:int" />
</xs:complexType>
```

## 2.2. Секция `regions`

Контейнер для всех видимых элементов на рабочем листе.

```xml
<xs:element name="regions">
  <xs:complexType>
    <xs:sequence>
      <xs:element name="region" type="RegionType" minOccurs="0" maxOccurs="unbounded"/>
    </xs:sequence>
    <xs:attribute name="type" type="xs:string" use="required"/>
  </xs:complexType>

  <xs:unique name="uniqueRegionId">
    <xs:selector xpath="s:region"/>
    <xs:field xpath="@id"/>
  </xs:unique>
</xs:element>

<xs:complexType name="RegionType">
  <xs:choice>
    <xs:element name="picture" type="PictureRegionType"/>
    <xs:element name="math" type="MathRegionType"/>
    <xs:element name="mathcadblock" type="MathcadBlockRegionType"/>
    <xs:element name="xyplot" type="XyplotRegionType"/>
    <xs:element name="text" type="TextRegionType"/>
  </xs:choice>
  <xs:attribute name="id" type="xs:int" use="optional"/>
  <xs:attribute name="left" type="xs:int" use="required"/>
  <xs:attribute name="top" type="xs:int" use="required"/>
  <xs:attribute name="width" type="xs:int" use="required"/>
  <xs:attribute name="height" type="xs:int" use="required"/>
  <xs:attribute name="color" type="xs:string" use="optional"/>
  <xs:attribute name="bgColor" type="xs:string" use="optional"/>
  <xs:attribute name="fontSize" type="xs:int" use="required"/>
</xs:complexType>
```

## 2.3. Секция `dependencies`

```xml
<!-- 
================================================================================
  Определение элемента <dependencies> и всех связанных с ним типов сборок.
  Сценарий: 8 обязательных сборок в строгом порядке + неограниченное
            количество опциональных сборок после них.
================================================================================
-->

<!-- 
  ШАГ 1: Определяем общий ("generic") тип для любой сборки.
  Он будет использоваться для необязательных плагинов в конце списка.
-->
<xs:complexType name="GenericAssemblyType">
  <xs:attribute name="name" type="xs:string" use="required"/>
  <xs:attribute name="version" type="xs:string" use="required"/>
  <xs:attribute name="guid" type="xs:string" use="required"/>
</xs:complexType>

<!-- 
  ШАГ 2: Определяем отдельные типы для КАЖДОЙ из 8 обязательных сборок.
  Использование "fixed" гарантирует, что значения атрибутов будут строго
  соответствовать указанным.
-->
<xs:complexType name="SMathStudioAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="SMath Studio настольная"/>
  <xs:attribute name="version" type="xs:string" fixed="1.2.9018.0"/>
  <xs:attribute name="guid" type="xs:string" fixed="a37cba83-b69c-4c71-9992-55ff666763bd"/>
</xs:complexType>

<xs:complexType name="MathRegionAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="MathRegion"/>
  <xs:attribute name="version" type="xs:string" fixed="1.11.9018.0"/>
  <xs:attribute name="guid" type="xs:string" fixed="02f1ab51-215b-466e-a74d-5d8b1cf85e8d"/>
</xs:complexType>

<xs:complexType name="PictureRegionAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="PictureRegion"/>
  <xs:attribute name="version" type="xs:string" fixed="1.10.9018.0"/>
  <xs:attribute name="guid" type="xs:string" fixed="06b5df04-393e-4be7-9107-305196fcb861"/>
</xs:complexType>

<xs:complexType name="CustomFunctionsAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="Custom Functions"/>
  <xs:attribute name="version" type="xs:string" fixed="1.1.8726.29023"/>
  <xs:attribute name="guid" type="xs:string" fixed="18dadffd-79a3-4cf9-aee1-d66deb0ea720"/>
</xs:complexType>

<xs:complexType name="SpecialFunctionsAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="SpecialFunctions"/>
  <xs:attribute name="version" type="xs:string" fixed="1.12.9018.0"/>
  <xs:attribute name="guid" type="xs:string" fixed="2814e667-4e12-48b1-8d51-194e480eabc5"/>
</xs:complexType>

<xs:complexType name="TextRegionAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="TextRegion"/>
  <xs:attribute name="version" type="xs:string" fixed="1.11.9018.0"/>
  <xs:attribute name="guid" type="xs:string" fixed="485d28c5-349a-48b6-93be-12a35a1c1e39"/>
</xs:complexType>

<xs:complexType name="XYPlotAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="X-Y Plot Region (JXCharts)"/>
  <xs:attribute name="version" type="xs:string" fixed="0.3.9043.20036"/>
  <xs:attribute name="guid" type="xs:string" fixed="c12231ec-4873-43c1-a7d0-a167ebd17066"/>
</xs:complexType>

<xs:complexType name="MathcadToolboxAssemblyType">
  <xs:attribute name="name" type="xs:string" fixed="Mathcad Toolbox"/>
  <xs:attribute name="version" type="xs:string" fixed="0.5.9065.32492"/>
  <xs:attribute name="guid" type="xs:string" fixed="ddc09821-49f1-4c21-a829-6499de0a8f06"/>
</xs:complexType>


<!-- 
  ШАГ 3: Определяем сам элемент <dependencies> как последовательность
  сначала обязательных, а затем необязательных сборок.
-->
<xs:element name="dependencies">
  <xs:complexType>
    <xs:sequence>
      <!-- Сначала идут 8 обязательных сборок в строгом порядке -->
      <xs:element name="assembly" type="tns:SMathStudioAssemblyType"/>
      <xs:element name="assembly" type="tns:MathRegionAssemblyType"/>
      <xs:element name="assembly" type="tns:PictureRegionAssemblyType"/>
      <xs:element name="assembly" type="tns:CustomFunctionsAssemblyType"/>
      <xs:element name="assembly" type="tns:SpecialFunctionsAssemblyType"/>
      <xs:element name="assembly" type="tns:TextRegionAssemblyType"/>
      <xs:element name="assembly" type="tns:XYPlotAssemblyType"/>
      <xs:element name="assembly" type="tns:MathcadToolboxAssemblyType"/>
      
      <!-- 
        Затем могут идти любые другие сборки (например, от сторонних плагинов).
        minOccurs="0" делает их необязательными.
        maxOccurs="unbounded" разрешает любое их количество.
      -->
      <xs:element name="assembly" type="tns:GenericAssemblyType" minOccurs="0" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>
</xs:element>
```

---

## 3. Типы регионов

### 3.1. Секция `picture`

```xml
<xs:complexType name="PictureRegionType">
  <xs:sequence>
    <xs:element name="raw">
      <xs:simpleContent>
        <xs:extension base="xs:string">
          <xs:attribute name="format" type="xs:string" use="required"/>
          <xs:attribute name="encoding" type="xs:string" use="required"/>
        </xs:extension>
      </xs:simpleContent>
    </xs:element>
  </xs:sequence>
</xs:complexType>
```

### 3.2. Секция `math`

```xml
<xs:complexType name="MathRegionType">
  <xs:sequence>
    <xs:element name="input" type="RpnExpressionType"/>
    <xs:element name="contract" type="RpnExpressionType" minOccurs="0"/>
    <xs:element name="result" minOccurs="0">
      <xs:complexType>
        <xs:complexContent>
          <xs:extension base="RpnExpressionType">
            <xs:attribute name="action" type="xs:string" use="optional" default="numeric"/>
          </xs:extension>
        </xs:complexContent>
      </xs:complexType>
    </xs:element>
  </xs:sequence>
  <xs:attribute name="evaluate" type="xs:boolean" use="optional" default="true"/>
  <xs:attribute name="optimize" type="xs:int" use="optional"/>
  <xs:attribute name="breaking" type="xs:int" use="optional"/>
  <xs:attribute name="decimalPlaces" type="xs:int" use="optional"/>
</xs:complexType>

<!-- Общий тип для RPN-выражений -->
<xs:complexType name="RpnExpressionType">
  <xs:sequence>
    <xs:element name="e" minOccurs="0" maxOccurs="unbounded">
      <xs:simpleContent>
        <xs:extension base="xs:string">
          <xs:attribute name="type" type="xs:string" use="required"/>
          <xs:attribute name="args" type="xs:int" use="optional"/>
          <!-- Атрибут style теперь явно включает 'unit' -->
          <xs:attribute name="style" type="xs:string" use="optional"/>
        </xs:extension>
      </xs:simpleContent>
    </xs:element>
  </xs:sequence>
</xs:complexType>
```

### 3.3. Секция `mathcadblock`

```xml
<xs:complexType name="MathcadBlockRegionType">
  <xs:sequence>
    <xs:element name="input" type="RpnExpressionType"/>
  </xs:sequence>
  <xs:attribute name="width" type="xs:int" use="required"/>
  <xs:attribute name="height" type="xs:int" use="required"/>
  <xs:attribute name="seqop" type="xs:string" use="required"/>
</xs:complexType>
```

### 3.4. Секция `xyplot`

```xml
<xs:complexType name="ChartStyleType">
  <xs:attribute name="usedefault" type="xs:boolean" use="optional"/>
  <xs:attribute name="backcolor" type="xs:string" use="optional"/>
  <xs:attribute name="bordercolor" type="xs:string" use="optional"/>
</xs:complexType>

<!-- Базовый тип для осей -->
<xs:complexType name="AxisBaseType">
  <xs:attribute name="visible" type="xs:boolean" use="optional"/>
  <xs:attribute name="decimalplaces" type="xs:int" use="optional"/>
  <xs:attribute name="numberformat" type="NumberFormatEnum" use="optional"/>
</xs:complexType>

<!-- Конкретные оси -->
<xs:complexType name="XAxisType">
  <xs:complexContent>
    <xs:extension base="AxisBaseType">
      <xs:attribute name="xmin" type="xs:float" use="optional"/>
      <xs:attribute name="xmax" type="xs:float" use="optional"/>
      <xs:attribute name="xtick" type="xs:float" use="optional"/>
    </xs:extension>
  </xs:complexContent>
</xs:complexType>

<xs:complexType name="YAxisType">
  <xs:complexContent>
    <xs:extension base="AxisBaseType">
      <xs:attribute name="ymin" type="xs:float" use="optional"/>
      <xs:attribute name="ymax" type="xs:float" use="optional"/>
      <xs:attribute name="ytick" type="xs:float" use="optional"/>
    </xs:extension>
  </xs:complexContent>
</xs:complexType>

<xs:complexType name="Y2AxisType">
  <xs:complexContent>
    <xs:extension base="AxisBaseType">
      <xs:attribute name="isy2axis" type="xs:boolean" use="optional"/>
      <xs:attribute name="y2min" type="xs:float" use="optional"/>
      <xs:attribute name="y2max" type="xs:float" use="optional"/>
      <xs:attribute name="y2tick" type="xs:float" use="optional"/>
    </xs:extension>
  </xs:complexContent>
</xs:complexType>

<xs:complexType name="GridType">
  <xs:attribute name="gridcolor" type="xs:string" use="optional"/>
  <xs:attribute name="gridpattern" type="LinePatternEnum" use="optional"/>
  <xs:attribute name="gridthickness" type="xs:float" use="optional"/>
  <xs:attribute name="isxgrid" type="xs:boolean" use="optional"/>
  <xs:attribute name="isygrid" type="xs:boolean" use="optional"/>
  <xs:attribute name="isy2grid" type="xs:boolean" use="optional"/>
</xs:complexType>

<xs:complexType name="Title2DType">
  <xs:attribute name="title" type="xs:string" use="optional"/>
  <xs:attribute name="titlefont" type="xs:string" use="optional"/>
  <xs:attribute name="titlefontcolor" type="xs:string" use="optional"/>
</xs:complexType>

<xs:complexType name="XYLabelType">
  <xs:attribute name="labelfont" type="xs:string" use="optional"/>
  <xs:attribute name="labelfontcolor" type="xs:string" use="optional"/>
  <xs:attribute name="tickfont" type="xs:string" use="optional"/>
  <xs:attribute name="tickfontcolor" type="xs:string" use="optional"/>
  <xs:attribute name="xlabel" type="xs:string" use="optional"/>
  <xs:attribute name="ylabel" type="xs:string" use="optional"/>
  <xs:attribute name="y2label" type="xs:string" use="optional"/>
</xs:complexType>

<xs:complexType name="LegendType">
  <xs:attribute name="isbordervisible" type="xs:boolean" use="optional"/>
  <xs:attribute name="islegendvisible" type="xs:boolean" use="optional"/>
  <xs:attribute name="legendbackcolor" type="xs:string" use="optional"/>
  <xs:attribute name="legendbordercolor" type="xs:string" use="optional"/>
  <xs:attribute name="legendfont" type="xs:string" use="optional"/>
  <xs:attribute name="legendposition" type="LegendPositionEnum" use="optional"/>
  <xs:attribute name="textcolor" type="xs:string" use="optional"/>
</xs:complexType>

<!-- Перечисления -->
<xs:simpleType name="PlotMethodEnum">
  <xs:restriction base="xs:string">
    <xs:enumeration value="Lines"/>
    <xs:enumeration value="Splines"/>
    <xs:enumeration value="Labels"/>
    <xs:enumeration value="Shapes"/>
  </xs:restriction>
</xs:simpleType>

<xs:simpleType name="LinePatternEnum">
  <xs:restriction base="xs:string">
    <xs:enumeration value="Solid"/>
    <xs:enumeration value="Dash"/>
    <xs:enumeration value="Dot"/>
    <xs:enumeration value="DashDot"/>
    <xs:enumeration value="DashDotDot"/>
  </xs:restriction>
</xs:simpleType>

<xs:simpleType name="LegendPositionEnum">
  <xs:restriction base="xs:string">
    <xs:enumeration value="NorthWest"/>
    <xs:enumeration value="NorthEast"/>
    <xs:enumeration value="SouthWest"/>
    <xs:enumeration value="SouthEast"/>
    <xs:enumeration value="Center"/>
  </xs:restriction>
</xs:simpleType>

<xs:simpleType name="HatchStyleEnum">
  <xs:restriction base="xs:string">
    <xs:enumeration value="Horizontal"/>
    <xs:enumeration value="Vertical"/>
    <xs:enumeration value="ForwardDiagonal"/>
    <xs:enumeration value="BackwardDiagonal"/>
    <xs:enumeration value="Cross"/>
    <xs:enumeration value="DiagonalCross"/>
  </xs:restriction>
</xs:simpleType>

<xs:simpleType name="SymbolTypeEnum">
  <xs:restriction base="xs:string">
    <xs:enumeration value="None"/>
    <xs:enumeration value="Circle"/>
    <xs:enumeration value="Square"/>
    <xs:enumeration value="Triangle"/>
    <xs:enumeration value="Diamond"/>
  </xs:restriction>
</xs:simpleType>

<xs:simpleType name="NumberFormatEnum">
  <xs:restriction base="xs:string">
    <xs:enumeration value="General"/>
    <xs:enumeration value="Fixed"/>
    <xs:enumeration value="Scientific"/>
    <xs:enumeration value="Engineering"/>
    <xs:enumeration value="Currency"/>
  </xs:restriction>
</xs:simpleType>

<!-- Trace -->
<xs:complexType name="TraceType">
  <xs:attribute name="seriesname" type="xs:string" use="optional"/>
  <xs:attribute name="isy2data" type="xs:boolean" use="optional"/>
  <xs:attribute name="isvisible" type="xs:boolean" use="optional"/>
  <xs:attribute name="plotmethod" type="PlotMethodEnum" use="optional"/>
  <xs:attribute name="lineantialias" type="xs:boolean" use="optional"/>
  <xs:attribute name="linecolor" type="xs:string" use="optional"/>
  <xs:attribute name="linethickness" type="xs:float" use="optional"/>
  <xs:attribute name="linepattern" type="LinePatternEnum" use="optional"/>
  <xs:attribute name="fillmode" type="xs:string" use="optional"/>
  <xs:attribute name="filltotrace" type="xs:int" use="optional"/>
  <xs:attribute name="filled" type="xs:boolean" use="optional"/>
  <xs:attribute name="hatched" type="xs:boolean" use="optional"/>
  <xs:attribute name="fillcolor" type="xs:string" use="optional"/>
  <xs:attribute name="hatchstyle" type="HatchStyleEnum" use="optional"/>
  <xs:attribute name="symbolantialias" type="xs:boolean" use="optional"/>
  <xs:attribute name="symbolsize" type="xs:int" use="optional"/>
  <xs:attribute name="symboltype" type="SymbolTypeEnum" use="optional"/>
  <xs:attribute name="symbolborderthickness" type="xs:int" use="optional"/>
  <xs:attribute name="symbolbordercolor" type="xs:string" use="optional"/>
  <xs:attribute name="symbolfillcolor" type="xs:string" use="optional"/>
</xs:complexType>

<!-- Основной тип XYPlot -->
<xs:complexType name="XyplotRegionType">
  <xs:sequence>
    <xs:element name="chartstyle" type="ChartStyleType" minOccurs="0"/>
    <xs:element name="grid" type="GridType" minOccurs="0"/>
    <xs:element name="xaxes" type="XAxisType" minOccurs="0"/>
    <xs:element name="yaxes" type="YAxisType" minOccurs="0"/>
    <xs:element name="y2axes" type="Y2AxisType" minOccurs="0"/>
    <xs:element name="title2d" type="Title2DType" minOccurs="0"/>
    <xs:element name="xylabel" type="XYLabelType" minOccurs="0"/>
    <xs:element name="legend" type="LegendType" minOccurs="0"/>
    <xs:element name="traces" minOccurs="0">
      <xs:complexType>
        <xs:sequence>
          <xs:element name="trace" type="TraceType" minOccurs="0" maxOccurs="unbounded"/>
        </xs:sequence>
      </xs:complexType>
    </xs:element>
    <xs:element name="input" type="RpnExpressionType"/>
  </xs:sequence>

  <!-- Атрибуты xyplot -->
  <xs:attribute name="animFrameRate" type="xs:int" use="optional"/>
  <xs:attribute name="animPlaybackMode" type="xs:string" use="optional"/>
  <xs:attribute name="keepAspectRatio" type="xs:boolean" use="optional"/>
  <xs:attribute name="width" type="xs:int" use="required"/>
  <xs:attribute name="height" type="xs:int" use="required"/>
  <xs:attribute name="points" type="xs:int" use="required"/>
  <xs:attribute name="name" type="xs:string" use="optional"/>
</xs:complexType>
```

### 3.5. Секция `text`

```xml
<xs:complexType name="TextRegionType">
  <xs:sequence>
    <xs:element name="content">
      <xs:complexType>
        <xs:sequence>
          <!-- 
            Содержимое <p> может быть сложным, включая теги <span> со стилями
            для форматирования отдельных частей текста.
          -->
          <xs:any namespace="##any" processContents="lax" minOccurs="0" maxOccurs="unbounded"/>
        </xs:sequence>
      </xs:complexType>
    </xs:element>
  </xs:sequence>
  <xs:attribute name="lang" type="xs:string" use="required"/>
  <xs:attribute name="fontFamily" type="xs:string" use="required"/>
  <xs:attribute name="fontSize" type="xs:int" use="required"/>
</xs:complexType>
```

---

### 4. Генерация XML RPN из текстовых выражений

Этот раздел описывает **критически важные правила** для преобразования текстовых выражений (описанных в EBNF) в корректную RPN-последовательность XML-элементов (`<e>`).
Генератор **обязан** сначала анализировать синтаксическую структуру всего выражения, чтобы понять его иерархию.
Генератор **обязан** рекурсивно обходить дерево текстового выражения, строго следуя этим правилам для формирования корректной, плоской RPN-последовательности в XML.
Генератор должен быть перестроен для работы по принципу "сверху вниз": сначала найти главный оператор, рекурсивно обработать его дочерние узлы (LHS/RHS или аргументы), а затем собрать итоговую RPN-последовательность, строго следуя этим структурным правилам.
Несоблюдение этих правил приведет к тому, что SMath Studio не сможет отобразить математический регион.

#### Правило 4.1: Определите главный оператор (иерархический разбор)

Перед началом генерации RPN для любого выражения, необходимо найти его **главный (top-level) оператор**. Это может быть определение функции (`:`), бинарный оператор (`+`, `-`) или вызов функции. Весь процесс генерации должен строиться рекурсивно вокруг этого оператора.

*   **Текстовое выражение:** `geo2ecef(geo) : line(...)`
*   **Анализ структуры:**
    *   **Главный оператор:** `:` (определение функции).
    *   **Левая часть (LHS):** `geo2ecef(geo)`.
    *   **Правая часть (RHS):** `line(...)`.
*   **Общая RPN-структура:** `[RPN для LHS] [RPN для RHS] [Оператор :]`

Все, что находится внутри `line(...)`, является частью правой стороны **единственного** оператора определения.

#### Правило 4.2: Корректная RPN-генерация для определения функции (`f(x):body`)

Определение функции — это частный случай иерархического разбора, имеющий строгую RPN-форму.

1.  **Разберите левую часть (LHS): `f(arg1, arg2, ...)`**. Ее RPN-формат: сначала все аргументы, затем сама функция с указанием их числа в `args`.
2.  **Разберите правую часть (RHS): `body`**. Рекурсивно сгенерируйте полную RPN-последовательность для тела функции.
3.  **Соберите итоговую RPN:** `[RPN для LHS] [RPN для RHS] [Оператор ":"]`.

*   **Текстовое выражение:** `geo2ecef(geo) : line(...)`
*   **Правильная XML RPN последовательность:**
    ```xml
    <!-- ШАГ 1: RPN для левой части "geo2ecef(geo)" -->
    <e type="operand">geo</e>
    <e type="function" args="1">geo2ecef</e>

    <!-- ШАГ 2: RPN для правой части (тела функции) "line(...)" -->
    <!-- (Здесь будет полная RPN-последовательность для line, см. рабочий пример) -->
    ... RPN для 6 выражений внутри line ...
    <e type="operand">6</e>
    <e type="operand">1</e>
    <e type="function" args="8">line</e>

    <!-- ШАГ 3: В самом конце - главный оператор определения -->
    <e type="operator" args="2">:</e>
    ```

#### Правило 4.3: Имя функции принадлежит тегу `<e type="function">`

Генератор не должен помещать имя функции (`line`, `mat`, `sin`) в стек как операнд (`<e type="operand">line</e>`), а затем вызывать некую универсальную функцию-заглушку (`<e type="function">f</e>`). Это грубая структурная ошибка.

**Правило:** Имя вызываемой функции является **текстовым содержимым** тега `<e type="function">...</e>`.

*   **Текстовое выражение:** `mat(lat, lon, alt, 1, 3)`
*   **Неправильный XML (ошибка генератора):**
    ```xml
    <e type="operand">mat</e> <!-- ОШИБКА: 'mat' помещен в стек как строка -->
    <e type="operand">lat</e>
    <e type="operand">lon</e>
    <e type="operand">alt</e>
    <e type="operand">1</e>
    <e type="operand">3</e>
    <e type="function" args="5" style="matrix-placeholder">f</e> <!-- ОШИБКА: вызов безымянной функции -->
    ```
*   **Правильный XML:**
    ```xml
    <!-- Сначала все 5 аргументов -->
    <e type="operand">lat</e>
    <e type="operand">lon</e>
    <e type="operand">alt</e>
    <e type="operand">1</e>
    <e type="operand">3</e>
    <!-- Затем вызов КОНКРЕТНОЙ функции 'mat' -->
    <e type="function" args="5">mat</e>
    ```


#### Правило 4.4: Сначала операнды, затем оператор (Основной принцип RPN)

Это фундаментальное правило. Для любой операции или функции все ее аргументы (операнды) должны быть помещены в RPN-стек **до** того, как будет помещен сам оператор или функция.

*   **Текстовое выражение:** `f(x, y)`
*   **Правильная XML RPN последовательность:**
    ```xml
    <!-- Сначала операнд 'x' -->
    <e type="operand">x</e>
    <!-- Затем операнд 'y' -->
    <e type="operand">y</e>
    <!-- В конце - функция 'f', которая забирает со стека 2 аргумента -->
    <e type="function" args="2">f</e>
    ```

#### Правило 4.5: Критические правила для блочных функций (`line`, `mat`, `sys`)

Неправильная генерация этих блоков — самая частая причина ошибок. Для них действуют два взаимосвязанных правила.

**Правило A: Аргументы-размеры должны быть в стеке**

Два последних аргумента в текстовом представлении блочной функции (`rows`, `cols`) **должны** быть помещены в RPN-стек как обычные операнды (`<e type="operand">`) сразу после всех выражений, содержащихся в блоке, и непосредственно перед вызовом самой блочной функции.

**Правило B: Правильный подсчет `args`**

Атрибут `args` для блочной функции **всегда** вычисляется по формуле:
`args = (количество выражений внутри блока) + 2`

Эта двойка (`+2`) соответствует двум операндам-размерам (`rows` и `cols`), которые функция также забирает из стека.

**Полный пример разбора `line()`**

*   **Текстовое выражение:**
    `line(expr1, expr2, expr3, expr4, expr5, expr6, 6, 1)`

*   **Анализ:**
    1.  Количество выражений в блоке: **6** (`expr1` ... `expr6`).
    2.  Аргументы-размеры: `rows=6`, `cols=1`.
    3.  Общее количество аргументов для функции `line`: 6 (выражений) + 2 (размеров) = **8**.

*   **Правильная XML RPN последовательность:**
    ```xml
    <!-- 1. Сначала в стек помещаются RPN-представления ВСЕХ 6 выражений -->
    ... RPN для expr1 ...
    ... RPN для expr2 ...
    ... RPN для expr3 ...
    ... RPN для expr4 ...
    ... RPN для expr5 ...
    ... RPN для expr6 ...

    <!-- 2. Затем в стек помещаются два операнда-размера -->
    <e type="operand">6</e>
    <e type="operand">1</e>

    <!-- 3. В самом конце вызывается функция line с правильным числом args (8) -->
    <e type="function" args="8" style="program">line</e>
    ```

#### Правило 4.6: Группировка математических операций с помощью `<e type="bracket">(</e>`

В текстовых выражениях круглые скобки `()` используются для изменения стандартного приоритета операций. В XML RPN для этой цели служит специальный тег `<e type="bracket">(</e>`.

**Правило:** Тег `<e type="bracket">(</e>` должен быть помещен в стек **сразу после** того, как RPN-последовательность для выражения внутри скобок была полностью вычислена. Он помечает результат этой под-операции как единый, сгруппированный операнд для следующей операции.

**Пример сравнения**

*   **Выражение 1 (без скобок):** `n * 1 - e2` (RPN: `n 1 * e2 -`)
*   **Выражение 2 (со скобками):** `n * (1 - e2)` (RPN: `n 1 e2 - ( *`)

*   **Правильная XML RPN для `n * (1 - e2)`:**
    ```xml
    <!-- Операнд 'n' -->
    <e type="operand">n</e>
    
    <!-- RPN-последовательность для выражения в скобках (1 - e2) -->
    <e type="operand">1</e>
    <e type="operand">e2</e>
    <e type="operator" args="2">-</e>

    <!-- Сразу после вычисления (1 - e2) ставим тег-маркер группировки -->
    <e type="bracket">(</e>

    <!-- В конце - оператор, который применится к 'n' и результату группы (1-e2) -->
    <e type="operator" args="2">*</e>
    ```

### Правило 4.7. Units of Measurement

*   **XML Representation:** A unit is an operand with `style="unit"`. It is attached to a value via multiplication. The expression "75*'km" is represented as:
    ```xml
    <e type="operand">75</e>
    <e type="operand" style="unit">km</e>
    <e type="operator" args="2">*</e>
    ```

---

### Правило 4.8. Displaying and Evaluating Results

This section explains the correct structure for showing a calculated value. The evaluation operator (`=`) is a user interface element that creates a special `math` region; it is **not** an RPN operator.

*   **Structure:** A result display region consists of three key parts within a `<math>` element:
    1.  `<input>`: Contains the RPN for the expression to be evaluated (e.g., the name of a variable).
    2.  `<contract>` (Optional): Contains the unit of measurement in which the result should be displayed.
    3.  `<result>`: A tag that holds a placeholder for the computed value. The SMath engine will perform the calculation and populate this tag with the final result.

*   **The `<result>` Tag Rule:** The `<result>` tag **must not be empty**. It must contain a single, specific placeholder element that represents the "unevaluated" state:
    ```xml
    <e type="operand">#</e>
    ```
    The AI's role is to generate this exact placeholder. The SMath application will replace this `#` operand with the actual computed value upon calculation.

*   **Evaluation Action:** For standard numerical calculations, the `<result>` tag must include the attribute `action="numeric"`.

*   **Complete Example:** To display the value of the variable `t.2` in hours (`'hr`), the following XML structure must be generated:
    ```xml
    <region left="126" top="459" width="62" height="30" fontSize="10">
      <math>
        <input>
          <e type="operand">t.2</e>
        </input>
        <contract>
          <e type="operand" style="unit">hr</e>
        </contract>
        <!-- The result tag contains the standard placeholder for a pending calculation. -->
        <result action="numeric">
          <e type="operand">#</e>
        </result>
      </math>
    </region>
    ```
    
---

## 5. Примеры использования дополнений

### 5.1. MathRegion

```xml
<region left="10" top="100" width="80" height="20">
  <math>
    <input>
      <e type="operand">x</e>
      <e type="operand">2</e>
      <e type="operator" args="2">^</e>
    </input>
  </math>
</region>
```

(результат: `x^2`)

### 5.2. TextRegion

```xml
<region left="50" top="20" width="200" height="30">
  <text lang="eng" fontFamily="Calibri" fontSize="12">
    <content>
      <p style="font-size:12px;">Simulation results</p>
    </content>
  </text>
</region>
```

### 5.3. PictureRegion

```xml
<region left="0" top="200" width="300" height="150">
  <picture>
    <raw format="png" encoding="base64">iVBORw0KGgoAAAANS...</raw>
  </picture>
</region>
```

### 5.4. X-Y Plot Region

```xml
<region left="0" top="400" width="400" height="300">
  <xyplot width="390" height="290" points="100" name="Trajectory">
    <input>
      <e type="operand">~t</e>
      <e type="function" args="1">x1</e>
      <e type="operand">~t</e>
      <e type="function" args="1">y1</e>
      <e type="operand">2</e>
      <e type="operand">1</e>
      <e type="function" args="4">mat</e>
    </input>
  </xyplot>
</region>
```

### 5.5. Mathcad Toolbox (пример блоков)

```xml
<region left="100" top="500" width="250" height="80">
  <mathcadblock width="240" height="70" seqop="sys">
    <input>
      <e type="operand">M</e>
      <e type="operand">R</e>
      <e type="operand">t.end</e>
      <e type="operand">1000</e>
      <e type="function" args="3">Rkadapt</e>
      <e type="operator" args="2">:</e>
    </input>
  </mathcadblock>
</region>
```

---

## 6. Минимальные шаблоны регионов

### Пустой текстовый регион

```xml
<region left="0" top="0" width="100" height="20">
  <text lang="eng" fontFamily="Arial" fontSize="10">
    <content><p></p></content>
  </text>
</region>
```

### Пустой math регион

```xml
<region left="0" top="0" width="50" height="20">
  <math>
    <input />
  </math>
</region>
```

### Пустой mathcadblock

```xml
<region left="0" top="0" width="100" height="40">
  <mathcadblock width="90" height="30" seqop="sys">
    <input />
  </mathcadblock>
</region>
```

### Пустой xyplot

```xml
<region left="0" top="0" width="200" height="150">
  <xyplot width="190" height="140" points="10" name="Plot">
    <input />
  </xyplot>
</region>
```

### Пустой picture

```xml
<region left="0" top="0" width="100" height="100">
  <picture>
    <raw format="png" encoding="base64"></raw>
  </picture>
</region>
```

---

## 7. Минимальный пример пустого документа

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<?application progid="SMath Studio настольная" version="1.2.9018.0"?>
<worksheet xmlns="http://smath.info/schemas/worksheet/1.0">
  <settings ppi="96">
    <identity>
      <id>00000000-0000-0000-0000-000000000000</id>
      <revision>1</revision>
    </identity>
    <metadata lang="eng">
      <author>Author</author>
    </metadata>
    <calculation>
      <precision>4</precision>
      <exponentialThreshold>5</exponentialThreshold>
      <trailingZeros>false</trailingZeros>
      <significantDigitsMode>false</significantDigitsMode>
      <mixedNumbers>false</mixedNumbers>
      <roundingMode>0</roundingMode>
      <approximateEqualAccuracy>0</approximateEqualAccuracy>
      <fractions>decimal</fractions>
    </calculation>
    <pageModel active="false" viewMode="2" printGrid="false" printAreas="true" simpleEqualsOnly="false" printBackgroundImages="true" hideElementsHighlightings="false">
      <paper id="9" orientation="Portrait" width="827" height="1169" />
      <margins left="19" right="19" top="39" bottom="39" />
      <header alignment="Center" color="#a9a9a9" />
      <footer alignment="Center" color="#a9a9a9" />
      <backgrounds />
    </pageModel>
    <dependencies>
      <assembly name="SMath Studio настольная" version="1.2.9018.0" guid="a37cba83-b69c-4c71-9992-55ff666763bd" />
      <assembly name="MathRegion" version="1.11.9018.0" guid="02f1ab51-215b-466e-a74d-5d8b1cf85e8d" />
      <assembly name="PictureRegion" version="1.10.9018.0" guid="06b5df04-393e-4be7-9107-305196fcb861" />
      <assembly name="Custom Functions" version="1.1.8726.29023" guid="18dadffd-79a3-4cf9-aee1-d66deb0ea720" />
      <assembly name="SpecialFunctions" version="1.12.9018.0" guid="2814e667-4e12-48b1-8d51-194e480eabc5" />
      <assembly name="TextRegion" version="1.11.9018.0" guid="485d28c5-349a-48b6-93be-12a35a1c1e39" />
      <assembly name="X-Y Plot Region (JXCharts)" version="0.3.9043.20036" guid="c12231ec-4873-43c1-a7d0-a167ebd17066" />
      <assembly name="Mathcad Toolbox" version="0.5.9065.32492" guid="ddc09821-49f1-4c21-a829-6499de0a8f06" />
    </dependencies>
  </settings>
  <regions type="content">
  </regions>
</worksheet>
```
