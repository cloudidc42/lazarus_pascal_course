# Part 42 - XML Processing ใน Lazarus/Pascal

## บทนำ

XML (eXtensible Markup Language) เป็นภาษา markup ที่ใช้กันอย่างแพร่หลายในการแลกเปลี่ยนข้อมูล configuration files, web services และ document formats ต่างๆ Lazarus/FPC มี unit `DOM`, `XMLRead`, `XMLWrite` สำหรับการจัดการ XML แบบครบถ้วน

---

## 42.1 DOM vs SAX

### DOM (Document Object Model)
- โหลด XML ทั้งไฟล์เข้า memory
- สร้าง tree structure ที่ navigate ได้
- เหมาะสำหรับไฟล์ขนาดเล็ก-กลาง
- สามารถแก้ไข tree ได้
- ใช้ memory มากกว่า

### SAX (Simple API for XML)
- อ่าน XML แบบ streaming
- Event-driven (callback functions)
- เหมาะสำหรับไฟล์ขนาดใหญ่
- ไม่สามารถแก้ไข in-place ได้
- ใช้ memory น้อยกว่า

```pascal
// Units ที่ต้องใช้
uses
  DOM,           // DOM classes
  XMLRead,       // อ่าน XML
  XMLWrite,      // เขียน XML
  SAX,           // SAX parser
  SysUtils;
```

---

## 42.2 XMLDocument

### การสร้าง XML Document ใหม่

```pascal
program CreateXMLDocument;

{$mode objfpc}{$H+}

uses
  DOM, XMLWrite, SysUtils;

var
  Doc: TXMLDocument;
  Root: TDOMElement;
  Child: TDOMElement;
  TextNode: TDOMText;
begin
  // สร้าง XML Document ใหม่
  Doc := TXMLDocument.Create;
  try
    // สร้าง root element
    Root := Doc.CreateElement('employees');
    Doc.AppendChild(Root);
    
    // เพิ่ม attribute ให้ root
    Root.SetAttribute('version', '1.0');
    Root.SetAttribute('company', 'บริษัท ตัวอย่าง จำกัด');
    
    // สร้าง employee element แรก
    Child := Doc.CreateElement('employee');
    Child.SetAttribute('id', '001');
    Root.AppendChild(Child);
    
    // เพิ่ม name element
    TextNode := Doc.CreateTextNode('สมชาย ใจดี');
    Child.AppendChild(Doc.CreateElement('name')).AppendChild(TextNode);
    
    // เพิ่ม position
    Child.AppendChild(Doc.CreateElement('position')).
      AppendChild(Doc.CreateTextNode('Software Engineer'));
    
    // เพิ่ม salary
    Child.AppendChild(Doc.CreateElement('salary')).
      AppendChild(Doc.CreateTextNode('45000'));
    
    // สร้าง employee element ที่สอง
    Child := Doc.CreateElement('employee');
    Child.SetAttribute('id', '002');
    Root.AppendChild(Child);
    
    Child.AppendChild(Doc.CreateElement('name')).
      AppendChild(Doc.CreateTextNode('สมหญิง รักดี'));
    Child.AppendChild(Doc.CreateElement('position')).
      AppendChild(Doc.CreateTextNode('Project Manager'));
    Child.AppendChild(Doc.CreateElement('salary')).
      AppendChild(Doc.CreateTextNode('65000'));
    
    // บันทึกลงไฟล์
    WriteXMLFile(Doc, 'employees.xml');
    WriteLn('XML file created: employees.xml');
    
    // แสดงผลใน console
    WriteXML(Doc, Output);
    
  finally
    Doc.Free;
  end;
end.
```

**Output XML:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<employees version="1.0" company="บริษัท ตัวอย่าง จำกัด">
  <employee id="001">
    <name>สมชาย ใจดี</name>
    <position>Software Engineer</position>
    <salary>45000</salary>
  </employee>
  <employee id="002">
    <name>สมหญิง รักดี</name>
    <position>Project Manager</position>
    <salary>65000</salary>
  </employee>
</employees>
```

---

## 42.3 Reading XML

### การอ่าน XML File

```pascal
program ReadXMLFile;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, SysUtils;

procedure ProcessEmployeeNode(Node: TDOMElement);
var
  NameNode: TDOMNode;
  ChildNodes: TDOMNodeList;
  i: integer;
  ElementName: string;
  Value: string;
begin
  WriteLn('');
  WriteLn('Employee ID: ', Node.GetAttribute('id'));
  
  // วน loop ผ่าน child nodes
  ChildNodes := Node.ChildNodes;
  for i := 0 to ChildNodes.Length - 1 do
  begin
    if ChildNodes[i].NodeType = ELEMENT_NODE then
    begin
      ElementName := TDOMElement(ChildNodes[i]).TagName;
      
      // อ่าน text content
      if ChildNodes[i].HasChildNodes then
        Value := ChildNodes[i].FirstChild.NodeValue
      else
        Value := '';
        
      WriteLn('  ', ElementName, ': ', Value);
    end;
  end;
end;

var
  Doc: TXMLDocument;
  Root: TDOMElement;
  EmployeeNodes: TDOMNodeList;
  i: integer;
  XMLContent: string;
begin
  // สร้าง XML content สำหรับทดสอบ
  XMLContent :=
    '<?xml version="1.0" encoding="UTF-8"?>' +
    '<employees>' +
    '<employee id="001">' +
    '<name>สมชาย ใจดี</name>' +
    '<position>Software Engineer</position>' +
    '<department>IT</department>' +
    '<salary>45000</salary>' +
    '<active>true</active>' +
    '</employee>' +
    '<employee id="002">' +
    '<name>สมหญิง รักดี</name>' +
    '<position>Project Manager</position>' +
    '<department>IT</department>' +
    '<salary>65000</salary>' +
    '<active>true</active>' +
    '</employee>' +
    '<employee id="003">' +
    '<name>วิชัย สุขสันต์</name>' +
    '<position>Database Admin</position>' +
    '<department>IT</department>' +
    '<salary>52000</salary>' +
    '<active>false</active>' +
    '</employee>' +
    '</employees>';
  
  // เขียน XML ลงไฟล์ก่อน
  with TStringList.Create do
  try
    Text := XMLContent;
    SaveToFile('test_employees.xml');
  finally
    Free;
  end;
  
  // อ่าน XML file
  ReadXMLFile(Doc, 'test_employees.xml');
  try
    Root := Doc.DocumentElement;
    WriteLn('Root element: ', Root.TagName);
    WriteLn('Attributes:');
    
    // หา employee elements ทั้งหมด
    EmployeeNodes := Root.GetElementsByTagName('employee');
    WriteLn('Number of employees: ', EmployeeNodes.Length);
    
    for i := 0 to EmployeeNodes.Length - 1 do
      ProcessEmployeeNode(TDOMElement(EmployeeNodes[i]));
    
    EmployeeNodes.Free; // ต้อง free TDOMNodeList
    
  finally
    Doc.Free;
  end;
  
  // ลบไฟล์ test
  DeleteFile('test_employees.xml');
end.
```

### การค้นหา Elements

```pascal
program SearchXMLElements;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, SysUtils;

// ค้นหา element ด้วย attribute value
function FindElementByAttribute(Parent: TDOMElement; 
  const TagName, AttrName, AttrValue: string): TDOMElement;
var
  Nodes: TDOMNodeList;
  i: integer;
  El: TDOMElement;
begin
  Result := nil;
  Nodes := Parent.GetElementsByTagName(TagName);
  try
    for i := 0 to Nodes.Length - 1 do
    begin
      El := TDOMElement(Nodes[i]);
      if El.GetAttribute(AttrName) = AttrValue then
      begin
        Result := El;
        Break;
      end;
    end;
  finally
    Nodes.Free;
  end;
end;

// อ่าน text content ของ element
function GetElementText(Parent: TDOMElement; const TagName: string): string;
var
  Nodes: TDOMNodeList;
begin
  Result := '';
  Nodes := Parent.GetElementsByTagName(TagName);
  try
    if Nodes.Length > 0 then
      if Nodes[0].HasChildNodes then
        Result := Nodes[0].FirstChild.NodeValue;
  finally
    Nodes.Free;
  end;
end;

