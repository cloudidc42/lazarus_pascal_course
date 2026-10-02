# Part 41 - JSON Handling ใน Lazarus/Pascal

## บทนำ

JSON (JavaScript Object Notation) เป็นรูปแบบการแลกเปลี่ยนข้อมูลที่ได้รับความนิยมสูงมากในปัจจุบัน ใช้ในการสื่อสารระหว่าง API, การจัดเก็บ configuration และการแลกเปลี่ยนข้อมูลระหว่างระบบต่างๆ Lazarus/FPC มี unit `fpjson` ที่ให้ความสามารถในการจัดการ JSON ได้อย่างครบถ้วน

---

## 41.1 fpjson Unit

Unit `fpjson` เป็น unit หลักสำหรับการจัดการ JSON ใน Free Pascal ให้ class และ function สำหรับการสร้าง อ่าน และแก้ไข JSON

### การ uses unit ที่จำเป็น

```pascal
uses
  fpjson,        // หลัก JSON support
  jsonparser,    // สำหรับ parse JSON string
  jsonscanner;   // สำหรับ scan JSON tokens
```

### โครงสร้าง JSON Types

| Pascal Class | JSON Type | ตัวอย่าง |
|-------------|-----------|---------|
| `TJSONObject` | Object | `{"key": "value"}` |
| `TJSONArray` | Array | `[1, 2, 3]` |
| `TJSONString` | String | `"hello"` |
| `TJSONIntegerNumber` | Integer | `42` |
| `TJSONFloatNumber` | Float | `3.14` |
| `TJSONBoolean` | Boolean | `true`/`false` |
| `TJSONNull` | Null | `null` |

---

## 41.2 การสร้าง JSON Objects

### ตัวอย่างพื้นฐาน: สร้าง JSON Object

```pascal
program CreateJSONObject;

{$mode objfpc}{$H+}

uses
  fpjson;

var
  JSONObj: TJSONObject;
  JSONStr: string;
begin
  // สร้าง JSON object ใหม่
  JSONObj := TJSONObject.Create;
  try
    // เพิ่ม properties ต่างๆ
    JSONObj.Add('name', 'สมชาย ใจดี');
    JSONObj.Add('age', 30);
    JSONObj.Add('email', 'somchai@example.com');
    JSONObj.Add('isActive', True);
    JSONObj.Add('salary', 35000.50);
    
    // แปลงเป็น string
    JSONStr := JSONObj.AsJSON;
    WriteLn('JSON Output:');
    WriteLn(JSONStr);
    
    // FormatJSON สำหรับการแสดงผลที่อ่านง่าย
    WriteLn('');
    WriteLn('Formatted JSON:');
    WriteLn(JSONObj.FormatJSON);
    
  finally
    JSONObj.Free;
  end;
end.
```

**Output:**
```json
{"name":"สมชาย ใจดี","age":30,"email":"somchai@example.com","isActive":true,"salary":35000.5}
```

### การสร้าง JSON Object แบบ Nested

```pascal
program NestedJSONObject;

{$mode objfpc}{$H+}

uses
  fpjson;

var
  Person: TJSONObject;
  Address: TJSONObject;
  ContactInfo: TJSONObject;
begin
  Person := TJSONObject.Create;
  try
    Person.Add('name', 'สมหญิง รักดี');
    Person.Add('age', 25);
    
    // สร้าง nested object สำหรับที่อยู่
    Address := TJSONObject.Create;
    Address.Add('street', '123 ถนนสุขุมวิท');
    Address.Add('district', 'คลองเตย');
    Address.Add('province', 'กรุงเทพมหานคร');
    Address.Add('zipCode', '10110');
    
    // เพิ่ม address เข้าไปใน person
    // หมายเหตุ: เมื่อ Add object แล้ว ownership จะถูกโอนไป
    Person.Add('address', Address);
    
    // สร้าง contact info
    ContactInfo := TJSONObject.Create;
    ContactInfo.Add('phone', '081-234-5678');
    ContactInfo.Add('email', 'somying@example.com');
    Person.Add('contact', ContactInfo);
    
    WriteLn(Person.FormatJSON);
    
  finally
    Person.Free; // จะ free ลูกๆ อัตโนมัติ
  end;
end.
```

---

## 41.3 การ Parse JSON Strings

### การอ่าน JSON จาก String

```pascal
program ParseJSONString;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, SysUtils;

var
  JSONStr: string;
  JSONData: TJSONData;
  JSONObj: TJSONObject;
  Name: string;
  Age: integer;
  Email: string;
begin
  // JSON string ที่ต้องการ parse
  JSONStr := '{"name":"วิชัย สุขสันต์","age":28,"email":"wichai@test.com","active":true}';
  
  // Parse JSON string
  JSONData := GetJSON(JSONStr);
  try
    if JSONData.JSONType = jtObject then
    begin
      JSONObj := TJSONObject(JSONData);
      
      // อ่านค่าต่างๆ
      Name := JSONObj.Get('name', '');
      Age := JSONObj.Get('age', 0);
      Email := JSONObj.Get('email', '');
      
      WriteLn('Name: ', Name);
      WriteLn('Age: ', Age);
      WriteLn('Email: ', Email);
      WriteLn('Active: ', JSONObj.Get('active', False));
    end;
  finally
    JSONData.Free;
  end;
end.
```

### การ Parse JSON แบบปลอดภัย (Error Handling)

```pascal
program SafeJSONParse;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, SysUtils;

function TryParseJSON(const JSONStr: string; out Data: TJSONData): Boolean;
begin
  Result := False;
  Data := nil;
  try
    Data := GetJSON(JSONStr);
    Result := True;
  except
    on E: Exception do
    begin
      WriteLn('JSON Parse Error: ', E.Message);
      Data := nil;
    end;
  end;
end;

procedure ProcessJSON(const JSONStr: string);
var
  JSONData: TJSONData;
  JSONObj: TJSONObject;
begin
  if TryParseJSON(JSONStr, JSONData) then
  begin
    try
      if JSONData.JSONType = jtObject then
      begin
        JSONObj := TJSONObject(JSONData);
        WriteLn('Parsed successfully!');
        WriteLn('Name: ', JSONObj.Get('name', 'N/A'));
        WriteLn('Value: ', JSONObj.Get('value', 0));
      end
      else
        WriteLn('Expected object, got: ', JSONData.ClassName);
    finally
      JSONData.Free;
    end;
  end
  else
    WriteLn('Failed to parse JSON');
end;

begin
  // Test ด้วย JSON ที่ถูกต้อง
  WriteLn('=== Valid JSON ===');
  ProcessJSON('{"name":"test","value":42}');
  
  // Test ด้วย JSON ที่ไม่ถูกต้อง
  WriteLn('');
  WriteLn('=== Invalid JSON ===');
  ProcessJSON('{invalid json}');
end.
```

---

## 41.4 JSON Arrays

### การสร้าง JSON Array