var
  Doc: TXMLDocument;
  Root: TDOMElement;
  Employee: TDOMElement;
  XMLStream: TStringStream;
begin
  XMLStream := TStringStream.Create(
    '<company>' +
    '<departments>' +
    '<department id="IT" name="Information Technology">' +
    '<employees>' +
    '<employee id="E001" active="true">' +
    '<name>ณัฐพงษ์ มีปัญญา</name>' +
    '<email>nat@company.com</email>' +
    '<salary>55000</salary>' +
    '</employee>' +
    '<employee id="E002" active="true">' +
    '<name>ปิยะ สร้างสรรค์</name>' +
    '<email>piya@company.com</email>' +
    '<salary>48000</salary>' +
    '</employee>' +
    '</employees>' +
    '</department>' +
    '</departments>' +
    '</company>'
  );
  
  ReadXMLFile(Doc, XMLStream);
  XMLStream.Free;
  
  try
    Root := Doc.DocumentElement;
    
    // ค้นหา employee ด้วย ID
    Employee := FindElementByAttribute(Root, 'employee', 'id', 'E001');
    if Assigned(Employee) then
    begin
      WriteLn('Found employee E001:');
      WriteLn('Name: ', GetElementText(Employee, 'name'));
      WriteLn('Email: ', GetElementText(Employee, 'email'));
      WriteLn('Salary: ', GetElementText(Employee, 'salary'));
      WriteLn('Active: ', Employee.GetAttribute('active'));
    end;
    
  finally
    Doc.Free;
  end;
end.
```

---

## 42.4 Writing XML

### การเขียน XML แบบสมบูรณ์

```pascal
program WriteXMLComplete;

{$mode objfpc}{$H+}

uses
  DOM, XMLWrite, SysUtils;

type
  TProduct = record
    ID: string;
    Name: string;
    Category: string;
    Price: double;
    Stock: integer;
    Description: string;
    Tags: array of string;
  end;

function CreateProductElement(Doc: TXMLDocument; const Product: TProduct): TDOMElement;
var
  El: TDOMElement;
  TagsEl: TDOMElement;
  TagEl: TDOMElement;
  Tag: string;
begin
  El := Doc.CreateElement('product');
  El.SetAttribute('id', Product.ID);
  El.SetAttribute('category', Product.Category);
  
  // Name
  El.AppendChild(Doc.CreateElement('name')).
    AppendChild(Doc.CreateTextNode(Product.Name));
  
  // Price (with currency attribute)
  with TDOMElement(El.AppendChild(Doc.CreateElement('price'))) do
  begin
    SetAttribute('currency', 'THB');
    AppendChild(Doc.CreateTextNode(FloatToStrF(Product.Price, ffFixed, 10, 2)));
  end;
  
  // Stock
  El.AppendChild(Doc.CreateElement('stock')).
    AppendChild(Doc.CreateTextNode(IntToStr(Product.Stock)));
  
  // Description (CDATA for special characters)
  if Product.Description <> '' then
  begin
    El.AppendChild(Doc.CreateElement('description')).
      AppendChild(Doc.CreateCDATASection(Product.Description));
  end;
  
  // Tags
  if Length(Product.Tags) > 0 then
  begin
    TagsEl := TDOMElement(El.AppendChild(Doc.CreateElement('tags')));
    for Tag in Product.Tags do
    begin
      TagEl := TDOMElement(TagsEl.AppendChild(Doc.CreateElement('tag')));
      TagEl.AppendChild(Doc.CreateTextNode(Tag));
    end;
  end;
  
  Result := El;
end;

procedure CreateProductCatalog(const Filename: string);
var
  Doc: TXMLDocument;
  Root: TDOMElement;
  Products: array[1..3] of TProduct;
  i: integer;
begin
  // สร้างข้อมูลสินค้า
  Products[1].ID := 'P001';
  Products[1].Name := 'แล็ปท็อป ASUS VivoBook';
  Products[1].Category := 'Electronics';
  Products[1].Price := 25000.00;
  Products[1].Stock := 50;
  Products[1].Description := 'แล็ปท็อปสำหรับใช้งานทั่วไป Intel Core i5';
  SetLength(Products[1].Tags, 3);
  Products[1].Tags[0] := 'laptop';
  Products[1].Tags[1] := 'asus';
  Products[1].Tags[2] := 'intel';
  
  Products[2].ID := 'P002';
  Products[2].Name := 'เมาส์ Logitech MX Master 3';
  Products[2].Category := 'Accessories';
  Products[2].Price := 3990.00;
  Products[2].Stock := 200;
  Products[2].Description := 'เมาส์ wireless ประสิทธิภาพสูง';
  SetLength(Products[2].Tags, 2);
  Products[2].Tags[0] := 'mouse';
  Products[2].Tags[1] := 'wireless';
  
  Products[3].ID := 'P003';
  Products[3].Name := 'คีย์บอร์ด Mechanical';
  Products[3].Category := 'Accessories';
  Products[3].Price := 2500.00;
  Products[3].Stock := 150;
  Products[3].Description := 'คีย์บอร์ด mechanical สำหรับ Gaming & Programming';
  SetLength(Products[3].Tags, 3);
  Products[3].Tags[0] := 'keyboard';
  Products[3].Tags[1] := 'mechanical';
  Products[3].Tags[2] := 'gaming';
  
  // สร้าง XML document
  Doc := TXMLDocument.Create;
  try
    // Processing instruction
    Doc.AppendChild(Doc.CreateProcessingInstruction('xml-stylesheet',
      'type="text/xsl" href="products.xsl"'));
    
    // Root element
    Root := TDOMElement(Doc.AppendChild(Doc.CreateElement('catalog')));
    Root.SetAttribute('version', '2.0');
    Root.SetAttribute('xmlns:xsi', 'http://www.w3.org/2001/XMLSchema-instance');
    
    // Comment
    Root.AppendChild(Doc.CreateComment(
      ' Product Catalog - Generated on ' + DateTimeToStr(Now) + ' '));
    
    // เพิ่ม products
    for i := 1 to 3 do
      Root.AppendChild(CreateProductElement(Doc, Products[i]));
    
    // บันทึกไฟล์
    WriteXMLFile(Doc, Filename);
    WriteLn('XML catalog saved: ', Filename);
    
  finally
    Doc.Free;
  end;
end;

begin
  CreateProductCatalog('products.xml');
  
  // แสดง content
  with TStringList.Create do
  try
    LoadFromFile('products.xml');
    WriteLn(Text);
  finally
    Free;
  end;
  
  DeleteFile('products.xml');
end.
```

---

## 42.5 XPath Queries

### การใช้ XPath

```pascal
program XPathQueries;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, XPath, SysUtils;

var
  Doc: TXMLDocument;
  XMLStream: TStringStream;
  XPathResult: TXPathVariable;
  NodeSet: TNodeSet;
  i: integer;
begin
  XMLStream := TStringStream.Create(
    '<?xml version="1.0"?>' +
    '<library>' +
    '<books>' +
    '<book category="fiction" isbn="978-1">' +
    '<title>The Great Adventure</title>' +
    '<author>John Smith</author>' +
    '<year>2020</year>' +
    '<price>350.00</price>' +
    '</book>' +
    '<book category="science" isbn="978-2">' +
    '<title>Physics Explained</title>' +
    '<author>Jane Doe</author>' +
    '<year>2021</year>' +
    '<price>450.00</price>' +
    '</book>' +
    '<book category="fiction" isbn="978-3">' +
    '<title>Mystery in the Dark</title>' +
    '<author>Bob Wilson</author>' +
    '<year>2019</year>' +
    '<price>280.00</price>' +
    '</book>' +
    '</books>' +
    '</library>'
  );
  
  ReadXMLFile(Doc, XMLStream);
  XMLStream.Free;
  
  try
    WriteLn('=== XPath Queries Demo ===');
    WriteLn('');
    
    // Query 1: หา title ทั้งหมด
    WriteLn('1. All book titles:');
    XPathResult := EvaluateXPathExpression('//book/title', Doc.DocumentElement);
    if XPathResult.AsNodeSet.Count > 0 then
    begin
      NodeSet := XPathResult.AsNodeSet;
      for i := 0 to NodeSet.Count - 1 do
        if TDOMNode(NodeSet[i]).HasChildNodes then
          WriteLn('  - ', TDOMNode(NodeSet[i]).FirstChild.NodeValue);
    end;
    XPathResult.Free;
    
    // Query 2: หนังสือ fiction
    WriteLn('');
    WriteLn('2. Fiction books:');
    XPathResult := EvaluateXPathExpression(
      '//book[@category="fiction"]/title', 
      Doc.DocumentElement
    );
    NodeSet := XPathResult.AsNodeSet;
    for i := 0 to NodeSet.Count - 1 do
      if TDOMNode(NodeSet[i]).HasChildNodes then
        WriteLn('  - ', TDOMNode(NodeSet[i]).FirstChild.NodeValue);
    XPathResult.Free;
    
    // Query 3: นับจำนวนหนังสือ
    WriteLn('');
    WriteLn('3. Book count:');
    XPathResult := EvaluateXPathExpression('count(//book)', Doc.DocumentElement);
    WriteLn('  Total: ', Round(XPathResult.AsNumber));
    XPathResult.Free;
    
    // Query 4: หนังสือที่ราคา > 300
    WriteLn('');
    WriteLn('4. Books with price > 300:');
    XPathResult := EvaluateXPathExpression(
      '//book[price>300]/title',
      Doc.DocumentElement
    );
    NodeSet := XPathResult.AsNodeSet;
    for i := 0 to NodeSet.Count - 1 do
      if TDOMNode(NodeSet[i]).HasChildNodes then
        WriteLn('  - ', TDOMNode(NodeSet[i]).FirstChild.NodeValue);
    XPathResult.Free;
    
  finally
    Doc.Free;
  end;
end.
```

---

## 42.6 XML Schema Validation

### การ Validate XML กับ Schema

```pascal
program XMLSchemaValidation;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, SysUtils;

type
  TFieldType = (ftString, ftInteger, ftFloat, ftBoolean, ftDate);
  
  TFieldRule = record
    Name: string;
    Required: boolean;
    FieldType: TFieldType;
    MinLength: integer;
    MaxLength: integer;
    MinValue: double;
    MaxValue: double;
  end;

  TXMLValidator = class
  private
    FRules: array of TFieldRule;
    FErrors: TStringList;
    
    function ValidateField(const FieldName, Value: string; 
      const Rule: TFieldRule): boolean;
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure AddRule(const FieldName: string; Required: boolean;
      FieldType: TFieldType; MinLen: integer = 0; MaxLen: integer = MaxInt;
      MinVal: double = -MaxDouble; MaxVal: double = MaxDouble);
    
    function Validate(Element: TDOMElement): boolean;
    
    property Errors: TStringList read FErrors;
  end;

constructor TXMLValidator.Create;
begin
  inherited Create;
  SetLength(FRules, 0);
  FErrors := TStringList.Create;
end;

destructor TXMLValidator.Destroy;
begin
  FErrors.Free;
  inherited Destroy;
end;

procedure TXMLValidator.AddRule(const FieldName: string; Required: boolean;
  FieldType: TFieldType; MinLen, MaxLen: integer; MinVal, MaxVal: double);
var
  Rule: TFieldRule;
  Len: integer;
begin
  Rule.Name := FieldName;
  Rule.Required := Required;
  Rule.FieldType := FieldType;
  Rule.MinLength := MinLen;
  Rule.MaxLength := MaxLen;
  Rule.MinValue := MinVal;
  Rule.MaxValue := MaxVal;
  
  Len := Length(FRules);
  SetLength(FRules, Len + 1);
  FRules[Len] := Rule;
end;

function TXMLValidator.ValidateField(const FieldName, Value: string;
  const Rule: TFieldRule): boolean;
var
  IntVal: integer;
  FloatVal: double;
  Code: integer;
begin
  Result := True;
  
  // ตรวจสอบ length
  if (Rule.FieldType = ftString) then
  begin
    if Length(Value) < Rule.MinLength then
    begin
      FErrors.Add(Format('Field "%s": ยาวน้อยกว่า %d ตัวอักษร', 
        [FieldName, Rule.MinLength]));
      Result := False;
    end;
    
    if (Rule.MaxLength < MaxInt) and (Length(Value) > Rule.MaxLength) then
    begin
      FErrors.Add(Format('Field "%s": ยาวเกิน %d ตัวอักษร', 
        [FieldName, Rule.MaxLength]));
      Result := False;
    end;
  end;
  
  // ตรวจสอบ integer
  if Rule.FieldType = ftInteger then
  begin
    Val(Value, IntVal, Code);
    if Code <> 0 then
    begin
      FErrors.Add(Format('Field "%s": ต้องเป็น integer', [FieldName]));
      Result := False;
    end
    else
    begin
      if IntVal < Rule.MinValue then
      begin
        FErrors.Add(Format('Field "%s": ต้องมากกว่า %g', [FieldName, Rule.MinValue]));
        Result := False;
      end;
      if IntVal > Rule.MaxValue then
      begin
        FErrors.Add(Format('Field "%s": ต้องน้อยกว่า %g', [FieldName, Rule.MaxValue]));
        Result := False;
      end;
    end;
  end;
  
  // ตรวจสอบ float
  if Rule.FieldType = ftFloat then
  begin
    Val(Value, FloatVal, Code);
    if Code <> 0 then
    begin
      FErrors.Add(Format('Field "%s": ต้องเป็น number', [FieldName]));
      Result := False;
    end;
  end;
  
  // ตรวจสอบ boolean
  if Rule.FieldType = ftBoolean then
    if not (Value = 'true') and not (Value = 'false') and
       not (Value = '0') and not (Value = '1') then
    begin
      FErrors.Add(Format('Field "%s": ต้องเป็น true/false', [FieldName]));
      Result := False;
    end;
end;

function TXMLValidator.Validate(Element: TDOMElement): boolean;
var
  Rule: TFieldRule;
  Nodes: TDOMNodeList;
  FieldValue: string;
  i: integer;
begin
  FErrors.Clear;
  Result := True;
  
  for Rule in FRules do
  begin
    Nodes := Element.GetElementsByTagName(Rule.Name);
    try
      if Nodes.Length = 0 then
      begin
        if Rule.Required then
        begin
          FErrors.Add(Format('Required field "%s" is missing', [Rule.Name]));
          Result := False;
        end;
      end
      else
      begin
        // อ่าน text value
        if Nodes[0].HasChildNodes then
          FieldValue := Nodes[0].FirstChild.NodeValue
        else
          FieldValue := '';
          
        // ตรวจสอบว่า required field ไม่ว่าง
        if Rule.Required and (Trim(FieldValue) = '') then
        begin
          FErrors.Add(Format('Field "%s" ต้องไม่ว่าง', [Rule.Name]));
          Result := False;
        end
        else if Trim(FieldValue) <> '' then
        begin
          if not ValidateField(Rule.Name, Trim(FieldValue), Rule) then
            Result := False;
        end;
      end;
    finally
      Nodes.Free;
    end;
  end;
end;

var
  Validator: TXMLValidator;
  Doc: TXMLDocument;
  XMLStream: TStringStream;
  Root: TDOMElement;
  PersonNode: TDOMNode;
  i: integer;
begin
  // สร้าง validator กับ rules
  Validator := TXMLValidator.Create;
  try
    Validator.AddRule('name', True, ftString, 2, 100);
    Validator.AddRule('email', True, ftString, 5, 100);
    Validator.AddRule('age', True, ftInteger, 0, 0, 1, 150);
    Validator.AddRule('salary', False, ftFloat);
    Validator.AddRule('active', False, ftBoolean);
    
    // Test XML
    XMLStream := TStringStream.Create(
      '<people>' +
      '<person>' +
      '<name>สมชาย ใจดี</name>' +
      '<email>somchai@test.com</email>' +
      '<age>30</age>' +
      '<salary>45000.00</salary>' +
      '<active>true</active>' +
      '</person>' +
      '<person>' +
      '<name>X</name>' +
      '<email></email>' +
      '<age>abc</age>' +
      '<active>maybe</active>' +
      '</person>' +
      '</people>'
    );
    
    ReadXMLFile(Doc, XMLStream);
    XMLStream.Free;
    
    try
      Root := Doc.DocumentElement;
      WriteLn('Validating XML persons...');
      WriteLn('');
      
      i := 0;
      PersonNode := Root.FirstChild;
      while Assigned(PersonNode) do
      begin
        if PersonNode.NodeType = ELEMENT_NODE then
        begin
          Inc(i);
          WriteLn('--- Person ', i, ' ---');
          
          if Validator.Validate(TDOMElement(PersonNode)) then
            WriteLn('Valid!')
          else
          begin
            WriteLn('Invalid! Errors:');
            for var Err in Validator.Errors do
              WriteLn('  - ', Err);
          end;
          WriteLn('');
        end;
        PersonNode := PersonNode.NextSibling;
      end;
      
    finally
      Doc.Free;
    end;
  finally
    Validator.Free;
  end;