```pascal
program JSONArrayExample;

{$mode objfpc}{$H+}

uses
  fpjson;

var
  JSONArr: TJSONArray;
  JSONObj: TJSONObject;
  Item: TJSONObject;
  i: integer;
begin
  // สร้าง array ของ strings
  JSONArr := TJSONArray.Create;
  try
    JSONArr.Add('แอปเปิ้ล');
    JSONArr.Add('กล้วย');
    JSONArr.Add('ส้ม');
    JSONArr.Add('มะม่วง');
    
    WriteLn('Array of strings:');
    WriteLn(JSONArr.AsJSON);
    WriteLn('Count: ', JSONArr.Count);
    
    // อ่านค่าจาก array
    for i := 0 to JSONArr.Count - 1 do
      WriteLn('Item[', i, ']: ', JSONArr.Strings[i]);
  finally
    JSONArr.Free;
  end;
  
  WriteLn('');
  
  // สร้าง array ของ objects
  JSONArr := TJSONArray.Create;
  try
    // เพิ่มสินค้า
    Item := TJSONObject.Create;
    Item.Add('id', 1);
    Item.Add('name', 'สินค้า A');
    Item.Add('price', 100.00);
    JSONArr.Add(Item);
    
    Item := TJSONObject.Create;
    Item.Add('id', 2);
    Item.Add('name', 'สินค้า B');
    Item.Add('price', 250.50);
    JSONArr.Add(Item);
    
    Item := TJSONObject.Create;
    Item.Add('id', 3);
    Item.Add('name', 'สินค้า C');
    Item.Add('price', 75.25);
    JSONArr.Add(Item);
    
    WriteLn('Array of objects:');
    WriteLn(JSONArr.FormatJSON);
    
  finally
    JSONArr.Free;
  end;
end.
```

### การ Iterate ผ่าน JSON Array

```pascal
program IterateJSONArray;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser;

var
  JSONStr: string;
  JSONData: TJSONData;
  JSONArr: TJSONArray;
  JSONObj: TJSONObject;
  i: integer;
  TotalPrice: double;
begin
  JSONStr := '[' +
    '{"id":1,"name":"กาแฟ","price":45.00,"qty":2},' +
    '{"id":2,"name":"ชาเขียว","price":35.00,"qty":3},' +
    '{"id":3,"name":"น้ำส้ม","price":30.00,"qty":1}' +
    ']';
    
  JSONData := GetJSON(JSONStr);
  try
    if JSONData.JSONType = jtArray then
    begin
      JSONArr := TJSONArray(JSONData);
      TotalPrice := 0;
      
      WriteLn('รายการสินค้า:');
      WriteLn('----------------------------');
      
      for i := 0 to JSONArr.Count - 1 do
      begin
        if JSONArr.Items[i].JSONType = jtObject then
        begin
          JSONObj := TJSONObject(JSONArr.Items[i]);
          
          WriteLn(Format('%-15s x%d = %.2f บาท', [
            JSONObj.Get('name', ''),
            JSONObj.Get('qty', 0),
            JSONObj.Get('price', 0.0) * JSONObj.Get('qty', 0)
          ]));
          
          TotalPrice := TotalPrice + 
            JSONObj.Get('price', 0.0) * JSONObj.Get('qty', 0);
        end;
      end;
      
      WriteLn('----------------------------');
      WriteLn('รวมทั้งหมด: ', TotalPrice:0:2, ' บาท');
    end;
  finally
    JSONData.Free;
  end;
end.
```

---

## 41.5 Nested JSON

### การสร้างและอ่าน JSON ซ้อนกัน

```pascal
program NestedJSONComplete;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser;

// สร้าง JSON ของนักเรียน
function CreateStudentJSON: TJSONObject;
var
  Student: TJSONObject;
  Grades: TJSONObject;
  Subjects: TJSONArray;
  Subject: TJSONObject;
  Hobbies: TJSONArray;
begin
  Student := TJSONObject.Create;
  
  // ข้อมูลพื้นฐาน
  Student.Add('studentId', 'ST001');
  Student.Add('firstName', 'กิตติ');
  Student.Add('lastName', 'มีสุข');
  Student.Add('age', 20);
  
  // เกรด
  Grades := TJSONObject.Create;
  Grades.Add('GPA', 3.75);
  Grades.Add('grade', 'A');
  Grades.Add('semester', 2);
  Grades.Add('year', 2024);
  Student.Add('grades', Grades);
  
  // วิชาที่ลงทะเบียน
  Subjects := TJSONArray.Create;
  
  Subject := TJSONObject.Create;
  Subject.Add('code', 'CS101');
  Subject.Add('name', 'Introduction to Programming');
  Subject.Add('credits', 3);
  Subject.Add('score', 90);
  Subjects.Add(Subject);
  
  Subject := TJSONObject.Create;
  Subject.Add('code', 'MATH201');
  Subject.Add('name', 'Calculus I');
  Subject.Add('credits', 4);
  Subject.Add('score', 85);
  Subjects.Add(Subject);
  
  Subject := TJSONObject.Create;
  Subject.Add('code', 'ENG101');
  Subject.Add('name', 'English for Communication');
  Subject.Add('credits', 3);
  Subject.Add('score', 88);
  Subjects.Add(Subject);
  
  Student.Add('subjects', Subjects);
  
  // งานอดิเรก
  Hobbies := TJSONArray.Create;
  Hobbies.Add('อ่านหนังสือ');
  Hobbies.Add('เล่นกีตาร์');
  Hobbies.Add('วาดรูป');
  Student.Add('hobbies', Hobbies);
  
  Result := Student;
end;

// อ่านข้อมูลจาก JSON
procedure ReadStudentData(const JSONStr: string);
var
  JSONData: TJSONData;
  Student: TJSONObject;
  Grades: TJSONObject;
  Subjects: TJSONArray;
  Subject: TJSONObject;
  Hobbies: TJSONArray;
  i: integer;
begin
  JSONData := GetJSON(JSONStr);
  try
    Student := TJSONObject(JSONData);
    
    WriteLn('=== ข้อมูลนักเรียน ===');
    WriteLn('รหัส: ', Student.Get('studentId', ''));
    WriteLn('ชื่อ: ', Student.Get('firstName', ''), ' ', Student.Get('lastName', ''));
    WriteLn('อายุ: ', Student.Get('age', 0), ' ปี');
    
    // อ่านเกรด (nested object)
    if Student.IndexOfName('grades') >= 0 then
    begin
      Grades := TJSONObject(Student.Find('grades'));
      WriteLn('');
      WriteLn('=== ผลการเรียน ===');
      WriteLn('GPA: ', Grades.Get('GPA', 0.0):0:2);
      WriteLn('เกรด: ', Grades.Get('grade', ''));
    end;
    
    // อ่านวิชา (nested array of objects)
    if Student.IndexOfName('subjects') >= 0 then
    begin
      Subjects := TJSONArray(Student.Find('subjects'));
      WriteLn('');
      WriteLn('=== รายวิชาที่ลงทะเบียน ===');
      
      for i := 0 to Subjects.Count - 1 do
      begin
        Subject := TJSONObject(Subjects.Items[i]);
        WriteLn(Format('  %s - %s (%d หน่วยกิต) คะแนน: %d', [
          Subject.Get('code', ''),
          Subject.Get('name', ''),
          Subject.Get('credits', 0),
          Subject.Get('score', 0)
        ]));
      end;
    end;
    
    // อ่านงานอดิเรก (nested array of strings)
    if Student.IndexOfName('hobbies') >= 0 then
    begin
      Hobbies := TJSONArray(Student.Find('hobbies'));
      WriteLn('');
      WriteLn('งานอดิเรก: ');
      for i := 0 to Hobbies.Count - 1 do
        WriteLn('  - ', Hobbies.Strings[i]);
    end;
    
  finally
    JSONData.Free;
  end;
end;

var
  StudentJSON: TJSONObject;
  JSONStr: string;
begin
  // สร้าง JSON
  StudentJSON := CreateStudentJSON;
  try
    JSONStr := StudentJSON.AsJSON;
    WriteLn('JSON สร้างสำเร็จ');
  finally
    StudentJSON.Free;
  end;
  
  WriteLn('');
  // อ่าน JSON
  ReadStudentData(JSONStr);
end.
```

---

## 41.6 JSON to Object Mapping

### การสร้าง Class สำหรับ JSON Mapping

```pascal
program JSONToObject;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, SysUtils;

type
  // Address record
  TAddress = record
    Street: string;
    City: string;
    ZipCode: string;
    Country: string;
  end;
  
  // Customer class ที่ map กับ JSON
  TCustomer = class
  private
    FID: integer;
    FName: string;
    FEmail: string;
    FPhone: string;
    FAddress: TAddress;
    FBalance: double;
    FActive: boolean;
    FTags: TStringArray;
  public
    constructor Create;
    
    // Load from JSON
    procedure FromJSON(const JSONData: TJSONObject);
    // Save to JSON
    function ToJSON: TJSONObject;
    // Display
    procedure Display;
    
    property ID: integer read FID write FID;
    property Name: string read FName write FName;
    property Email: string read FEmail write FEmail;
    property Phone: string read FPhone write FPhone;
    property Balance: double read FBalance write FBalance;
    property Active: boolean read FActive write FActive;
  end;

constructor TCustomer.Create;
begin
  inherited Create;
  FActive := True;
  FBalance := 0;
  SetLength(FTags, 0);
end;

procedure TCustomer.FromJSON(const JSONData: TJSONObject);
var
  AddrObj: TJSONObject;
  TagsArr: TJSONArray;
  i: integer;
begin
  FID := JSONData.Get('id', 0);
  FName := JSONData.Get('name', '');
  FEmail := JSONData.Get('email', '');
  FPhone := JSONData.Get('phone', '');
  FBalance := JSONData.Get('balance', 0.0);
  FActive := JSONData.Get('active', True);
  
  // อ่าน nested object
  if JSONData.IndexOfName('address') >= 0 then
  begin
    AddrObj := TJSONObject(JSONData.Find('address'));
    FAddress.Street := AddrObj.Get('street', '');
    FAddress.City := AddrObj.Get('city', '');
    FAddress.ZipCode := AddrObj.Get('zipCode', '');
    FAddress.Country := AddrObj.Get('country', '');
  end;
  
  // อ่าน array
  if JSONData.IndexOfName('tags') >= 0 then
  begin
    TagsArr := TJSONArray(JSONData.Find('tags'));
    SetLength(FTags, TagsArr.Count);
    for i := 0 to TagsArr.Count - 1 do
      FTags[i] := TagsArr.Strings[i];
  end;
end;

function TCustomer.ToJSON: TJSONObject;
var
  AddrObj: TJSONObject;
  TagsArr: TJSONArray;
  i: integer;
begin
  Result := TJSONObject.Create;
  
  Result.Add('id', FID);
  Result.Add('name', FName);
  Result.Add('email', FEmail);
  Result.Add('phone', FPhone);
  Result.Add('balance', FBalance);
  Result.Add('active', FActive);
  
  // สร้าง nested address
  AddrObj := TJSONObject.Create;
  AddrObj.Add('street', FAddress.Street);
  AddrObj.Add('city', FAddress.City);
  AddrObj.Add('zipCode', FAddress.ZipCode);
  AddrObj.Add('country', FAddress.Country);
  Result.Add('address', AddrObj);
  
  // สร้าง tags array
  TagsArr := TJSONArray.Create;
  for i := 0 to High(FTags) do
    TagsArr.Add(FTags[i]);
  Result.Add('tags', TagsArr);
end;

procedure TCustomer.Display;
var
  Tag: string;
begin
  WriteLn('=== Customer Info ===');
  WriteLn('ID: ', FID);
  WriteLn('Name: ', FName);
  WriteLn('Email: ', FEmail);
  WriteLn('Phone: ', FPhone);
  WriteLn('Balance: ', FBalance:0:2, ' บาท');
  WriteLn('Active: ', BoolToStr(FActive, 'Yes', 'No'));
  WriteLn('Address: ', FAddress.Street, ', ', FAddress.City);
  WriteLn('ZipCode: ', FAddress.ZipCode, ', ', FAddress.Country);
  
  if Length(FTags) > 0 then
  begin
    Write('Tags: ');
    for Tag in FTags do
      Write(Tag, ' ');
    WriteLn;
  end;
end;

var
  JSONStr: string;
  JSONData: TJSONData;
  Customer: TCustomer;
  JSONOut: TJSONObject;
begin
  // Test JSON
  JSONStr := '{' +
    '"id": 1001,' +
    '"name": "นิภา สวยงาม",' +
    '"email": "nipha@example.com",' +
    '"phone": "089-765-4321",' +
    '"balance": 15000.75,' +
    '"active": true,' +
    '"address": {' +
    '  "street": "456 ถนนพหลโยธิน",' +
    '  "city": "กรุงเทพมหานคร",' +
    '  "zipCode": "10400",' +
    '  "country": "ไทย"' +
    '},' +
    '"tags": ["VIP", "Premium", "LoyalCustomer"]' +
    '}';
  
  // Parse และ map ไปยัง object
  JSONData := GetJSON(JSONStr);
  try
    Customer := TCustomer.Create;
    try
      Customer.FromJSON(TJSONObject(JSONData));
      Customer.Display;
      
      WriteLn('');
      WriteLn('=== Convert back to JSON ===');
      JSONOut := Customer.ToJSON;
      try
        WriteLn(JSONOut.FormatJSON);
      finally
        JSONOut.Free;
      end;
    finally
      Customer.Free;
    end;
  finally
    JSONData.Free;
  end;
end.
```

---

## 41.7 Object to JSON

### Generic JSON Serializer