end.
```

---

## 42.7 XSLT Transformation

### การใช้ XSLT

```pascal
program XSLTTransformation;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, XMLWrite, SysUtils;

// สาธิตการสร้าง HTML จาก XML โดยไม่ใช้ XSLT library
// (เพราะ FPC ไม่มี built-in XSLT processor)
procedure TransformXMLToHTML(const XMLContent, OutputFile: string);
var
  Doc: TXMLDocument;
  XMLStream: TStringStream;
  Root, Node: TDOMElement;
  NodeList: TDOMNodeList;
  HTML: TStringList;
  i: integer;
  Name, Price, Category: string;
begin
  XMLStream := TStringStream.Create(XMLContent);
  ReadXMLFile(Doc, XMLStream);
  XMLStream.Free;
  
  HTML := TStringList.Create;
  try
    HTML.Add('<!DOCTYPE html>');
    HTML.Add('<html lang="th">');
    HTML.Add('<head>');
    HTML.Add('  <meta charset="UTF-8">');
    HTML.Add('  <title>Product Catalog</title>');
    HTML.Add('  <style>');
    HTML.Add('    body { font-family: Arial, sans-serif; }');
    HTML.Add('    table { border-collapse: collapse; width: 100%; }');
    HTML.Add('    th, td { border: 1px solid #ddd; padding: 8px; }');
    HTML.Add('    th { background-color: #4CAF50; color: white; }');
    HTML.Add('    tr:nth-child(even) { background-color: #f2f2f2; }');
    HTML.Add('  </style>');
    HTML.Add('</head>');
    HTML.Add('<body>');
    HTML.Add('<h1>รายการสินค้า</h1>');
    HTML.Add('<table>');
    HTML.Add('  <tr>');
    HTML.Add('    <th>ชื่อสินค้า</th>');
    HTML.Add('    <th>หมวดหมู่</th>');
    HTML.Add('    <th>ราคา</th>');
    HTML.Add('  </tr>');
    
    try
      Root := Doc.DocumentElement;
      NodeList := Root.GetElementsByTagName('product');
      try
        for i := 0 to NodeList.Length - 1 do
        begin
          Node := TDOMElement(NodeList[i]);
          Category := Node.GetAttribute('category');
          
          // อ่านชื่อสินค้า
          var NameNodes := Node.GetElementsByTagName('name');
          try
            if NameNodes.Length > 0 then
              Name := NameNodes[0].FirstChild.NodeValue
            else
              Name := '';
          finally
            NameNodes.Free;
          end;
          
          // อ่านราคา
          var PriceNodes := Node.GetElementsByTagName('price');
          try
            if PriceNodes.Length > 0 then
              Price := PriceNodes[0].FirstChild.NodeValue
            else
              Price := '';
          finally
            PriceNodes.Free;
          end;
          
          HTML.Add('  <tr>');
          HTML.Add(Format('    <td>%s</td>', [Name]));
          HTML.Add(Format('    <td>%s</td>', [Category]));
          HTML.Add(Format('    <td>%s บาท</td>', [Price]));
          HTML.Add('  </tr>');
        end;
      finally
        NodeList.Free;
      end;
    finally
      Doc.Free;
    end;
    
    HTML.Add('</table>');
    HTML.Add('</body>');
    HTML.Add('</html>');
    
    HTML.SaveToFile(OutputFile);
    WriteLn('HTML file created: ', OutputFile);
    
  finally
    HTML.Free;
  end;
end;

var
  XMLContent: string;
begin
  XMLContent :=
    '<?xml version="1.0"?>' +
    '<catalog>' +
    '<product id="001" category="Electronics">' +
    '<name>MacBook Pro 14"</name>' +
    '<price>89900</price>' +
    '</product>' +
    '<product id="002" category="Accessories">' +
    '<name>AirPods Pro</name>' +
    '<price>9990</price>' +
    '</product>' +
    '<product id="003" category="Tablet">' +
    '<name>iPad Air</name>' +
    '<price>21900</price>' +
    '</product>' +
    '</catalog>';
    
  TransformXMLToHTML(XMLContent, 'products.html');
  WriteLn('XSLT-like transformation completed');
  
  // แสดง HTML content
  with TStringList.Create do
  try
    LoadFromFile('products.html');
    WriteLn(Text);
  finally
    Free;
  end;
  
  DeleteFile('products.html');
end.
```

---

## 42.8 RSS Feeds

### การ Parse RSS Feed

```pascal
program RSSFeedParser;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, SysUtils, Classes;

type
  TRSSItem = record
    Title: string;
    Description: string;
    Link: string;
    PubDate: string;
    Author: string;
    Category: string;
    GUID: string;
  end;
  
  TRSSFeed = class
  private
    FTitle: string;
    FDescription: string;
    FLink: string;
    FLanguage: string;
    FItems: array of TRSSItem;
    FItemCount: integer;
    
    function GetElementText(Parent: TDOMNode; const TagName: string): string;
    procedure ParseChannel(Channel: TDOMElement);
    procedure ParseItem(Item: TDOMElement);
  public
    constructor Create;
    
    function ParseFromString(const XMLContent: string): boolean;
    function ParseFromFile(const FileName: string): boolean;
    
    procedure Display;
    
    property Title: string read FTitle;
    property Description: string read FDescription;
    property Link: string read FLink;
    property ItemCount: integer read FItemCount;
    function GetItem(Index: integer): TRSSItem;
  end;

constructor TRSSFeed.Create;
begin
  inherited Create;
  FItemCount := 0;
  SetLength(FItems, 0);
end;

function TRSSFeed.GetElementText(Parent: TDOMNode; const TagName: string): string;
var
  Node: TDOMNode;
  i: integer;
begin
  Result := '';
  for i := 0 to Parent.ChildNodes.Length - 1 do
  begin
    Node := Parent.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = TagName) then
    begin
      if Node.HasChildNodes then
        Result := Node.FirstChild.NodeValue;
      Break;
    end;
  end;
end;

procedure TRSSFeed.ParseItem(Item: TDOMElement);
var
  RSSItem: TRSSItem;
  Idx: integer;
begin
  RSSItem.Title := GetElementText(Item, 'title');
  RSSItem.Description := GetElementText(Item, 'description');
  RSSItem.Link := GetElementText(Item, 'link');
  RSSItem.PubDate := GetElementText(Item, 'pubDate');
  RSSItem.Author := GetElementText(Item, 'author');
  RSSItem.Category := GetElementText(Item, 'category');
  RSSItem.GUID := GetElementText(Item, 'guid');
  
  Idx := Length(FItems);
  SetLength(FItems, Idx + 1);
  FItems[Idx] := RSSItem;
  Inc(FItemCount);
end;

procedure TRSSFeed.ParseChannel(Channel: TDOMElement);
var
  i: integer;
  Node: TDOMNode;
begin
  FTitle := GetElementText(Channel, 'title');
  FDescription := GetElementText(Channel, 'description');
  FLink := GetElementText(Channel, 'link');
  FLanguage := GetElementText(Channel, 'language');
  
  // Parse items
  for i := 0 to Channel.ChildNodes.Length - 1 do
  begin
    Node := Channel.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = 'item') then
      ParseItem(TDOMElement(Node));
  end;
end;

function TRSSFeed.ParseFromString(const XMLContent: string): boolean;
var
  Doc: TXMLDocument;
  XMLStream: TStringStream;
  Root: TDOMElement;
  i: integer;
  Node: TDOMNode;