```pascal
program ObjectToJSONSerializer;

{$mode objfpc}{$H+}

uses
  fpjson, SysUtils, Classes;

type
  TProductCategory = (pcElectronics, pcClothing, pcFood, pcBooks);
  
  TProduct = class
  private
    FProductID: string;
    FName: string;
    FDescription: string;
    FPrice: double;
    FStock: integer;
    FCategory: TProductCategory;
    FTags: TStringList;
    FCreatedAt: TDateTime;
    FIsAvailable: boolean;
  public
    constructor Create;
    destructor Destroy; override;
    
    function Serialize: TJSONObject;
    class function Deserialize(const JSON: TJSONObject): TProduct;
    
    property ProductID: string read FProductID write FProductID;
    property Name: string read FName write FName;
    property Description: string read FDescription write FDescription;
    property Price: double read FPrice write FPrice;
    property Stock: integer read FStock write FStock;
    property Category: TProductCategory read FCategory write FCategory;
    property Tags: TStringList read FTags;
    property CreatedAt: TDateTime read FCreatedAt write FCreatedAt;
    property IsAvailable: boolean read FIsAvailable write FIsAvailable;
  end;

const
  CategoryNames: array[TProductCategory] of string = (
    'Electronics', 'Clothing', 'Food', 'Books'
  );

constructor TProduct.Create;
begin
  inherited Create;
  FTags := TStringList.Create;
  FCreatedAt := Now;
  FIsAvailable := True;
end;

destructor TProduct.Destroy;
begin
  FTags.Free;
  inherited Destroy;
end;

function TProduct.Serialize: TJSONObject;
var
  TagsArray: TJSONArray;
  i: integer;
begin
  Result := TJSONObject.Create;
  Result.Add('productId', FProductID);
  Result.Add('name', FName);
  Result.Add('description', FDescription);
  Result.Add('price', FPrice);
  Result.Add('stock', FStock);
  Result.Add('category', CategoryNames[FCategory]);
  Result.Add('createdAt', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss', FCreatedAt));
  Result.Add('isAvailable', FIsAvailable);
  
  TagsArray := TJSONArray.Create;
  for i := 0 to FTags.Count - 1 do
    TagsArray.Add(FTags[i]);
  Result.Add('tags', TagsArray);
end;

class function TProduct.Deserialize(const JSON: TJSONObject): TProduct;
var
  Cat: TProductCategory;
  CatStr: string;
  TagsArr: TJSONArray;
  i: integer;
begin
  Result := TProduct.Create;
  Result.ProductID := JSON.Get('productId', '');
  Result.Name := JSON.Get('name', '');
  Result.Description := JSON.Get('description', '');
  Result.Price := JSON.Get('price', 0.0);
  Result.Stock := JSON.Get('stock', 0);
  Result.IsAvailable := JSON.Get('isAvailable', True);
  
  // Map category string กลับเป็น enum
  CatStr := JSON.Get('category', 'Electronics');
  for Cat := Low(TProductCategory) to High(TProductCategory) do
    if CategoryNames[Cat] = CatStr then
    begin
      Result.Category := Cat;
      Break;
    end;
  
  // อ่าน tags
  if JSON.IndexOfName('tags') >= 0 then
  begin
    TagsArr := TJSONArray(JSON.Find('tags'));
    for i := 0 to TagsArr.Count - 1 do
      Result.Tags.Add(TagsArr.Strings[i]);
  end;
end;

var
  Product: TProduct;
  JSON: TJSONObject;
  JSONStr: string;
  Product2: TProduct;
begin
  // สร้าง product
  Product := TProduct.Create;
  try
    Product.ProductID := 'PROD-001';
    Product.Name := 'MacBook Pro 14"';
    Product.Description := 'Apple MacBook Pro 14 inch M3 chip';
    Product.Price := 89900.00;
    Product.Stock := 15;
    Product.Category := pcElectronics;
    Product.Tags.Add('Apple');
    Product.Tags.Add('Laptop');
    Product.Tags.Add('M3');
    Product.IsAvailable := True;
    
    // Serialize เป็น JSON
    JSON := Product.Serialize;
    try
      JSONStr := JSON.FormatJSON;
      WriteLn('Serialized JSON:');
      WriteLn(JSONStr);
    finally
      JSON.Free;
    end;
  finally
    Product.Free;
  end;
  
  // Deserialize กลับมาเป็น object
  WriteLn('');
  WriteLn('Deserializing...');
  JSON := TJSONObject(GetJSON(JSONStr));
  try
    Product2 := TProduct.Deserialize(JSON);
    try
      WriteLn('Product ID: ', Product2.ProductID);
      WriteLn('Name: ', Product2.Name);
      WriteLn('Price: ', Product2.Price:0:2);
      WriteLn('Category: ', CategoryNames[Product2.Category]);
      WriteLn('Tags: ', Product2.Tags.CommaText);
    finally
      Product2.Free;
    end;
  finally
    JSON.Free;
  end;
end.
```

---

## 41.8 Error Handling

### การจัดการ Error ใน JSON

```pascal
program JSONErrorHandling;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, SysUtils;

type
  EJSONError = class(Exception);
  EJSONMissingField = class(EJSONError);
  EJSONTypeError = class(EJSONError);

// ฟังก์ชัน helper สำหรับการอ่านค่าอย่างปลอดภัย
function GetRequiredString(const JSON: TJSONObject; const Key: string): string;
begin
  if JSON.IndexOfName(Key) < 0 then
    raise EJSONMissingField.CreateFmt('Required field "%s" is missing', [Key]);
    
  if JSON.Find(Key).JSONType <> jtString then
    raise EJSONTypeError.CreateFmt('Field "%s" must be a string', [Key]);
    
  Result := JSON.Strings[Key];
end;

function GetRequiredInteger(const JSON: TJSONObject; const Key: string): integer;
begin
  if JSON.IndexOfName(Key) < 0 then
    raise EJSONMissingField.CreateFmt('Required field "%s" is missing', [Key]);
    
  if not (JSON.Find(Key).JSONType in [jtNumber]) then
    raise EJSONTypeError.CreateFmt('Field "%s" must be a number', [Key]);
    
  Result := JSON.Integers[Key];
end;

function GetOptionalString(const JSON: TJSONObject; const Key, Default: string): string;
begin
  if JSON.IndexOfName(Key) >= 0 then
    Result := JSON.Get(Key, Default)
  else
    Result := Default;
end;

procedure ProcessUserData(const JSONStr: string);
var
  JSONData: TJSONData;
  JSON: TJSONObject;
begin
  try
    JSONData := GetJSON(JSONStr);
    try
      if JSONData.JSONType <> jtObject then
        raise EJSONError.Create('Expected JSON object');
        
      JSON := TJSONObject(JSONData);
      
      WriteLn('ID: ', GetRequiredInteger(JSON, 'id'));
      WriteLn('Username: ', GetRequiredString(JSON, 'username'));
      WriteLn('Email: ', GetRequiredString(JSON, 'email'));
      WriteLn('Role: ', GetOptionalString(JSON, 'role', 'user'));
      WriteLn('Bio: ', GetOptionalString(JSON, 'bio', '(ไม่มีข้อมูล)'));
      
    finally
      JSONData.Free;
    end;
  except
    on E: EJSONMissingField do
      WriteLn('ข้อมูลไม่ครบ: ', E.Message);
    on E: EJSONTypeError do
      WriteLn('ประเภทข้อมูลผิด: ', E.Message);
    on E: EJSONError do
      WriteLn('JSON Error: ', E.Message);
    on E: Exception do
      WriteLn('Unexpected error: ', E.Message);
  end;
end;

begin
  WriteLn('=== Test 1: Valid JSON ===');
  ProcessUserData('{"id":1,"username":"admin","email":"admin@test.com","role":"admin"}');
  
  WriteLn('');
  WriteLn('=== Test 2: Missing required field ===');
  ProcessUserData('{"id":2,"username":"user2"}');  // ขาด email
  
  WriteLn('');
  WriteLn('=== Test 3: Wrong type ===');
  ProcessUserData('{"id":"not-a-number","username":"user3","email":"u3@test.com"}');
  
  WriteLn('');
  WriteLn('=== Test 4: Invalid JSON ===');
  ProcessUserData('{broken json here}');
end.
```

---

## 41.9 REST API Responses

### การจัดการ REST API Response

```pascal
program RESTAPIResponse;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, fphttpclient, SysUtils, Classes;

type
  TAPIResponse = record
    Success: boolean;
    StatusCode: integer;
    Message: string;
    Data: TJSONData;
  end;
  
  TAPIClient = class
  private
    FBaseURL: string;
    function ParseResponse(const ResponseStr: string): TAPIResponse;
  public
    constructor Create(const BaseURL: string);
    destructor Destroy; override;
    
    // Simulate API call (ในกรณีจริงจะใช้ HTTP client)
    function SimulateGetUser(UserID: integer): TAPIResponse;
    function SimulateGetProducts: TAPIResponse;
  end;

constructor TAPIClient.Create(const BaseURL: string);
begin
  inherited Create;
  FBaseURL := BaseURL;
end;

destructor TAPIClient.Destroy;
begin
  inherited Destroy;
end;

function TAPIClient.ParseResponse(const ResponseStr: string): TAPIResponse;
var
  JSONData: TJSONData;
  JSONObj: TJSONObject;
begin
  Result.Success := False;
  Result.StatusCode := 0;
  Result.Message := '';
  Result.Data := nil;
  
  try
    JSONData := GetJSON(ResponseStr);
    if JSONData.JSONType = jtObject then
    begin
      JSONObj := TJSONObject(JSONData);
      Result.Success := JSONObj.Get('success', False);
      Result.StatusCode := JSONObj.Get('statusCode', 0);
      Result.Message := JSONObj.Get('message', '');
      
      // Extract data
      if JSONObj.IndexOfName('data') >= 0 then
        Result.Data := JSONObj.Extract('data');
        
      JSONData.Free; // Free ที่เหลือ (ไม่รวม data ที่ extract แล้ว)
    end
    else
      JSONData.Free;
  except
    on E: Exception do
    begin
      Result.Success := False;
      Result.Message := 'Parse error: ' + E.Message;
    end;
  end;
end;

function TAPIClient.SimulateGetUser(UserID: integer): TAPIResponse;
var
  FakeResponse: string;
begin
  // Simulate REST API response
  case UserID of
    1: FakeResponse := '{"success":true,"statusCode":200,"message":"OK",' +
       '"data":{"id":1,"name":"สมชาย ใจดี","email":"somchai@test.com",' +
       '"role":"admin","lastLogin":"2024-01-15T10:30:00"}}';
    2: FakeResponse := '{"success":true,"statusCode":200,"message":"OK",' +
       '"data":{"id":2,"name":"สมหญิง รักดี","email":"somying@test.com",' +
       '"role":"user","lastLogin":"2024-01-14T15:20:00"}}';
    else
      FakeResponse := '{"success":false,"statusCode":404,"message":"User not found","data":null}';
  end;
  
  Result := ParseResponse(FakeResponse);
end;

function TAPIClient.SimulateGetProducts: TAPIResponse;
var
  FakeResponse: string;
begin
  FakeResponse := '{"success":true,"statusCode":200,"message":"OK",' +
    '"data":{"total":3,"page":1,"perPage":10,' +
    '"items":[' +
    '{"id":"P001","name":"iPhone 15","price":35000,"stock":50},' +
    '{"id":"P002","name":"Samsung S24","price":28000,"stock":30},' +
    '{"id":"P003","name":"Google Pixel 8","price":22000,"stock":20}' +
    ']}}';
    
  Result := ParseResponse(FakeResponse);
end;

procedure DisplayUserResponse(const Response: TAPIResponse);
var
  UserObj: TJSONObject;
begin
  WriteLn('Status Code: ', Response.StatusCode);
  WriteLn('Message: ', Response.Message);
  
  if Response.Success and Assigned(Response.Data) then
  begin
    UserObj := TJSONObject(Response.Data);
    WriteLn('User Data:');
    WriteLn('  ID: ', UserObj.Get('id', 0));
    WriteLn('  Name: ', UserObj.Get('name', ''));
    WriteLn('  Email: ', UserObj.Get('email', ''));
    WriteLn('  Role: ', UserObj.Get('role', ''));
    WriteLn('  Last Login: ', UserObj.Get('lastLogin', ''));
  end
  else
    WriteLn('Error: ', Response.Message);
    
  Response.Data.Free; // ต้อง free ด้วยตัวเอง
end;

procedure DisplayProductsResponse(const Response: TAPIResponse);
var
  DataObj: TJSONObject;
  Items: TJSONArray;
  Item: TJSONObject;
  i: integer;
begin
  WriteLn('Status Code: ', Response.StatusCode);
  
  if Response.Success and Assigned(Response.Data) then
  begin
    DataObj := TJSONObject(Response.Data);
    WriteLn('Total products: ', DataObj.Get('total', 0));
    WriteLn('');
    
    Items := TJSONArray(DataObj.Find('items'));
    if Assigned(Items) then
    begin
      WriteLn('Products List:');
      for i := 0 to Items.Count - 1 do
      begin
        Item := TJSONObject(Items.Items[i]);
        WriteLn(Format('  [%s] %s - %.0f บาท (stock: %d)', [
          Item.Get('id', ''),
          Item.Get('name', ''),
          Item.Get('price', 0.0),
          Item.Get('stock', 0)
        ]));
      end;
    end;
  end;
  
  Response.Data.Free;
end;

var
  Client: TAPIClient;
  Response: TAPIResponse;
begin
  Client := TAPIClient.Create('https://api.example.com');
  try
    // Test get user
    WriteLn('=== Get User ID=1 ===');
    Response := Client.SimulateGetUser(1);
    DisplayUserResponse(Response);
    
    WriteLn('');
    WriteLn('=== Get User ID=999 (Not Found) ===');
    Response := Client.SimulateGetUser(999);
    DisplayUserResponse(Response);
    
    WriteLn('');
    WriteLn('=== Get Products ===');
    Response := Client.SimulateGetProducts;
    DisplayProductsResponse(Response);
    
  finally
    Client.Free;
  end;
end.
```

---

## 41.10 Complete Example: Config File Using JSON