begin
  Result := False;
  FItemCount := 0;
  SetLength(FItems, 0);
  
  try
    XMLStream := TStringStream.Create(XMLContent);
    ReadXMLFile(Doc, XMLStream);
    XMLStream.Free;
    
    try
      Root := Doc.DocumentElement;
      
      // หา channel element
      for i := 0 to Root.ChildNodes.Length - 1 do
      begin
        Node := Root.ChildNodes[i];
        if (Node.NodeType = ELEMENT_NODE) and 
           (TDOMElement(Node).TagName = 'channel') then
        begin
          ParseChannel(TDOMElement(Node));
          Break;
        end;
      end;
      
      Result := True;
    finally
      Doc.Free;
    end;
  except
    on E: Exception do
      WriteLn('RSS Parse Error: ', E.Message);
  end;
end;

function TRSSFeed.ParseFromFile(const FileName: string): boolean;
var
  Content: TStringList;
begin
  Content := TStringList.Create;
  try
    Content.LoadFromFile(FileName);
    Result := ParseFromString(Content.Text);
  finally
    Content.Free;
  end;
end;

procedure TRSSFeed.Display;
var
  i: integer;
begin
  WriteLn('=== RSS Feed ===');
  WriteLn('Title: ', FTitle);
  WriteLn('Description: ', FDescription);
  WriteLn('Link: ', FLink);
  WriteLn('Items: ', FItemCount);
  WriteLn('');
  
  for i := 0 to FItemCount - 1 do
  begin
    WriteLn('--- Item ', i + 1, ' ---');
    WriteLn('Title: ', FItems[i].Title);
    WriteLn('Date: ', FItems[i].PubDate);
    WriteLn('Author: ', FItems[i].Author);
    if FItems[i].Category <> '' then
      WriteLn('Category: ', FItems[i].Category);
    // แสดง description เพียง 100 ตัวอักษรแรก
    if Length(FItems[i].Description) > 100 then
      WriteLn('Description: ', Copy(FItems[i].Description, 1, 100), '...')
    else
      WriteLn('Description: ', FItems[i].Description);
    WriteLn('Link: ', FItems[i].Link);
    WriteLn('');
  end;
end;

function TRSSFeed.GetItem(Index: integer): TRSSItem;
begin
  if (Index >= 0) and (Index < FItemCount) then
    Result := FItems[Index]
  else
    raise Exception.CreateFmt('Index %d out of range (0-%d)', [Index, FItemCount - 1]);
end;

// สร้าง RSS Feed
procedure CreateRSSFeed(const FileName: string);
var
  Doc: TXMLDocument;
  RSS, Channel, Item: TDOMElement;
  
  procedure AddTextElement(Parent: TDOMElement; const Tag, Text: string);
  var
    El: TDOMElement;
  begin
    El := TDOMElement(Parent.AppendChild(Doc.CreateElement(Tag)));
    El.AppendChild(Doc.CreateTextNode(Text));
  end;
  
begin
  Doc := TXMLDocument.Create;
  try
    RSS := TDOMElement(Doc.AppendChild(Doc.CreateElement('rss')));
    RSS.SetAttribute('version', '2.0');
    
    Channel := TDOMElement(RSS.AppendChild(Doc.CreateElement('channel')));
    
    AddTextElement(Channel, 'title', 'บล็อกเทคโนโลยี');
    AddTextElement(Channel, 'description', 'ข่าวสารด้านเทคโนโลยีและการพัฒนาซอฟต์แวร์');
    AddTextElement(Channel, 'link', 'https://techblog.example.com');
    AddTextElement(Channel, 'language', 'th');
    AddTextElement(Channel, 'lastBuildDate', 'Mon, 15 Jan 2024 10:00:00 +0700');
    
    // Item 1
    Item := TDOMElement(Channel.AppendChild(Doc.CreateElement('item')));
    AddTextElement(Item, 'title', 'Lazarus Pascal ยังมีชีวิตและน่าสนใจมาก');
    AddTextElement(Item, 'description', 'บทความแนะนำการเขียนโปรแกรมด้วย Lazarus Pascal');
    AddTextElement(Item, 'link', 'https://techblog.example.com/lazarus-pascal');
    AddTextElement(Item, 'author', 'webmaster@techblog.example.com');
    AddTextElement(Item, 'pubDate', 'Mon, 15 Jan 2024 08:00:00 +0700');
    AddTextElement(Item, 'category', 'Programming');
    AddTextElement(Item, 'guid', 'https://techblog.example.com/lazarus-pascal');
    
    // Item 2
    Item := TDOMElement(Channel.AppendChild(Doc.CreateElement('item')));
    AddTextElement(Item, 'title', 'การใช้ JSON ใน Free Pascal');
    AddTextElement(Item, 'description', 'วิธีการ parse และสร้าง JSON ใน FPC/Lazarus');
    AddTextElement(Item, 'link', 'https://techblog.example.com/json-fpc');
    AddTextElement(Item, 'author', 'webmaster@techblog.example.com');
    AddTextElement(Item, 'pubDate', 'Wed, 10 Jan 2024 10:00:00 +0700');
    AddTextElement(Item, 'category', 'Programming');
    AddTextElement(Item, 'guid', 'https://techblog.example.com/json-fpc');
    
    WriteXMLFile(Doc, FileName);
    WriteLn('RSS feed created: ', FileName);
  finally
    Doc.Free;
  end;
end;

var
  Feed: TRSSFeed;
  RSSFile: string;
begin
  RSSFile := 'techblog.rss';
  
  // สร้าง RSS file
  CreateRSSFeed(RSSFile);
  
  // Parse RSS
  Feed := TRSSFeed.Create;
  try
    if Feed.ParseFromFile(RSSFile) then
      Feed.Display
    else
      WriteLn('Failed to parse RSS feed');
  finally
    Feed.Free;
  end;
  
  DeleteFile(RSSFile);
end.
```

---

## 42.9 Configuration XML

### การจัดการ App Configuration ด้วย XML

```pascal
program XMLConfigManager;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, XMLWrite, SysUtils;

type
  TXMLConfig = class
  private
    FDoc: TXMLDocument;
    FRoot: TDOMElement;
    FFilePath: string;
    
    function GetOrCreateSection(const SectionName: string): TDOMElement;
    function GetTextNode(Parent: TDOMElement; const Key: string): TDOMText;
    procedure SetTextNode(Parent: TDOMElement; const Key, Value: string);
  public
    constructor Create(const FilePath: string);
    destructor Destroy; override;
    
    procedure Load;
    procedure Save;
    
    function ReadString(const Section, Key, Default: string): string;
    function ReadInteger(const Section, Key: string; Default: integer): integer;
    function ReadBoolean(const Section, Key: string; Default: boolean): boolean;
    function ReadFloat(const Section, Key: string; Default: double): double;
    
    procedure WriteString(const Section, Key, Value: string);
    procedure WriteInteger(const Section, Key: string; Value: integer);
    procedure WriteBoolean(const Section, Key: string; Value: boolean);
    procedure WriteFloat(const Section, Key: string; Value: double);
    
    procedure DeleteKey(const Section, Key: string);
    procedure DeleteSection(const Section: string);
  end;

constructor TXMLConfig.Create(const FilePath: string);
begin
  inherited Create;
  FFilePath := FilePath;
  FDoc := nil;
  Load;
end;

destructor TXMLConfig.Destroy;
begin
  if Assigned(FDoc) then
    FDoc.Free;
  inherited Destroy;
end;

procedure TXMLConfig.Load;
begin
  if Assigned(FDoc) then
    FDoc.Free;
    
  if FileExists(FFilePath) then
  begin
    try
      ReadXMLFile(FDoc, FFilePath);
      FRoot := FDoc.DocumentElement;
    except
      FDoc := nil;
    end;
  end;
  
  if not Assigned(FDoc) then
  begin
    FDoc := TXMLDocument.Create;
    FRoot := TDOMElement(FDoc.AppendChild(FDoc.CreateElement('configuration')));
  end;
end;

procedure TXMLConfig.Save;
begin
  if Assigned(FDoc) then
    WriteXMLFile(FDoc, FFilePath);
end;

function TXMLConfig.GetOrCreateSection(const SectionName: string): TDOMElement;
var
  i: integer;
  Node: TDOMNode;
begin
  // ค้นหา section
  for i := 0 to FRoot.ChildNodes.Length - 1 do
  begin
    Node := FRoot.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = SectionName) then
    begin
      Result := TDOMElement(Node);
      Exit;
    end;
  end;
  
  // สร้างถ้าไม่มี
  Result := TDOMElement(FRoot.AppendChild(FDoc.CreateElement(SectionName)));
end;

function TXMLConfig.GetTextNode(Parent: TDOMElement; const Key: string): TDOMText;
var
  i: integer;
  Node: TDOMNode;
begin
  Result := nil;
  for i := 0 to Parent.ChildNodes.Length - 1 do
  begin
    Node := Parent.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = Key) then
    begin
      if Node.HasChildNodes and (Node.FirstChild.NodeType = TEXT_NODE) then
        Result := TDOMText(Node.FirstChild);
      Exit;
    end;
  end;
end;

procedure TXMLConfig.SetTextNode(Parent: TDOMElement; const Key, Value: string);
var
  i: integer;
  Node: TDOMNode;
  El: TDOMElement;
  TextNode: TDOMText;
begin
  // ค้นหา element ที่มีอยู่
  for i := 0 to Parent.ChildNodes.Length - 1 do
  begin
    Node := Parent.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = Key) then
    begin
      if Node.HasChildNodes and (Node.FirstChild.NodeType = TEXT_NODE) then
        TDOMText(Node.FirstChild).NodeValue := Value
      else
        Node.AppendChild(FDoc.CreateTextNode(Value));
      Exit;
    end;
  end;
  
  // สร้าง element ใหม่
  El := TDOMElement(Parent.AppendChild(FDoc.CreateElement(Key)));
  El.AppendChild(FDoc.CreateTextNode(Value));
end;

function TXMLConfig.ReadString(const Section, Key, Default: string): string;
var
  SectionEl: TDOMElement;
  i: integer;
  Node: TDOMNode;
begin
  Result := Default;
  
  // ค้นหา section
  for i := 0 to FRoot.ChildNodes.Length - 1 do
  begin
    Node := FRoot.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = Section) then
    begin
      SectionEl := TDOMElement(Node);
      // ค้นหา key
      var TextNode := GetTextNode(SectionEl, Key);
      if Assigned(TextNode) then
        Result := TextNode.NodeValue;
      Exit;
    end;
  end;
end;

function TXMLConfig.ReadInteger(const Section, Key: string; Default: integer): integer;
var
  Str: string;
  Code: integer;
begin
  Str := ReadString(Section, Key, IntToStr(Default));
  Val(Str, Result, Code);
  if Code <> 0 then
    Result := Default;
end;

function TXMLConfig.ReadBoolean(const Section, Key: string; Default: boolean): boolean;
var
  Str: string;
begin
  Str := LowerCase(ReadString(Section, Key, BoolToStr(Default, 'true', 'false')));
  Result := (Str = 'true') or (Str = '1') or (Str = 'yes');
end;

function TXMLConfig.ReadFloat(const Section, Key: string; Default: double): double;
var
  Str: string;
  Code: integer;
begin
  Str := ReadString(Section, Key, FloatToStr(Default));
  Val(Str, Result, Code);
  if Code <> 0 then
    Result := Default;
end;

procedure TXMLConfig.WriteString(const Section, Key, Value: string);
var
  SectionEl: TDOMElement;
begin
  SectionEl := GetOrCreateSection(Section);
  SetTextNode(SectionEl, Key, Value);
end;

procedure TXMLConfig.WriteInteger(const Section, Key: string; Value: integer);
begin
  WriteString(Section, Key, IntToStr(Value));
end;

procedure TXMLConfig.WriteBoolean(const Section, Key: string; Value: boolean);
begin
  WriteString(Section, Key, BoolToStr(Value, 'true', 'false'));
end;

procedure TXMLConfig.WriteFloat(const Section, Key: string; Value: double);
begin
  WriteString(Section, Key, FloatToStr(Value));
end;

procedure TXMLConfig.DeleteKey(const Section, Key: string);
var
  i: integer;
  Node: TDOMNode;
  SectionNode: TDOMNode;
begin
  for i := 0 to FRoot.ChildNodes.Length - 1 do
  begin
    SectionNode := FRoot.ChildNodes[i];
    if (SectionNode.NodeType = ELEMENT_NODE) and 
       (TDOMElement(SectionNode).TagName = Section) then
    begin
      var j := 0;
      while j < SectionNode.ChildNodes.Length do
      begin
        Node := SectionNode.ChildNodes[j];
        if (Node.NodeType = ELEMENT_NODE) and 
           (TDOMElement(Node).TagName = Key) then
        begin
          SectionNode.RemoveChild(Node);
          Node.Free;
          Break;
        end;
        Inc(j);
      end;
      Exit;
    end;
  end;
end;

procedure TXMLConfig.DeleteSection(const Section: string);
var
  i: integer;
  Node: TDOMNode;
begin
  for i := 0 to FRoot.ChildNodes.Length - 1 do
  begin
    Node := FRoot.ChildNodes[i];
    if (Node.NodeType = ELEMENT_NODE) and 
       (TDOMElement(Node).TagName = Section) then
    begin
      FRoot.RemoveChild(Node);
      Node.Free;
      Exit;
    end;
  end;
end;

var
  Config: TXMLConfig;
  ConfigFile: string;
begin
  ConfigFile := 'app_config.xml';
  
  Config := TXMLConfig.Create(ConfigFile);
  try
    // เขียนการตั้งค่า
    Config.WriteString('database', 'host', 'localhost');
    Config.WriteInteger('database', 'port', 5432);
    Config.WriteString('database', 'name', 'myapp');
    Config.WriteString('database', 'user', 'admin');
    
    Config.WriteString('server', 'host', '0.0.0.0');
    Config.WriteInteger('server', 'port', 8080);
    Config.WriteBoolean('server', 'debug', True);
    Config.WriteFloat('server', 'version', 2.1);
    
    Config.WriteString('ui', 'theme', 'dark');
    Config.WriteString('ui', 'language', 'th');
    Config.WriteInteger('ui', 'fontSize', 14);
    
    Config.Save;
    WriteLn('Config saved');
    
    // อ่านการตั้งค่ากลับ
    WriteLn('');
    WriteLn('=== Reading Config ===');
    WriteLn('DB Host: ', Config.ReadString('database', 'host', 'localhost'));
    WriteLn('DB Port: ', Config.ReadInteger('database', 'port', 5432));
    WriteLn('Server Port: ', Config.ReadInteger('server', 'port', 8080));
    WriteLn('Debug Mode: ', Config.ReadBoolean('server', 'debug', False));
    WriteLn('Version: ', Config.ReadFloat('server', 'version', 1.0):0:1);
    WriteLn('Theme: ', Config.ReadString('ui', 'theme', 'light'));
    WriteLn('Language: ', Config.ReadString('ui', 'language', 'en'));
    
    // อ่านค่าที่ไม่มี (จะได้ default)
    WriteLn('Missing Key: ', Config.ReadString('ui', 'notExist', 'default_value'));
    
    // แสดง XML file content
    WriteLn('');
    WriteLn('=== XML Content ===');
    with TStringList.Create do
    try
      LoadFromFile(ConfigFile);
      WriteLn(Text);
    finally
      Free;
    end;
    
  finally
    Config.Free;
  end;
  
  DeleteFile(ConfigFile);
end.
```

---

## 42.10 Complete Example: RSS Reader

```pascal
program RSSReaderComplete;

{$mode objfpc}{$H+}

uses
  DOM, XMLRead, XMLWrite, SysUtils, Classes, fphttpclient;