```pascal
program JSONConfigManager;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, SysUtils, Classes;

type
  TDatabaseConfig = record
    Host: string;
    Port: integer;
    DatabaseName: string;
    Username: string;
    Password: string;
    MaxConnections: integer;
  end;
  
  TServerConfig = record
    Host: string;
    Port: integer;
    EnableSSL: boolean;
    SSLCertPath: string;
    MaxRequestSize: integer;
    TimeoutSeconds: integer;
  end;
  
  TLoggingConfig = record
    Level: string;
    FilePath: string;
    MaxFileSize: integer;
    MaxFiles: integer;
    EnableConsole: boolean;
  end;
  
  TAppConfig = class
  private
    FConfigPath: string;
    FAppName: string;
    FVersion: string;
    FEnvironment: string;
    FDatabase: TDatabaseConfig;
    FServer: TServerConfig;
    FLogging: TLoggingConfig;
    FFeatureFlags: TStringList;
    
    procedure LoadDefaults;
    procedure ParseDatabase(const JSON: TJSONObject);
    procedure ParseServer(const JSON: TJSONObject);
    procedure ParseLogging(const JSON: TJSONObject);
    function BuildDatabaseJSON: TJSONObject;
    function BuildServerJSON: TJSONObject;
    function BuildLoggingJSON: TJSONObject;
  public
    constructor Create(const ConfigPath: string);
    destructor Destroy; override;
    
    function Load: boolean;
    function Save: boolean;
    procedure Display;
    
    property AppName: string read FAppName write FAppName;
    property Version: string read FVersion write FVersion;
    property Environment: string read FEnvironment write FEnvironment;
    property Database: TDatabaseConfig read FDatabase write FDatabase;
    property Server: TServerConfig read FServer write FServer;
    property Logging: TLoggingConfig read FLogging write FLogging;
    property FeatureFlags: TStringList read FFeatureFlags;
  end;

constructor TAppConfig.Create(const ConfigPath: string);
begin
  inherited Create;
  FConfigPath := ConfigPath;
  FFeatureFlags := TStringList.Create;
  LoadDefaults;
end;

destructor TAppConfig.Destroy;
begin
  FFeatureFlags.Free;
  inherited Destroy;
end;

procedure TAppConfig.LoadDefaults;
begin
  FAppName := 'MyApplication';
  FVersion := '1.0.0';
  FEnvironment := 'development';
  
  // Database defaults
  FDatabase.Host := 'localhost';
  FDatabase.Port := 5432;
  FDatabase.DatabaseName := 'myapp_db';
  FDatabase.Username := 'admin';
  FDatabase.Password := '';
  FDatabase.MaxConnections := 10;
  
  // Server defaults
  FServer.Host := '0.0.0.0';
  FServer.Port := 8080;
  FServer.EnableSSL := False;
  FServer.SSLCertPath := '';
  FServer.MaxRequestSize := 10485760; // 10MB
  FServer.TimeoutSeconds := 30;
  
  // Logging defaults
  FLogging.Level := 'info';
  FLogging.FilePath := 'logs/app.log';
  FLogging.MaxFileSize := 10485760; // 10MB
  FLogging.MaxFiles := 5;
  FLogging.EnableConsole := True;
end;

procedure TAppConfig.ParseDatabase(const JSON: TJSONObject);
begin
  FDatabase.Host := JSON.Get('host', FDatabase.Host);
  FDatabase.Port := JSON.Get('port', FDatabase.Port);
  FDatabase.DatabaseName := JSON.Get('database', FDatabase.DatabaseName);
  FDatabase.Username := JSON.Get('username', FDatabase.Username);
  FDatabase.Password := JSON.Get('password', FDatabase.Password);
  FDatabase.MaxConnections := JSON.Get('maxConnections', FDatabase.MaxConnections);
end;

procedure TAppConfig.ParseServer(const JSON: TJSONObject);
begin
  FServer.Host := JSON.Get('host', FServer.Host);
  FServer.Port := JSON.Get('port', FServer.Port);
  FServer.EnableSSL := JSON.Get('enableSSL', FServer.EnableSSL);
  FServer.SSLCertPath := JSON.Get('sslCertPath', FServer.SSLCertPath);
  FServer.MaxRequestSize := JSON.Get('maxRequestSize', FServer.MaxRequestSize);
  FServer.TimeoutSeconds := JSON.Get('timeoutSeconds', FServer.TimeoutSeconds);
end;

procedure TAppConfig.ParseLogging(const JSON: TJSONObject);
begin
  FLogging.Level := JSON.Get('level', FLogging.Level);
  FLogging.FilePath := JSON.Get('filePath', FLogging.FilePath);
  FLogging.MaxFileSize := JSON.Get('maxFileSize', FLogging.MaxFileSize);
  FLogging.MaxFiles := JSON.Get('maxFiles', FLogging.MaxFiles);
  FLogging.EnableConsole := JSON.Get('enableConsole', FLogging.EnableConsole);
end;

function TAppConfig.Load: boolean;
var
  ConfigStr: string;
  ConfigFile: TStringList;
  JSONData: TJSONData;
  JSONObj: TJSONObject;
  FlagsArr: TJSONArray;
  i: integer;
begin
  Result := False;
  
  if not FileExists(FConfigPath) then
  begin
    WriteLn('Config file not found: ', FConfigPath);
    WriteLn('Using defaults...');
    Result := True;
    Exit;
  end;
  
  try
    ConfigFile := TStringList.Create;
    try
      ConfigFile.LoadFromFile(FConfigPath);
      ConfigStr := ConfigFile.Text;
    finally
      ConfigFile.Free;
    end;
    
    JSONData := GetJSON(ConfigStr);
    try
      if JSONData.JSONType <> jtObject then
        raise Exception.Create('Config file must be a JSON object');
        
      JSONObj := TJSONObject(JSONData);
      
      FAppName := JSONObj.Get('appName', FAppName);
      FVersion := JSONObj.Get('version', FVersion);
      FEnvironment := JSONObj.Get('environment', FEnvironment);
      
      if JSONObj.IndexOfName('database') >= 0 then
        ParseDatabase(TJSONObject(JSONObj.Find('database')));
        
      if JSONObj.IndexOfName('server') >= 0 then
        ParseServer(TJSONObject(JSONObj.Find('server')));
        
      if JSONObj.IndexOfName('logging') >= 0 then
        ParseLogging(TJSONObject(JSONObj.Find('logging')));
      
      // Feature flags
      FFeatureFlags.Clear;
      if JSONObj.IndexOfName('featureFlags') >= 0 then
      begin
        FlagsArr := TJSONArray(JSONObj.Find('featureFlags'));
        for i := 0 to FlagsArr.Count - 1 do
          FFeatureFlags.Add(FlagsArr.Strings[i]);
      end;
      
      Result := True;
      WriteLn('Config loaded from: ', FConfigPath);
      
    finally
      JSONData.Free;
    end;
  except
    on E: Exception do
    begin
      WriteLn('Failed to load config: ', E.Message);
      Result := False;
    end;
  end;
end;

function TAppConfig.BuildDatabaseJSON: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('host', FDatabase.Host);
  Result.Add('port', FDatabase.Port);
  Result.Add('database', FDatabase.DatabaseName);
  Result.Add('username', FDatabase.Username);
  Result.Add('password', FDatabase.Password);
  Result.Add('maxConnections', FDatabase.MaxConnections);
end;

function TAppConfig.BuildServerJSON: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('host', FServer.Host);
  Result.Add('port', FServer.Port);
  Result.Add('enableSSL', FServer.EnableSSL);
  Result.Add('sslCertPath', FServer.SSLCertPath);
  Result.Add('maxRequestSize', FServer.MaxRequestSize);
  Result.Add('timeoutSeconds', FServer.TimeoutSeconds);
end;

function TAppConfig.BuildLoggingJSON: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('level', FLogging.Level);
  Result.Add('filePath', FLogging.FilePath);
  Result.Add('maxFileSize', FLogging.MaxFileSize);
  Result.Add('maxFiles', FLogging.MaxFiles);
  Result.Add('enableConsole', FLogging.EnableConsole);
end;

function TAppConfig.Save: boolean;
var
  JSONObj: TJSONObject;
  FlagsArr: TJSONArray;
  i: integer;
  ConfigStr: string;
  ConfigFile: TStringList;
begin
  Result := False;
  
  try
    JSONObj := TJSONObject.Create;
    try
      JSONObj.Add('appName', FAppName);
      JSONObj.Add('version', FVersion);
      JSONObj.Add('environment', FEnvironment);
      JSONObj.Add('database', BuildDatabaseJSON);
      JSONObj.Add('server', BuildServerJSON);
      JSONObj.Add('logging', BuildLoggingJSON);
      
      FlagsArr := TJSONArray.Create;
      for i := 0 to FFeatureFlags.Count - 1 do
        FlagsArr.Add(FFeatureFlags[i]);
      JSONObj.Add('featureFlags', FlagsArr);
      
      ConfigStr := JSONObj.FormatJSON;
    finally
      JSONObj.Free;
    end;
    
    ConfigFile := TStringList.Create;
    try
      ConfigFile.Text := ConfigStr;
      ConfigFile.SaveToFile(FConfigPath);
    finally
      ConfigFile.Free;
    end;
    
    Result := True;
    WriteLn('Config saved to: ', FConfigPath);
    
  except
    on E: Exception do
    begin
      WriteLn('Failed to save config: ', E.Message);
      Result := False;
    end;
  end;
end;

procedure TAppConfig.Display;
begin
  WriteLn('=== Application Configuration ===');
  WriteLn('App Name: ', FAppName);
  WriteLn('Version: ', FVersion);
  WriteLn('Environment: ', FEnvironment);
  WriteLn('');
  WriteLn('--- Database ---');
  WriteLn('Host: ', FDatabase.Host, ':', FDatabase.Port);
  WriteLn('Database: ', FDatabase.DatabaseName);
  WriteLn('Username: ', FDatabase.Username);
  WriteLn('Max Connections: ', FDatabase.MaxConnections);
  WriteLn('');
  WriteLn('--- Server ---');
  WriteLn('Listen: ', FServer.Host, ':', FServer.Port);
  WriteLn('SSL: ', BoolToStr(FServer.EnableSSL, 'Enabled', 'Disabled'));
  WriteLn('Timeout: ', FServer.TimeoutSeconds, 's');
  WriteLn('');
  WriteLn('--- Logging ---');
  WriteLn('Level: ', FLogging.Level);
  WriteLn('File: ', FLogging.FilePath);
  WriteLn('Console: ', BoolToStr(FLogging.EnableConsole, 'Yes', 'No'));
  WriteLn('');
  if FFeatureFlags.Count > 0 then
  begin
    WriteLn('--- Feature Flags ---');
    WriteLn(FFeatureFlags.CommaText);
  end;
end;

var
  Config: TAppConfig;
  ConfigPath: string;
begin
  ConfigPath := 'app_config.json';
  
  Config := TAppConfig.Create(ConfigPath);
  try
    // กำหนดค่า
    Config.AppName := 'MyShop ERP';
    Config.Version := '2.1.0';
    Config.Environment := 'production';
    
    Config.Database.Host := 'db.myshop.com';
    Config.Database.Port := 5432;
    Config.Database.DatabaseName := 'myshop_prod';
    Config.Database.Username := 'app_user';
    Config.Database.Password := 'secret123';
    Config.Database.MaxConnections := 50;
    
    Config.Server.Host := '0.0.0.0';
    Config.Server.Port := 443;
    Config.Server.EnableSSL := True;
    Config.Server.SSLCertPath := '/etc/ssl/myshop.crt';
    Config.Server.TimeoutSeconds := 60;
    
    Config.Logging.Level := 'warning';
    Config.Logging.FilePath := '/var/log/myshop/app.log';
    Config.Logging.EnableConsole := False;
    
    Config.FeatureFlags.Add('new-dashboard');
    Config.FeatureFlags.Add('ai-recommendations');
    Config.FeatureFlags.Add('multi-currency');
    
    // แสดงค่า
    Config.Display;
    
    // บันทึก
    if Config.Save then
    begin
      WriteLn('');
      WriteLn('บันทึก config สำเร็จ!');
      
      // ทดสอบโหลดกลับ
      WriteLn('');
      WriteLn('กำลังโหลด config กลับมา...');
    end;
    
    // โหลดกลับมา
    if Config.Load then
    begin
      WriteLn('');
      WriteLn('โหลด config สำเร็จ!');
      Config.Display;
    end;
    
  finally
    Config.Free;
  end;
  
  // ลบไฟล์ test
  if FileExists(ConfigPath) then
    DeleteFile(ConfigPath);
end.
```