type
  TRSSItem = record
    Title: string;
    Description: string;
    Link: string;
    PubDate: string;
    Category: string;
    Author: string;
  end;
  
  TRSSChannel = record
    Title: string;
    Description: string;
    Link: string;
    Language: string;
    PubDate: string;
    ImageURL: string;
  end;

  TRSSReader = class
  private
    FChannel: TRSSChannel;
    FItems: TList;
    
    function CreateItem: PRSSItem;
    function GetElementContent(Node: TDOMNode; const TagName: string): string;
    procedure ParseChannel(ChannelNode: TDOMNode);
    procedure ParseItem(ItemNode: TDOMNode);
    procedure ClearItems;
  public
    constructor Create;
    destructor Destroy; override;
    
    // โหลด RSS จาก string content
    function LoadFromString(const XMLContent: string): boolean;
    // โหลด RSS จาก file
    function LoadFromFile(const FileName: string): boolean;
    // โหลด RSS จาก URL (ต้องมี fphttpclient)
    function LoadFromURL(const URL: string): boolean;
    
    // Export เป็น format ต่างๆ
    procedure ExportToText(const FileName: string);
    procedure ExportToHTML(const FileName: string);
    procedure ExportToXMLSummary(const FileName: string);
    
    // แสดงผล
    procedure Display;
    procedure DisplayItem(Index: integer);
    
    property Channel: TRSSChannel read FChannel;
    function GetItemCount: integer;
    function GetItem(Index: integer): TRSSItem;
  end;

{ TRSSItem helper }
type
  PRSSItem = ^TRSSItem;

constructor TRSSReader.Create;
begin
  inherited Create;
  FItems := TList.Create;
  FillChar(FChannel, SizeOf(FChannel), 0);
end;

destructor TRSSReader.Destroy;
begin
  ClearItems;
  FItems.Free;
  inherited Destroy;
end;

function TRSSReader.CreateItem: PRSSItem;
begin
  New(Result);
  FillChar(Result^, SizeOf(TRSSItem), 0);
end;

procedure TRSSReader.ClearItems;
var
  i: integer;
  Item: PRSSItem;
begin
  for i := 0 to FItems.Count - 1 do
  begin
    Item := PRSSItem(FItems[i]);
    Dispose(Item);
  end;
  FItems.Clear;
end;

function TRSSReader.GetItemCount: integer;
begin
  Result := FItems.Count;
end;

function TRSSReader.GetItem(Index: integer): TRSSItem;
begin
  Result := PRSSItem(FItems[Index])^;
end;

function TRSSReader.GetElementContent(Node: TDOMNode; const TagName: string): string;
var
  i: integer;
  Child: TDOMNode;
begin
  Result := '';
  for i := 0 to Node.ChildNodes.Length - 1 do
  begin
    Child := Node.ChildNodes[i];
    if (Child.NodeType = ELEMENT_NODE) and
       (TDOMElement(Child).TagName = TagName) then
    begin
      if Child.HasChildNodes then
        Result := Child.FirstChild.NodeValue;
      Break;
    end;
  end;
end;

procedure TRSSReader.ParseChannel(ChannelNode: TDOMNode);
begin
  FChannel.Title := GetElementContent(ChannelNode, 'title');
  FChannel.Description := GetElementContent(ChannelNode, 'description');
  FChannel.Link := GetElementContent(ChannelNode, 'link');
  FChannel.Language := GetElementContent(ChannelNode, 'language');
  FChannel.PubDate := GetElementContent(ChannelNode, 'lastBuildDate');
  
  // ค้นหา image URL
  var ImageNode: TDOMNode := nil;
  var j: integer;
  for j := 0 to ChannelNode.ChildNodes.Length - 1 do
  begin
    var Child := ChannelNode.ChildNodes[j];
    if (Child.NodeType = ELEMENT_NODE) and
       (TDOMElement(Child).TagName = 'image') then
    begin
      ImageNode := Child;
      Break;
    end;
  end;
  
  if Assigned(ImageNode) then
    FChannel.ImageURL := GetElementContent(ImageNode, 'url');
end;

procedure TRSSReader.ParseItem(ItemNode: TDOMNode);
var
  Item: PRSSItem;
begin
  Item := CreateItem;
  Item^.Title := GetElementContent(ItemNode, 'title');
  Item^.Description := GetElementContent(ItemNode, 'description');
  Item^.Link := GetElementContent(ItemNode, 'link');
  Item^.PubDate := GetElementContent(ItemNode, 'pubDate');
  Item^.Category := GetElementContent(ItemNode, 'category');
  Item^.Author := GetElementContent(ItemNode, 'author');
  FItems.Add(Item);
end;

function TRSSReader.LoadFromString(const XMLContent: string): boolean;
var
  Doc: TXMLDocument;
  XMLStream: TStringStream;
  Root: TDOMElement;
  i: integer;
  Node: TDOMNode;
  ChannelNode: TDOMNode;
begin
  Result := False;
  ClearItems;
  
  try
    XMLStream := TStringStream.Create(XMLContent);
    ReadXMLFile(Doc, XMLStream);
    XMLStream.Free;
    
    try
      Root := Doc.DocumentElement;
      
      // ค้นหา channel
      ChannelNode := nil;
      for i := 0 to Root.ChildNodes.Length - 1 do
      begin
        Node := Root.ChildNodes[i];
        if (Node.NodeType = ELEMENT_NODE) and
           (TDOMElement(Node).TagName = 'channel') then
        begin
          ChannelNode := Node;
          Break;
        end;
      end;
      
      if not Assigned(ChannelNode) then
      begin
        WriteLn('No channel element found');
        Exit;
      end;
      
      ParseChannel(ChannelNode);
      
      // Parse items
      for i := 0 to ChannelNode.ChildNodes.Length - 1 do
      begin
        Node := ChannelNode.ChildNodes[i];
        if (Node.NodeType = ELEMENT_NODE) and
           (TDOMElement(Node).TagName = 'item') then
          ParseItem(Node);
      end;
      
      Result := True;
    finally
      Doc.Free;
    end;
  except
    on E: Exception do
      WriteLn('RSS Load Error: ', E.Message);
  end;
end;

function TRSSReader.LoadFromFile(const FileName: string): boolean;
var
  Content: TStringList;
begin
  Content := TStringList.Create;
  try
    Content.LoadFromFile(FileName);
    Result := LoadFromString(Content.Text);
  finally
    Content.Free;
  end;
end;

function TRSSReader.LoadFromURL(const URL: string): boolean;
var
  HTTP: TFPHTTPClient;
  Stream: TStringStream;
begin
  Result := False;
  HTTP := TFPHTTPClient.Create(nil);
  Stream := TStringStream.Create('');
  try
    try
      HTTP.Get(URL, Stream);
      Result := LoadFromString(Stream.DataString);
    except
      on E: Exception do
        WriteLn('HTTP Error: ', E.Message);
    end;
  finally
    HTTP.Free;
    Stream.Free;
  end;
end;

procedure TRSSReader.ExportToText(const FileName: string);
var
  Lines: TStringList;
  i: integer;
  Item: TRSSItem;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('=== ' + FChannel.Title + ' ===');
    Lines.Add(FChannel.Description);
    Lines.Add('');
    
    for i := 0 to FItems.Count - 1 do
    begin
      Item := PRSSItem(FItems[i])^;
      Lines.Add('--- Item ' + IntToStr(i + 1) + ' ---');
      Lines.Add('Title: ' + Item.Title);
      Lines.Add('Date: ' + Item.PubDate);
      if Item.Author <> '' then
        Lines.Add('Author: ' + Item.Author);
      Lines.Add('Link: ' + Item.Link);
      Lines.Add('');
    end;
    
    Lines.SaveToFile(FileName);
    WriteLn('Text exported: ', FileName);
  finally
    Lines.Free;
  end;
end;

procedure TRSSReader.ExportToHTML(const FileName: string);
var
  Lines: TStringList;
  i: integer;
  Item: TRSSItem;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('<!DOCTYPE html>');
    Lines.Add('<html lang="th">');
    Lines.Add('<head><meta charset="UTF-8">');
    Lines.Add('<title>' + FChannel.Title + '</title>');
    Lines.Add('<style>');
    Lines.Add('body { font-family: sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }');
    Lines.Add('.item { border: 1px solid #ddd; margin: 10px 0; padding: 15px; border-radius: 5px; }');
    Lines.Add('h1 { color: #333; } h2 { color: #666; font-size: 1.1em; }');
    Lines.Add('a { color: #0066cc; } .date { color: #999; font-size: 0.9em; }');
    Lines.Add('</style></head><body>');
    Lines.Add('<h1>' + FChannel.Title + '</h1>');
    Lines.Add('<p>' + FChannel.Description + '</p>');
    Lines.Add('<hr>');
    
    for i := 0 to FItems.Count - 1 do
    begin
      Item := PRSSItem(FItems[i])^;
      Lines.Add('<div class="item">');
      Lines.Add('<h2><a href="' + Item.Link + '">' + Item.Title + '</a></h2>');
      Lines.Add('<p class="date">' + Item.PubDate);
      if Item.Author <> '' then
        Lines.Add(' | by ' + Item.Author);
      Lines.Add('</p>');
      Lines.Add('<p>' + Item.Description + '</p>');
      Lines.Add('</div>');
    end;
    
    Lines.Add('</body></html>');
    Lines.SaveToFile(FileName);
    WriteLn('HTML exported: ', FileName);
  finally
    Lines.Free;
  end;
end;

procedure TRSSReader.ExportToXMLSummary(const FileName: string);
var
  Doc: TXMLDocument;
  Root, Summary, ItemEl: TDOMElement;
  i: integer;
  Item: TRSSItem;
  
  procedure AddEl(Parent: TDOMElement; const Tag, Value: string);
  begin
    TDOMElement(Parent.AppendChild(Doc.CreateElement(Tag))).
      AppendChild(Doc.CreateTextNode(Value));
  end;
  
begin
  Doc := TXMLDocument.Create;
  try
    Root := TDOMElement(Doc.AppendChild(Doc.CreateElement('rss_summary')));
    Root.SetAttribute('generated', DateTimeToStr(Now));
    
    Summary := TDOMElement(Root.AppendChild(Doc.CreateElement('channel_info')));
    AddEl(Summary, 'title', FChannel.Title);
    AddEl(Summary, 'items_count', IntToStr(FItems.Count));
    
    for i := 0 to FItems.Count - 1 do
    begin
      Item := PRSSItem(FItems[i])^;
      ItemEl := TDOMElement(Root.AppendChild(Doc.CreateElement('item')));
      ItemEl.SetAttribute('index', IntToStr(i + 1));
      AddEl(ItemEl, 'title', Item.Title);
      AddEl(ItemEl, 'link', Item.Link);
      AddEl(ItemEl, 'date', Item.PubDate);
    end;
    
    WriteXMLFile(Doc, FileName);
    WriteLn('XML Summary exported: ', FileName);
  finally
    Doc.Free;
  end;
end;

procedure TRSSReader.Display;
var
  i: integer;
begin
  WriteLn('Feed: ', FChannel.Title);
  WriteLn('Description: ', FChannel.Description);
  WriteLn('Items: ', FItems.Count);
  WriteLn('');
  
  for i := 0 to FItems.Count - 1 do
  begin
    WriteLn('[', i + 1, '] ', PRSSItem(FItems[i])^.Title);
    WriteLn('    Date: ', PRSSItem(FItems[i])^.PubDate);
    WriteLn('    URL: ', PRSSItem(FItems[i])^.Link);
    WriteLn('');
  end;
end;

procedure TRSSReader.DisplayItem(Index: integer);
var
  Item: TRSSItem;
begin
  if (Index < 0) or (Index >= FItems.Count) then
  begin
    WriteLn('Invalid index');
    Exit;
  end;
  
  Item := PRSSItem(FItems[Index])^;
  WriteLn('Title: ', Item.Title);
  WriteLn('Date: ', Item.PubDate);
  WriteLn('Author: ', Item.Author);
  WriteLn('Category: ', Item.Category);
  WriteLn('Link: ', Item.Link);
  WriteLn('Description: ');
  WriteLn(Item.Description);
end;

// Demo RSS content
const
  SAMPLE_RSS =
    '<?xml version="1.0" encoding="UTF-8"?>' +
    '<rss version="2.0">' +
    '<channel>' +
    '<title>ข่าวเทคโนโลยี</title>' +
    '<description>ข่าวสารด้านไอทีและเทคโนโลยีใหม่</description>' +
    '<link>https://technews.example.com</link>' +
    '<language>th</language>' +
    '<lastBuildDate>Mon, 15 Jan 2024 10:00:00 +0700</lastBuildDate>' +
    '<item>' +
    '<title>Apple เปิดตัว iPhone 16 Series</title>' +
    '<description>Apple เปิดตัว iPhone รุ่นใหม่พร้อม AI features</description>' +
    '<link>https://technews.example.com/iphone16</link>' +
    '<pubDate>Mon, 15 Jan 2024 08:00:00 +0700</pubDate>' +
    '<author>editor@technews.example.com</author>' +
    '<category>Smartphones</category>' +
    '</item>' +
    '<item>' +
    '<title>Google ปล่อย Gemini Pro API</title>' +
    '<description>Google เปิดให้ developer เข้าถึง Gemini Pro API แล้ว</description>' +
    '<link>https://technews.example.com/gemini-api</link>' +
    '<pubDate>Sun, 14 Jan 2024 16:00:00 +0700</pubDate>' +
    '<author>tech@technews.example.com</author>' +
    '<category>AI</category>' +
    '</item>' +
    '<item>' +
    '<title>Lazarus 3.0 Released</title>' +
    '<description>Lazarus IDE version 3.0 ออกแล้วพร้อม features ใหม่มากมาย</description>' +
    '<link>https://technews.example.com/lazarus-30</link>' +
    '<pubDate>Sat, 13 Jan 2024 12:00:00 +0700</pubDate>' +
    '<author>pascal@technews.example.com</author>' +
    '<category>Development</category>' +
    '</item>' +
    '</channel>' +
    '</rss>';

var
  Reader: TRSSReader;
begin
  Reader := TRSSReader.Create;
  try
    WriteLn('Loading RSS feed...');
    if Reader.LoadFromString(SAMPLE_RSS) then
    begin
      WriteLn('RSS loaded successfully!');
      WriteLn('');
      Reader.Display;
      
      WriteLn('=== Detailed view of item 1 ===');
      Reader.DisplayItem(0);
      WriteLn('');
      
      // Export
      Reader.ExportToText('rss_export.txt');
      Reader.ExportToHTML('rss_export.html');
      Reader.ExportToXMLSummary('rss_summary.xml');
      
      // Cleanup
      DeleteFile('rss_export.txt');
      DeleteFile('rss_export.html');
      DeleteFile('rss_summary.xml');
    end
    else
      WriteLn('Failed to load RSS');
  finally
    Reader.Free;
  end;
end.
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1: XML Contact Book
สร้างสมุดโทรศัพท์ XML ที่รองรับ:
- เพิ่ม/ลบ/แก้ไขผู้ติดต่อ
- ค้นหาด้วยชื่อหรือเบอร์
- Export เป็น VCard format

### ข้อ 2: XML Inventory System
ระบบ inventory ที่:
- เก็บข้อมูลสินค้าใน XML
- ติดตาม stock level
- Alert เมื่อ stock ต่ำ

### ข้อ 3: XML Log Analyzer
Parser สำหรับ XML log files:
- อ่าน log ที่มีโครงสร้าง XML
- Filter ด้วย level (ERROR, WARN, INFO)
- สรุปสถิติ

### ข้อ 4: OPML Reader
Parser สำหรับ OPML (Outline Processor Markup Language):
- อ่าน feed list จาก OPML
- แสดง hierarchy
- Export เป็น RSS subscriptions list

### ข้อ 5: XML Merge Tool
เครื่องมือ merge XML files:
- รวม 2 XML files เข้าด้วยกัน
- จัดการ conflicts (duplicate keys)
- บันทึก merged result

### ข้อ 6: SVG Generator
สร้าง SVG graphics ด้วย XML DOM:
- สร้าง shapes (rect, circle, line)
- เพิ่ม text labels
- สร้าง bar chart จากข้อมูล

### ข้อ 7: XML Database
สร้าง simple XML database:
- CRUD operations
- Query ด้วย XPath
- Index สำหรับ fast search

### ข้อ 8: DocBook to HTML
Converter จาก DocBook XML เป็น HTML:
- แปลง chapter structure
- จัดการ tables
- สร้าง table of contents

### ข้อ 9: XML Diff Tool
เครื่องมือ compare XML:
- แสดง elements ที่เพิ่ม/ลบ/เปลี่ยน
- Output เป็น XML diff format
- รองรับ namespaces

### ข้อ 10: RSS Aggregator
RSS aggregator ที่:
- อ่าน feeds จาก หลาย URLs
- รวมรายการข่าวทั้งหมด
- เรียงลำดับตามวันที่
- Export เป็น single RSS feed