---

## 41.11 Complete Example: API Client with JSON

```pascal
program APIClientWithJSON;

{$mode objfpc}{$H+}

uses
  fpjson, jsonparser, fphttpclient, openssl, opensslsockets,
  SysUtils, Classes;

type
  THTTPMethod = (hmGET, hmPOST, hmPUT, hmDELETE, hmPATCH);

  TAPIError = class(Exception)
  private
    FStatusCode: integer;
  public
    constructor Create(const Msg: string; StatusCode: integer);
    property StatusCode: integer read FStatusCode;
  end;
  
  TJSONAPIClient = class
  private
    FBaseURL: string;
    FAuthToken: string;
    FTimeout: integer;
    
    function DoRequest(const Method: THTTPMethod; const Endpoint: string;
      const Body: TJSONData = nil): TJSONData;
    function MethodToString(Method: THTTPMethod): string;
  public
    constructor Create(const BaseURL: string);
    
    // HTTP Methods
    function Get(const Endpoint: string): TJSONData;
    function Post(const Endpoint: string; const Body: TJSONObject): TJSONData;
    function Put(const Endpoint: string; const Body: TJSONObject): TJSONData;
    function Delete(const Endpoint: string): TJSONData;
    
    // Auth
    procedure SetBearerToken(const Token: string);
    
    property BaseURL: string read FBaseURL;
    property Timeout: integer read FTimeout write FTimeout;
  end;
  
  // User management client
  TUserAPIClient = class
  private
    FClient: TJSONAPIClient;
  public
    constructor Create(const BaseURL: string);
    destructor Destroy; override;
    
    function GetAllUsers: TJSONArray;
    function GetUser(UserID: integer): TJSONObject;
    function CreateUser(const Name, Email: string; Age: integer): TJSONObject;
    function UpdateUser(UserID: integer; const Name, Email: string): TJSONObject;
    function DeleteUser(UserID: integer): boolean;
    
    procedure SetAuthToken(const Token: string);
  end;

constructor TAPIError.Create(const Msg: string; StatusCode: integer);
begin
  inherited Create(Msg);
  FStatusCode := StatusCode;
end;

constructor TJSONAPIClient.Create(const BaseURL: string);
begin
  inherited Create;
  FBaseURL := BaseURL;
  FTimeout := 30000; // 30 seconds
end;

function TJSONAPIClient.MethodToString(Method: THTTPMethod): string;
begin
  case Method of
    hmGET: Result := 'GET';
    hmPOST: Result := 'POST';
    hmPUT: Result := 'PUT';
    hmDELETE: Result := 'DELETE';
    hmPATCH: Result := 'PATCH';
    else Result := 'GET';
  end;
end;

function TJSONAPIClient.DoRequest(const Method: THTTPMethod; 
  const Endpoint: string; const Body: TJSONData): TJSONData;
var
  HTTP: TFPHTTPClient;
  URL: string;
  RequestBody: string;
  ResponseStream: TStringStream;
  Response: string;
begin
  Result := nil;
  URL := FBaseURL + Endpoint;
  
  HTTP := TFPHTTPClient.Create(nil);
  try
    HTTP.ConnectTimeout := FTimeout;
    HTTP.ReadTimeout := FTimeout;
    
    // Set headers
    HTTP.AddHeader('Content-Type', 'application/json');
    HTTP.AddHeader('Accept', 'application/json');
    
    if FAuthToken <> '' then
      HTTP.AddHeader('Authorization', 'Bearer ' + FAuthToken);
    
    ResponseStream := TStringStream.Create('');
    try
      try
        case Method of
          hmGET:
            HTTP.Get(URL, ResponseStream);
          hmPOST:
          begin
            if Assigned(Body) then
              RequestBody := Body.AsJSON
            else
              RequestBody := '{}';
            HTTP.RequestBody := TStringStream.Create(RequestBody);
            HTTP.Post(URL, ResponseStream);
          end;
          hmPUT:
          begin
            if Assigned(Body) then
              RequestBody := Body.AsJSON
            else
              RequestBody := '{}';
            HTTP.RequestBody := TStringStream.Create(RequestBody);
            HTTP.Put(URL, ResponseStream);
          end;
          hmDELETE:
            HTTP.Delete(URL, ResponseStream);
        end;
        
        Response := ResponseStream.DataString;
        
        if HTTP.ResponseStatusCode >= 400 then
          raise TAPIError.Create(
            Format('HTTP %d: %s', [HTTP.ResponseStatusCode, Response]),
            HTTP.ResponseStatusCode
          );
          
        if Response <> '' then
          Result := GetJSON(Response);
          
      except
        on E: TAPIError do
          raise;
        on E: Exception do
          raise TAPIError.Create('Request failed: ' + E.Message, 0);
      end;
    finally
      ResponseStream.Free;
    end;
  finally
    HTTP.Free;
  end;
end;

function TJSONAPIClient.Get(const Endpoint: string): TJSONData;
begin
  Result := DoRequest(hmGET, Endpoint);
end;

function TJSONAPIClient.Post(const Endpoint: string; const Body: TJSONObject): TJSONData;
begin
  Result := DoRequest(hmPOST, Endpoint, Body);
end;

function TJSONAPIClient.Put(const Endpoint: string; const Body: TJSONObject): TJSONData;
begin
  Result := DoRequest(hmPUT, Endpoint, Body);
end;

function TJSONAPIClient.Delete(const Endpoint: string): TJSONData;
begin
  Result := DoRequest(hmDELETE, Endpoint);
end;

procedure TJSONAPIClient.SetBearerToken(const Token: string);
begin
  FAuthToken := Token;
end;

constructor TUserAPIClient.Create(const BaseURL: string);
begin
  inherited Create;
  FClient := TJSONAPIClient.Create(BaseURL);
end;

destructor TUserAPIClient.Destroy;
begin
  FClient.Free;
  inherited Destroy;
end;

function TUserAPIClient.GetAllUsers: TJSONArray;
var
  Response: TJSONData;
begin
  Result := nil;
  Response := FClient.Get('/users');
  if Assigned(Response) then
  begin
    if Response.JSONType = jtArray then
      Result := TJSONArray(Response)
    else
    begin
      Response.Free;
      raise Exception.Create('Expected array response');
    end;
  end;
end;

function TUserAPIClient.GetUser(UserID: integer): TJSONObject;
var
  Response: TJSONData;
begin
  Result := nil;
  Response := FClient.Get('/users/' + IntToStr(UserID));
  if Assigned(Response) then
  begin
    if Response.JSONType = jtObject then
      Result := TJSONObject(Response)
    else
    begin
      Response.Free;
      raise Exception.Create('Expected object response');
    end;
  end;
end;

function TUserAPIClient.CreateUser(const Name, Email: string; Age: integer): TJSONObject;
var
  RequestBody: TJSONObject;
  Response: TJSONData;
begin
  Result := nil;
  RequestBody := TJSONObject.Create;
  try
    RequestBody.Add('name', Name);
    RequestBody.Add('email', Email);
    RequestBody.Add('age', Age);
    
    Response := FClient.Post('/users', RequestBody);
    if Assigned(Response) and (Response.JSONType = jtObject) then
      Result := TJSONObject(Response)
    else
      Response.Free;
  finally
    RequestBody.Free;
  end;
end;

function TUserAPIClient.UpdateUser(UserID: integer; const Name, Email: string): TJSONObject;
var
  RequestBody: TJSONObject;
  Response: TJSONData;
begin
  Result := nil;
  RequestBody := TJSONObject.Create;
  try
    RequestBody.Add('name', Name);
    RequestBody.Add('email', Email);
    
    Response := FClient.Put('/users/' + IntToStr(UserID), RequestBody);
    if Assigned(Response) and (Response.JSONType = jtObject) then
      Result := TJSONObject(Response)
    else
      Response.Free;
  finally
    RequestBody.Free;
  end;
end;

function TUserAPIClient.DeleteUser(UserID: integer): boolean;
var
  Response: TJSONData;
begin
  try
    Response := FClient.Delete('/users/' + IntToStr(UserID));
    Response.Free;
    Result := True;
  except
    Result := False;
  end;
end;

procedure TUserAPIClient.SetAuthToken(const Token: string);
begin
  FClient.SetBearerToken(Token);
end;

// Demo โดยไม่ต้องมี server จริง (simulate)
procedure DemoAPIClient;
var
  // สาธิตการสร้าง JSON request/response
  RequestBody: TJSONObject;
  ResponseData: TJSONObject;
  UserList: TJSONArray;
  i: integer;
begin
  WriteLn('=== API Client Demo ===');
  WriteLn('');
  
  // สาธิต: สร้าง user request
  WriteLn('--- Create User Request ---');
  RequestBody := TJSONObject.Create;
  try
    RequestBody.Add('name', 'ทดสอบ ระบบ');
    RequestBody.Add('email', 'test@example.com');
    RequestBody.Add('age', 25);
    RequestBody.Add('role', 'customer');
    
    WriteLn('POST /api/users');
    WriteLn('Request Body:');
    WriteLn(RequestBody.FormatJSON);
  finally
    RequestBody.Free;
  end;
  
  // สาธิต: parse response
  WriteLn('');
  WriteLn('--- Server Response ---');
  ResponseData := TJSONObject.Create;
  try
    ResponseData.Add('success', True);
    ResponseData.Add('id', 1001);
    ResponseData.Add('name', 'ทดสอบ ระบบ');
    ResponseData.Add('email', 'test@example.com');
    ResponseData.Add('createdAt', '2024-01-15T12:00:00Z');
    
    WriteLn('Response:');
    WriteLn(ResponseData.FormatJSON);
    WriteLn('User created with ID: ', ResponseData.Get('id', 0));
  finally
    ResponseData.Free;
  end;
  
  // สาธิต: list users response
  WriteLn('');
  WriteLn('--- Users List Response ---');
  UserList := TJSONArray.Create;
  try
    for i := 1 to 3 do
    begin
      ResponseData := TJSONObject.Create;
      ResponseData.Add('id', i);
      ResponseData.Add('name', 'User ' + IntToStr(i));
      ResponseData.Add('email', Format('user%d@example.com', [i]));
      UserList.Add(ResponseData);
    end;
    
    WriteLn('GET /api/users');
    WriteLn('Found ', UserList.Count, ' users');
    for i := 0 to UserList.Count - 1 do
    begin
      ResponseData := TJSONObject(UserList.Items[i]);
      WriteLn(Format('  [%d] %s <%s>', [
        ResponseData.Get('id', 0),
        ResponseData.Get('name', ''),
        ResponseData.Get('email', '')
      ]));
    end;
  finally
    UserList.Free;
  end;
end;

begin
  DemoAPIClient;
end.
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1: JSON Parser พื้นฐาน
เขียนโปรแกรมที่รับ JSON string และแสดงข้อมูลในรูปแบบที่อ่านง่าย รองรับ string, number, boolean, null, array, object

### ข้อ 2: Student Grade System
สร้าง JSON structure สำหรับระบบเกรดนักเรียน รองรับ:
- ข้อมูลนักเรียนหลายคน
- แต่ละคนมีหลายวิชา
- แต่ละวิชามีคะแนนและเกรด
- คำนวณ GPA อัตโนมัติ

### ข้อ 3: JSON Config Loader
สร้าง config loader ที่:
- โหลด config จากไฟล์ JSON
- รองรับ default values
- Validate schema (required fields)
- Override ด้วย environment variables

### ข้อ 4: JSON to CSV
เขียนโปรแกรมแปลง JSON array ของ objects เป็น CSV file:
- ใช้ keys ของ object แรกเป็น header
- แปลง nested objects เป็น flattened fields
- รองรับ special characters

### ข้อ 5: Merge JSON Objects
เขียนฟังก์ชัน `MergeJSON(base, override: TJSONObject): TJSONObject` ที่:
- รวม properties จากทั้งสอง objects
- ถ้า key ซ้ำกัน ให้ override ชนะ
- รองรับ deep merge สำหรับ nested objects

### ข้อ 6: JSON Query
สร้าง simple JSON query engine ที่ใช้ dot notation:
```
"user.address.city" → ดึงค่า city จาก nested object
"items[0].name" → ดึงค่า name จาก element แรกของ array
"users.*.email" → ดึง email จาก users ทุกคน
```

### ข้อ 7: JSON Schema Validator
สร้าง JSON Schema validator ที่รองรับ:
- type validation (string, number, boolean, array, object)
- required fields
- min/max length สำหรับ string
- min/max value สำหรับ number
- array item type validation

### ข้อ 8: JSON Diff
เขียนโปรแกรม diff สอง JSON objects และแสดง:
- Fields ที่เพิ่มมา
- Fields ที่ลบออก
- Fields ที่เปลี่ยนแปลง (old value vs new value)

### ข้อ 9: JSON Cache Manager
สร้าง cache system ที่:
- บันทึก/โหลด data เป็น JSON
- รองรับ TTL (time to live)
- ลบ expired entries อัตโนมัติ
- สถิติ hit/miss

### ข้อ 10: REST API Mock Server
สร้าง mock REST API server ที่:
- อ่าน data จาก JSON file
- รองรับ GET, POST, PUT, DELETE
- Return JSON responses
- รองรับ filtering ด้วย query parameters

---

*หมายเหตุ: สำหรับโค้ดที่ใช้ `fphttpclient` ต้องติดตั้ง OpenSSL libraries และเพิ่ม `openssl`, `opensslsockets` ใน uses clause สำหรับ HTTPS support*
