# ตอนที่ 66: PDF Generation ใน Lazarus/Pascal

## บทนำ

การสร้าง PDF จาก Lazarus ทำได้หลายวิธี บทนี้จะครอบคลุม synPDF library และ FPReport สำหรับสร้าง PDF อย่างสมบูรณ์

---

## 66.1 ติดตั้ง PDF Libraries

### synPDF (Synopse PDF)
```bash
# ดาวน์โหลดจาก: https://synopse.info/fossil/wiki?name=Synopse+PDF
# หรือใช้ FPReport ที่มากับ Lazarus

# ใน Package Manager:
# - fpReport (มาพร้อม Lazarus)
# - SynPDF (ต้องติดตั้งเพิ่ม)
```

### FPReport (Built-in)
```pascal
// ใน uses ต้องเพิ่ม:
uses
  fpreport,
  fpreportpdfexport,
  fpreportpdftype;
```

---

## 66.2 สร้าง PDF ด้วย FPReport

```pascal
unit pdf_generator;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  fpreport,
  fpreportpdfexport;

type
  TPDFGenerator = class
  private
    FReport: TFPReport;
    FPage: TFPReportPage;
    FBand: TFPReportTitleBand;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure AddPage(AWidth, AHeight: Double);
    procedure AddText(const AText: string; AX, AY, AWidth, AHeight: Double;
      AFontSize: Integer = 10; ABold: Boolean = False);
    procedure AddLine(AX1, AY1, AX2, AY2: Double; AWidth: Double = 0.3);
    procedure AddRectangle(AX, AY, AWidth, AHeight: Double);
    procedure AddImage(const AFileName: string; AX, AY, AWidth, AHeight: Double);
    
    procedure SaveToFile(const AFileName: string);
    procedure SaveToStream(AStream: TStream);
  end;

implementation

constructor TPDFGenerator.Create;
begin
  inherited Create;
  FReport := TFPReport.Create(nil);
  FReport.Author := 'Lazarus Pascal App';
  FReport.Title := 'Generated PDF';
end;

destructor TPDFGenerator.Destroy;
begin
  FReport.Free;
  inherited Destroy;
end;

procedure TPDFGenerator.SaveToFile(const AFileName: string);
var
  Export: TFPReportPDFExporter;
begin
  Export := TFPReportPDFExporter.Create(nil);
  try
    Export.Report := FReport;
    Export.FileName := AFileName;
    FReport.RunReport;
    Export.Execute;
  finally
    Export.Free;
  end;
end;

end.
```

---

## 66.3 สร้าง PDF โดยตรงด้วย PDF Spec

```pascal
unit pdf_writer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Math;

type
  TPDFColor = record
    R, G, B: Double;  // 0.0 - 1.0
  end;

  TPDFFontType = (ftHelvetica, ftCourier, ftTimes, ftSymbol);

  TPDFTextAlign = (ptaLeft, ptaCenter, ptaRight);

  TPDFPageSize = (psA4, psA3, psLetter, psCustom);

  TPDFWriter = class
  private
    FObjects: TStringList;
    FPages: TList;
    FCurrentPage: Integer;
    FStream: TMemoryStream;
    FXRef: array of Int64;
    FObjectCount: Integer;
    
    FPageWidth, FPageHeight: Double;  // points (1 pt = 1/72 inch)
    FCurrentPageContent: TStringList;
    
    function AddObject(const AContent: string): Integer;
    function MMToPoints(AMM: Double): Double;
    function MakeColor(AR, AG, AB: Double): TPDFColor;
    procedure WriteStream(const AData: string);
    
    // PDF operators
    function OpMoveTo(AX, AY: Double): string;
    function OpLineTo(AX, AY: Double): string;
    function OpSetLineWidth(AWidth: Double): string;
    function OpSetColor(AColor: TPDFColor; AStroke: Boolean): string;
    function OpFillRect(AX, AY, AW, AH: Double; AFill: Boolean): string;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    // หน้า
    procedure BeginDocument;
    procedure AddPage(ASize: TPDFPageSize = psA4; 
      AWidth: Double = 0; AHeight: Double = 0);
    procedure EndDocument;
    
    // ข้อความ
    procedure DrawText(const AText: string; AX, AY: Double;
      AFontSize: Integer = 10; ABold: Boolean = False;
      AColor: TPDFColor = (R: 0; G: 0; B: 0));
    procedure DrawTextAligned(const AText: string; AX, AY, AWidth: Double;
      AAlign: TPDFTextAlign; AFontSize: Integer = 10);
    procedure DrawTextWrapped(const AText: string; AX, AY, AWidth: Double;
      ALineHeight: Double; AFontSize: Integer = 10);
    
    // รูปร่าง
    procedure DrawLine(AX1, AY1, AX2, AY2: Double; 
      AWidth: Double = 0.5; AColor: TPDFColor = (R: 0; G: 0; B: 0));
    procedure DrawRect(AX, AY, AWidth, AHeight: Double;
      ALineColor: TPDFColor = (R: 0; G: 0; B: 0);
      AFillColor: TPDFColor = (R: 1; G: 1; B: 1);
      AFill: Boolean = False; AStroke: Boolean = True);
    procedure DrawCircle(ACX, ACY, ARadius: Double;
      ALineColor: TPDFColor = (R: 0; G: 0; B: 0));
    
    // บันทึก
    procedure SaveToFile(const AFileName: string);
    procedure SaveToStream(AStream: TStream);
    
    property PageWidth: Double read FPageWidth;
    property PageHeight: Double read FPageHeight;
  end;

function PDFColor(AR, AG, AB: Byte): TPDFColor; overload;
function PDFColor(AHex: string): TPDFColor; overload;

implementation

function PDFColor(AR, AG, AB: Byte): TPDFColor;
begin
  Result.R := AR / 255;
  Result.G := AG / 255;
  Result.B := AB / 255;
end;

function PDFColor(AHex: string): TPDFColor;
var
  R, G, B: Integer;
begin
  if AHex[1] = '#' then Delete(AHex, 1, 1);
  R := StrToInt('$' + Copy(AHex, 1, 2));
  G := StrToInt('$' + Copy(AHex, 3, 2));
  B := StrToInt('$' + Copy(AHex, 5, 2));
  Result := PDFColor(R, G, B);
end;

{ TPDFWriter }

constructor TPDFWriter.Create;
begin
  inherited Create;
  FObjects := TStringList.Create;
  FPages := TList.Create;
  FStream := TMemoryStream.Create;
  FObjectCount := 0;
end;

destructor TPDFWriter.Destroy;
var
  i: Integer;
begin
  for i := 0 to FPages.Count - 1 do
    TStringList(FPages[i]).Free;
  FPages.Free;
  FObjects.Free;
  FStream.Free;
  if Assigned(FCurrentPageContent) then
    FCurrentPageContent.Free;
  inherited Destroy;
end;

function TPDFWriter.MMToPoints(AMM: Double): Double;
begin
  Result := AMM * 72 / 25.4;
end;

function TPDFWriter.AddObject(const AContent: string): Integer;
begin
  Inc(FObjectCount);
  Result := FObjectCount;
  FObjects.Add(IntToStr(Result) + ' 0 obj' + LineEnding + AContent + LineEnding + 'endobj');
end;

procedure TPDFWriter.WriteStream(const AData: string);
begin
  FStream.Write(AData[1], Length(AData));
end;

function TPDFWriter.OpMoveTo(AX, AY: Double): string;
begin
  Result := Format('%s %s m', [FormatFloat('0.##', AX), FormatFloat('0.##', AY)]);
end;

function TPDFWriter.OpLineTo(AX, AY: Double): string;
begin
  Result := Format('%s %s l', [FormatFloat('0.##', AX), FormatFloat('0.##', AY)]);
end;

function TPDFWriter.OpSetLineWidth(AWidth: Double): string;
begin
  Result := FormatFloat('0.##', AWidth) + ' w';
end;

function TPDFWriter.OpSetColor(AColor: TPDFColor; AStroke: Boolean): string;
var
  Op: string;
begin
  if AStroke then Op := 'RG' else Op := 'rg';
  Result := Format('%s %s %s %s',
    [FormatFloat('0.###', AColor.R),
     FormatFloat('0.###', AColor.G),
     FormatFloat('0.###', AColor.B), Op]);
end;

function TPDFWriter.OpFillRect(AX, AY, AW, AH: Double; AFill: Boolean): string;
var
  Op: string;
begin
  if AFill then Op := 'f' else Op := 'S';
  Result := Format('%s %s %s %s re %s',
    [FormatFloat('0.##', AX), FormatFloat('0.##', AY),
     FormatFloat('0.##', AW), FormatFloat('0.##', AH), Op]);
end;

procedure TPDFWriter.BeginDocument;
begin
  // PDF Header
  FStream.Clear;
  WriteStream('%PDF-1.7' + LineEnding);
  WriteStream('%âãÏÓ' + LineEnding);  // Binary marker
end;

procedure TPDFWriter.AddPage(ASize: TPDFPageSize; AWidth, AHeight: Double);
begin
  // บันทึกหน้าปัจจุบัน
  if Assigned(FCurrentPageContent) then
    FPages.Add(FCurrentPageContent);
    
  FCurrentPageContent := TStringList.Create;
  
  // กำหนดขนาดหน้า
  case ASize of
    psA4:     begin FPageWidth := 595; FPageHeight := 842; end;
    psA3:     begin FPageWidth := 842; FPageHeight := 1191; end;
    psLetter: begin FPageWidth := 612; FPageHeight := 792; end;
    psCustom: begin FPageWidth := MMToPoints(AWidth); FPageHeight := MMToPoints(AHeight); end;
  end;
  
  Inc(FCurrentPage);
  
  // Transform: PDF origin is bottom-left, แต่เราใช้ top-left
  FCurrentPageContent.Add(Format('q %s 0 0 %s 0 0 cm',
    [FormatFloat('0.##', 1.0),
     FormatFloat('0.##', 1.0)]));
end;

procedure TPDFWriter.DrawText(const AText: string; AX, AY: Double;
  AFontSize: Integer; ABold: Boolean; AColor: TPDFColor);
var
  FontName: string;
  TransY: Double;
begin
  if ABold then FontName := 'F2' else FontName := 'F1';
  
  // แปลง Y จาก top-left เป็น bottom-left
  TransY := FPageHeight - AY - AFontSize;
  
  FCurrentPageContent.Add('BT');
  FCurrentPageContent.Add(OpSetColor(AColor, False));
  FCurrentPageContent.Add('/' + FontName + ' ' + IntToStr(AFontSize) + ' Tf');
  FCurrentPageContent.Add(Format('%s %s Td', 
    [FormatFloat('0.##', AX), FormatFloat('0.##', TransY)]));
  FCurrentPageContent.Add('(' + AText + ') Tj');
  FCurrentPageContent.Add('ET');
end;

procedure TPDFWriter.DrawLine(AX1, AY1, AX2, AY2: Double; 
  AWidth: Double; AColor: TPDFColor);
var
  TY1, TY2: Double;
begin
  TY1 := FPageHeight - AY1;
  TY2 := FPageHeight - AY2;
  
  FCurrentPageContent.Add('q');
  FCurrentPageContent.Add(OpSetColor(AColor, True));
  FCurrentPageContent.Add(OpSetLineWidth(AWidth));
  FCurrentPageContent.Add(OpMoveTo(AX1, TY1));
  FCurrentPageContent.Add(OpLineTo(AX2, TY2));
  FCurrentPageContent.Add('S');
  FCurrentPageContent.Add('Q');
end;

procedure TPDFWriter.DrawRect(AX, AY, AWidth, AHeight: Double;
  ALineColor, AFillColor: TPDFColor; AFill, AStroke: Boolean);
var
  TransY: Double;
  Op: string;
begin
  TransY := FPageHeight - AY - AHeight;
  
  FCurrentPageContent.Add('q');
  
  if AFill and AStroke then
  begin
    FCurrentPageContent.Add(OpSetColor(AFillColor, False));
    FCurrentPageContent.Add(OpSetColor(ALineColor, True));
    Op := 'B';
  end
  else if AFill then
  begin
    FCurrentPageContent.Add(OpSetColor(AFillColor, False));
    Op := 'f';
  end
  else
  begin
    FCurrentPageContent.Add(OpSetColor(ALineColor, True));
    Op := 'S';
  end;
  
  FCurrentPageContent.Add(Format('%s %s %s %s re %s',
    [FormatFloat('0.##', AX), FormatFloat('0.##', TransY),
     FormatFloat('0.##', AWidth), FormatFloat('0.##', AHeight), Op]));
  FCurrentPageContent.Add('Q');
end;

procedure TPDFWriter.DrawCircle(ACX, ACY, ARadius: Double; ALineColor: TPDFColor);
var
  K: Double;
  TX, TY: Double;
begin
  // Bezier approximation of circle
  K := 0.5523;
  TX := ACX;
  TY := FPageHeight - ACY;
  
  FCurrentPageContent.Add('q');
  FCurrentPageContent.Add(OpSetColor(ALineColor, True));
  FCurrentPageContent.Add(Format('%s %s m',
    [FormatFloat('0.##', TX - ARadius), FormatFloat('0.##', TY)]));
  FCurrentPageContent.Add(Format('%s %s %s %s %s %s c',
    [FormatFloat('0.##', TX - ARadius), FormatFloat('0.##', TY + ARadius * K),
     FormatFloat('0.##', TX - ARadius * K), FormatFloat('0.##', TY + ARadius),
     FormatFloat('0.##', TX), FormatFloat('0.##', TY + ARadius)]));
  // ... (4 bezier curves for full circle)
  FCurrentPageContent.Add('S');
  FCurrentPageContent.Add('Q');
end;

procedure TPDFWriter.DrawTextAligned(const AText: string; AX, AY, AWidth: Double;
  AAlign: TPDFTextAlign; AFontSize: Integer);
begin
  // Simplified - ใช้ DrawText
  case AAlign of
    ptaLeft:   DrawText(AText, AX, AY, AFontSize);
    ptaCenter: DrawText(AText, AX + AWidth/2, AY, AFontSize);  // approximate
    ptaRight:  DrawText(AText, AX + AWidth, AY, AFontSize);
  end;
end;

procedure TPDFWriter.DrawTextWrapped(const AText: string; AX, AY, AWidth: Double;
  ALineHeight: Double; AFontSize: Integer);
var
  Words: TStringList;
  Line: string;
  Y: Double;
  i: Integer;
begin
  Words := TStringList.Create;
  try
    Words.Delimiter := ' ';
    Words.DelimitedText := AText;
    
    Line := '';
    Y := AY;
    
    for i := 0 to Words.Count - 1 do
    begin
      if Line = '' then
        Line := Words[i]
      else
        Line := Line + ' ' + Words[i];
        
      // ประมาณความกว้าง (6 pixel ต่อตัวอักษร ที่ size 10)
      if Length(Line) * AFontSize * 0.5 > AWidth then
      begin
        DrawText(Line, AX, Y, AFontSize);
        Y := Y + ALineHeight;
        Line := Words[i];
      end;
    end;
    
    if Line <> '' then
      DrawText(Line, AX, Y, AFontSize);
  finally
    Words.Free;
  end;
end;

procedure TPDFWriter.EndDocument;
var
  i: Integer;
  Content: string;
  PageObjNums: array of Integer;
  ResourcesObj, PagesObj, RootObj: Integer;
  XRefPos: Int64;
  PageContentObj: Integer;
begin
  // บันทึกหน้าสุดท้าย
  if Assigned(FCurrentPageContent) then
    FPages.Add(FCurrentPageContent);
    
  // สร้าง Objects
  SetLength(PageObjNums, FPages.Count);
  
  // Resources object (fonts)
  ResourcesObj := AddObject(
    '<< /Font << ' +
    '/F1 << /Type /Font /Subtype /Type1 /BaseFont /Helvetica /Encoding /WinAnsiEncoding >> ' +
    '/F2 << /Type /Font /Subtype /Type1 /BaseFont /Helvetica-Bold /Encoding /WinAnsiEncoding >> ' +
    '/F3 << /Type /Font /Subtype /Type1 /BaseFont /Times-Roman /Encoding /WinAnsiEncoding >> ' +
    '>> >>');
    
  // Page objects
  for i := 0 to FPages.Count - 1 do
  begin
    // Page content
    Content := TStringList(FPages[i]).Text;
    PageContentObj := AddObject(
      '<< /Length ' + IntToStr(Length(Content)) + ' >>' + LineEnding +
      'stream' + LineEnding +
      Content +
      'endstream');
      
    // Page object
    PageObjNums[i] := AddObject(
      '<< /Type /Page /MediaBox [0 0 ' + 
      FormatFloat('0.##', FPageWidth) + ' ' + 
      FormatFloat('0.##', FPageHeight) + '] ' +
      '/Resources ' + IntToStr(ResourcesObj) + ' 0 R ' +
      '/Contents ' + IntToStr(PageContentObj) + ' 0 R ' +
      '/Parent {PAGES} 0 R ' +
      '>>');
  end;
  
  // Pages object
  var KidsStr := '';
  for i := 0 to High(PageObjNums) do
  begin
    if i > 0 then KidsStr := KidsStr + ' ';
    KidsStr := KidsStr + IntToStr(PageObjNums[i]) + ' 0 R';
  end;
  
  PagesObj := AddObject(
    '<< /Type /Pages /Kids [' + KidsStr + '] /Count ' + 
    IntToStr(FPages.Count) + ' >>');
    
  // Fix parent references
  for i := 0 to FObjects.Count - 1 do
    FObjects[i] := StringReplace(FObjects[i], '{PAGES}', 
      IntToStr(PagesObj), [rfReplaceAll]);
      
  // Catalog (root)
  RootObj := AddObject(
    '<< /Type /Catalog /Pages ' + IntToStr(PagesObj) + ' 0 R >>');
    
  // Write all objects
  SetLength(FXRef, FObjectCount + 1);
  FXRef[0] := FStream.Position;
  
  WriteStream(LineEnding);
  
  for i := 0 to FObjects.Count - 1 do
  begin
    FXRef[i + 1] := FStream.Position;
    WriteStream(FObjects[i] + LineEnding + LineEnding);
  end;
  
  // Cross-reference table
  XRefPos := FStream.Position;
  WriteStream('xref' + LineEnding);
  WriteStream('0 ' + IntToStr(FObjectCount + 1) + LineEnding);
  WriteStream('0000000000 65535 f ' + LineEnding);
  
  for i := 1 to FObjectCount do
    WriteStream(Format('%010d 00000 n ' + LineEnding, [FXRef[i]]));
    
  // Trailer
  WriteStream('trailer' + LineEnding);
  WriteStream('<< /Size ' + IntToStr(FObjectCount + 1) + 
    ' /Root ' + IntToStr(RootObj) + ' 0 R >>' + LineEnding);
  WriteStream('startxref' + LineEnding);
  WriteStream(IntToStr(XRefPos) + LineEnding);
  WriteStream('%%EOF' + LineEnding);
end;

procedure TPDFWriter.SaveToFile(const AFileName: string);
begin
  FStream.SaveToFile(AFileName);
end;

procedure TPDFWriter.SaveToStream(AStream: TStream);
begin
  FStream.Position := 0;
  AStream.CopyFrom(FStream, FStream.Size);
end;

end.
```

---

## 66.4 ตัวอย่าง: สร้างใบแจ้งหนี้ PDF

```pascal
unit pdf_invoice;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics,
  pdf_writer;

type
  TInvoiceItem = record
    Description: string;
    Qty: Integer;
    UnitPrice: Double;
    Amount: Double;
  end;

  TPDFInvoice = class
  private
    FPDF: TPDFWriter;
    FMarginL, FMarginT, FMarginR, FMarginB: Double;
    FCurrentY: Double;
    
    // สี
    FColorBlue: TPDFColor;
    FColorDarkBlue: TPDFColor;
    FColorGray: TPDFColor;
    FColorLightGray: TPDFColor;
    FColorWhite: TPDFColor;
    FColorBlack: TPDFColor;
    FColorRed: TPDFColor;
    
    function W: Double;  // Page width
    function H: Double;  // Page height
    function ContentW: Double;
    
    procedure DrawHeader(const ACompanyName, AAddress, APhone, ATaxID: string);
    procedure DrawInvoiceInfo(const AInvoiceNo: string; ADate, ADueDate: TDate);
    procedure DrawCustomerInfo(const AName, AAddress, APhone: string);
    procedure DrawItemsTable(const AItems: array of TInvoiceItem);
    procedure DrawTotals(ASubTotal, ADiscount, AVAT, ATotal: Double);
    procedure DrawFooter(const ANote, ABankInfo: string);
    procedure DrawWatermark(const AText: string);
    
    procedure DrawFilledRect(AX, AY, AW, AH: Double; AColor: TPDFColor);
    procedure DrawBorderedRect(AX, AY, AW, AH: Double; 
      ABorderColor, AFillColor: TPDFColor);
    procedure DrawColoredText(const AText: string; AX, AY: Double;
      ASize: Integer; AColor: TPDFColor; ABold: Boolean = False);
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Generate(
      // บริษัท
      const ACompanyName, ACompanyAddress, ACompanyPhone, ACompanyTaxID: string;
      // ลูกค้า
      const ACustomerName, ACustomerAddress, ACustomerPhone: string;
      // ใบแจ้งหนี้
      const AInvoiceNo: string; ADate, ADueDate: TDate;
      // รายการ
      const AItems: array of TInvoiceItem;
      // ยอดเงิน
      ADiscountPct, AVATPct: Double;
      // หมายเหตุ
      const ANote, ABankInfo: string;
      // บันทึก
      const AOutputFile: string;
      AAddWatermark: Boolean = False);
  end;

implementation

{ TPDFInvoice }

constructor TPDFInvoice.Create;
begin
  inherited Create;
  FPDF := TPDFWriter.Create;
  
  // Margins (mm -> points: x * 72/25.4)
  FMarginL := 20 * 72 / 25.4;
  FMarginT := 15 * 72 / 25.4;
  FMarginR := 20 * 72 / 25.4;
  FMarginB := 20 * 72 / 25.4;
  
  // สี
  FColorBlue := PDFColor(52, 152, 219);
  FColorDarkBlue := PDFColor(44, 62, 80);
  FColorGray := PDFColor(127, 140, 141);
  FColorLightGray := PDFColor(236, 240, 241);
  FColorWhite := PDFColor(255, 255, 255);
  FColorBlack := PDFColor(0, 0, 0);
  FColorRed := PDFColor(231, 76, 60);
end;

destructor TPDFInvoice.Destroy;
begin
  FPDF.Free;
  inherited Destroy;
end;

function TPDFInvoice.W: Double;
begin
  Result := FPDF.PageWidth;
end;

function TPDFInvoice.H: Double;
begin
  Result := FPDF.PageHeight;
end;

function TPDFInvoice.ContentW: Double;
begin
  Result := W - FMarginL - FMarginR;
end;

procedure TPDFInvoice.DrawFilledRect(AX, AY, AW, AH: Double; AColor: TPDFColor);
begin
  FPDF.DrawRect(AX, AY, AW, AH, AColor, AColor, True, False);
end;

procedure TPDFInvoice.DrawBorderedRect(AX, AY, AW, AH: Double; 
  ABorderColor, AFillColor: TPDFColor);
begin
  FPDF.DrawRect(AX, AY, AW, AH, ABorderColor, AFillColor, True, True);
end;

procedure TPDFInvoice.DrawColoredText(const AText: string; AX, AY: Double;
  ASize: Integer; AColor: TPDFColor; ABold: Boolean);
begin
  FPDF.DrawText(AText, AX, AY, ASize, ABold, AColor);
end;

procedure TPDFInvoice.DrawHeader(const ACompanyName, AAddress, APhone, ATaxID: string);
begin
  // แถบหัว - สีน้ำเงินเข้ม
  DrawFilledRect(0, 0, W, 30 * 72/25.4, FColorDarkBlue);
  
  // ชื่อบริษัท (ขาว, ใหญ่)
  DrawColoredText(ACompanyName, FMarginL, 8 * 72/25.4, 18, FColorWhite, True);
  
  // แถบสีน้ำเงินอ่อน
  DrawFilledRect(0, 30 * 72/25.4, W, 20 * 72/25.4, FColorBlue);
  
  // ข้อมูลบริษัท (ขาว)
  DrawColoredText(AAddress, FMarginL, 33 * 72/25.4, 9, FColorWhite);
  DrawColoredText('โทร: ' + APhone + '   เลขที่ผู้เสียภาษี: ' + ATaxID, 
    FMarginL, 40 * 72/25.4, 9, FColorWhite);
    
  // หัว "ใบแจ้งหนี้" ด้านขวา
  DrawColoredText('ใบแจ้งหนี้', W - FMarginR - 80 * 72/25.4, 10 * 72/25.4, 20, FColorWhite, True);
  DrawColoredText('TAX INVOICE', W - FMarginR - 80 * 72/25.4, 20 * 72/25.4, 10, FColorLightGray);
  
  FCurrentY := 55 * 72/25.4;
end;

procedure TPDFInvoice.DrawInvoiceInfo(const AInvoiceNo: string; ADate, ADueDate: TDate);
var
  InfoX: Double;
begin
  InfoX := W - FMarginR - 70 * 72/25.4;
  
  // กรอบข้อมูลใบแจ้งหนี้
  DrawBorderedRect(InfoX - 5, FCurrentY, 75 * 72/25.4, 25 * 72/25.4,
    FColorLightGray, FColorLightGray);
    
  DrawColoredText('เลขที่:', InfoX, FCurrentY + 3 * 72/25.4, 9, FColorGray);
  DrawColoredText(AInvoiceNo, InfoX + 15 * 72/25.4, FCurrentY + 3 * 72/25.4, 9, FColorBlack, True);
  
  DrawColoredText('วันที่:', InfoX, FCurrentY + 10 * 72/25.4, 9, FColorGray);
  DrawColoredText(FormatDateTime('dd/mm/yyyy', ADate), 
    InfoX + 15 * 72/25.4, FCurrentY + 10 * 72/25.4, 9, FColorBlack);
    
  DrawColoredText('ครบกำหนด:', InfoX, FCurrentY + 17 * 72/25.4, 9, FColorGray);
  DrawColoredText(FormatDateTime('dd/mm/yyyy', ADueDate), 
    InfoX + 20 * 72/25.4, FCurrentY + 17 * 72/25.4, 9, FColorRed);
end;

procedure TPDFInvoice.DrawCustomerInfo(const AName, AAddress, APhone: string);
begin
  DrawColoredText('ลูกค้า:', FMarginL, FCurrentY, 9, FColorGray);
  DrawColoredText(AName, FMarginL + 15 * 72/25.4, FCurrentY, 11, FColorBlack, True);
  FCurrentY := FCurrentY + 6 * 72/25.4;
  
  DrawColoredText('ที่อยู่:', FMarginL, FCurrentY, 9, FColorGray);
  DrawColoredText(AAddress, FMarginL + 15 * 72/25.4, FCurrentY, 9, FColorBlack);
  FCurrentY := FCurrentY + 5 * 72/25.4;
  
  DrawColoredText('โทร:', FMarginL, FCurrentY, 9, FColorGray);
  DrawColoredText(APhone, FMarginL + 15 * 72/25.4, FCurrentY, 9, FColorBlack);
  FCurrentY := FCurrentY + 10 * 72/25.4;
  
  // เส้นคั่น
  FPDF.DrawLine(FMarginL, FCurrentY, W - FMarginR, FCurrentY, 0.5, FColorLightGray);
  FCurrentY := FCurrentY + 5 * 72/25.4;
end;

procedure TPDFInvoice.DrawItemsTable(const AItems: array of TInvoiceItem);
const
  COL_NUM = 0.08;   // 8%
  COL_DESC = 0.40;  // 40%
  COL_QTY = 0.12;   // 12%
  COL_PRICE = 0.20; // 20%
  COL_TOTAL = 0.20; // 20%

var
  i: Integer;
  X1, X2, X3, X4, X5: Double;
  RowHeight: Double;
  IsEven: Boolean;
begin
  RowHeight := 7 * 72/25.4;
  X1 := FMarginL;
  X2 := X1 + ContentW * COL_NUM;
  X3 := X2 + ContentW * COL_DESC;
  X4 := X3 + ContentW * COL_QTY;
  X5 := X4 + ContentW * COL_PRICE;
  
  // Header row
  DrawFilledRect(FMarginL, FCurrentY, ContentW, RowHeight, FColorDarkBlue);
  
  DrawColoredText('#', X1 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorWhite, True);
  DrawColoredText('รายการ', X2 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorWhite, True);
  DrawColoredText('จำนวน', X3 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorWhite, True);
  DrawColoredText('ราคา/หน่วย', X4 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorWhite, True);
  DrawColoredText('รวม', X5 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorWhite, True);
  
  FCurrentY := FCurrentY + RowHeight;
  
  // Data rows
  for i := 0 to High(AItems) do
  begin
    IsEven := (i mod 2 = 0);
    var RowColor := FColorWhite;
    if not IsEven then RowColor := FColorLightGray;
    
    DrawFilledRect(FMarginL, FCurrentY, ContentW, RowHeight, RowColor);
    
    DrawColoredText(IntToStr(i + 1), X1 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorBlack);
    DrawColoredText(AItems[i].Description, X2 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorBlack);
    DrawColoredText(IntToStr(AItems[i].Qty), X3 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorBlack);
    DrawColoredText(FormatFloat('#,##0.00', AItems[i].UnitPrice), 
      X4 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorBlack);
    DrawColoredText(FormatFloat('#,##0.00', AItems[i].Amount), 
      X5 + 2 * 72/25.4, FCurrentY + 2 * 72/25.4, 9, FColorBlack);
    
    FCurrentY := FCurrentY + RowHeight;
  end;
  
  // เส้นล่าง
  FPDF.DrawLine(FMarginL, FCurrentY, W - FMarginR, FCurrentY, 1, FColorDarkBlue);
  FCurrentY := FCurrentY + 5 * 72/25.4;
end;

procedure TPDFInvoice.DrawTotals(ASubTotal, ADiscount, AVAT, ATotal: Double);
var
  TotX, TotW: Double;
begin
  TotX := W - FMarginR - 70 * 72/25.4;
  TotW := 70 * 72/25.4;
  
  DrawColoredText('ยอดรวมก่อน VAT:', TotX, FCurrentY, 9, FColorGray);
  DrawColoredText(FormatFloat('#,##0.00 บ.', ASubTotal), 
    W - FMarginR - 25 * 72/25.4, FCurrentY, 9, FColorBlack);
  FCurrentY := FCurrentY + 5 * 72/25.4;
  
  if ADiscount > 0 then
  begin
    DrawColoredText(Format('ส่วนลด (%.0f%%):', [ADiscount]), TotX, FCurrentY, 9, FColorGray);
    DrawColoredText(FormatFloat('#,##0.00 บ.', ASubTotal * ADiscount / 100), 
      W - FMarginR - 25 * 72/25.4, FCurrentY, 9, FColorRed);
    FCurrentY := FCurrentY + 5 * 72/25.4;
  end;
  
  if AVAT > 0 then
  begin
    DrawColoredText(Format('VAT (%.0f%%):', [AVAT]), TotX, FCurrentY, 9, FColorGray);
    var VATAmt := ASubTotal * (1 - ADiscount/100) * AVAT / 100;
    DrawColoredText(FormatFloat('#,##0.00 บ.', VATAmt), 
      W - FMarginR - 25 * 72/25.4, FCurrentY, 9, FColorBlack);
    FCurrentY := FCurrentY + 5 * 72/25.4;
  end;
  
  // เส้นคั่น
  FPDF.DrawLine(TotX, FCurrentY, W - FMarginR, FCurrentY, 1, FColorDarkBlue);
  FCurrentY := FCurrentY + 3 * 72/25.4;
  
  // ยอดรวมสุทธิ (โดดเด่น)
  DrawFilledRect(TotX - 5 * 72/25.4, FCurrentY, 
    TotW + 5 * 72/25.4, 10 * 72/25.4, FColorDarkBlue);
    
  DrawColoredText('ยอดรวมสุทธิ:', TotX, FCurrentY + 2 * 72/25.4, 10, FColorWhite, True);
  DrawColoredText(FormatFloat('#,##0.00 บ.', ATotal), 
    W - FMarginR - 30 * 72/25.4, FCurrentY + 2 * 72/25.4, 12, FColorWhite, True);
    
  FCurrentY := FCurrentY + 15 * 72/25.4;
end;

procedure TPDFInvoice.DrawFooter(const ANote, ABankInfo: string);
begin
  // เส้นคั่น
  FPDF.DrawLine(FMarginL, FCurrentY, W - FMarginR, FCurrentY, 0.5, FColorGray);
  FCurrentY := FCurrentY + 5 * 72/25.4;
  
  if ANote <> '' then
  begin
    DrawColoredText('หมายเหตุ:', FMarginL, FCurrentY, 9, FColorGray);
    FCurrentY := FCurrentY + 5 * 72/25.4;
    FPDF.DrawTextWrapped(ANote, FMarginL, FCurrentY, ContentW, 6 * 72/25.4, 9);
    FCurrentY := FCurrentY + 12 * 72/25.4;
  end;
  
  if ABankInfo <> '' then
  begin
    DrawColoredText('ข้อมูลบัญชีธนาคาร:', FMarginL, FCurrentY, 9, FColorGray);
    FCurrentY := FCurrentY + 5 * 72/25.4;
    DrawColoredText(ABankInfo, FMarginL, FCurrentY, 9, FColorBlue);
  end;
end;

procedure TPDFInvoice.DrawWatermark(const AText: string);
begin
  // วาด watermark (ข้อความกลางหน้า, หมุน 45 องศา)
  // ต้องใช้ PDF transform matrix
  FPDF.DrawText(AText, W/2 - 50, H/2, 48, True, 
    PDFColor(200, 200, 200));
end;

procedure TPDFInvoice.Generate(
  const ACompanyName, ACompanyAddress, ACompanyPhone, ACompanyTaxID: string;
  const ACustomerName, ACustomerAddress, ACustomerPhone: string;
  const AInvoiceNo: string; ADate, ADueDate: TDate;
  const AItems: array of TInvoiceItem;
  ADiscountPct, AVATPct: Double;
  const ANote, ABankInfo: string;
  const AOutputFile: string;
  AAddWatermark: Boolean);
var
  SubTotal, Total: Double;
  i: Integer;
begin
  FPDF.BeginDocument;
  FPDF.AddPage(psA4);
  
  // วาด watermark ก่อน (อยู่ข้างหลัง)
  if AAddWatermark then
    DrawWatermark('ต้นฉบับ');
  
  DrawHeader(ACompanyName, ACompanyAddress, ACompanyPhone, ACompanyTaxID);
  DrawInvoiceInfo(AInvoiceNo, ADate, ADueDate);
  
  FCurrentY := FCurrentY + 5 * 72/25.4;
  DrawCustomerInfo(ACustomerName, ACustomerAddress, ACustomerPhone);
  
  DrawItemsTable(AItems);
  
  // คำนวณยอดเงิน
  SubTotal := 0;
  for i := 0 to High(AItems) do
    SubTotal := SubTotal + AItems[i].Amount;
  Total := SubTotal * (1 - ADiscountPct/100) * (1 + AVATPct/100);
  
  DrawTotals(SubTotal, ADiscountPct, AVATPct, Total);
  DrawFooter(ANote, ABankInfo);
  
  FPDF.EndDocument;
  FPDF.SaveToFile(AOutputFile);
  
  WriteLn('สร้าง PDF สำเร็จ: ' + AOutputFile);
end;

// ตัวอย่างการใช้งาน
procedure DemoCreatePDF;
var
  PDF: TPDFInvoice;
  Items: array[0..2] of TInvoiceItem;
begin
  Items[0] := (Description: 'บริการพัฒนาโปรแกรม'; Qty: 1; UnitPrice: 50000; Amount: 50000);
  Items[1] := (Description: 'บำรุงรักษารายปี'; Qty: 1; UnitPrice: 10000; Amount: 10000);
  Items[2] := (Description: 'ฝึกอบรมผู้ใช้งาน (3 ชั่วโมง)'; Qty: 3; UnitPrice: 2000; Amount: 6000);
  
  PDF := TPDFInvoice.Create;
  try
    PDF.Generate(
      'บริษัท พาสคัล เทค จำกัด',
      '789 ถนนรัชดาภิเษก แขวงดินแดง เขตดินแดง กรุงเทพ 10400',
      '02-999-8888',
      '0105566099999',
      
      'บริษัท ลูกค้า จำกัด',
      '100 ถนนสาทร กรุงเทพ',
      '081-999-7777',
      
      'INV-2024-00099',
      Today,
      Today + 30,
      
      Items,
      5.0,  // discount 5%
      7.0,  // VAT 7%
      
      'กรุณาชำระเงินภายใน 30 วันนับจากวันที่ออกใบแจ้งหนี้',
      'ธนาคารไทยพาณิชย์ สาขาสีลม เลขที่ 001-234567-8',
      
      'invoice_2024_00099.pdf',
      True  // watermark
    );
  finally
    PDF.Free;
  end;
end;

end.
```

---

## 66.5 PDF Report Builder

```pascal
unit pdf_report;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  pdf_writer;

type
  TReportSection = (rsSummary, rsDetail, rsChart);

  TPDFReport = class
  private
    FPDF: TPDFWriter;
    FTitle: string;
    FSubtitle: string;
    FCurrentY: Double;
    FMargin: Double;
    
  public
    constructor Create(const ATitle, ASubtitle: string);
    destructor Destroy; override;
    
    procedure BeginReport;
    procedure EndReport(const AOutputFile: string);
    procedure AddSection(const ASectionTitle: string);
    procedure AddParagraph(const AText: string);
    procedure AddTable(const AHeaders: array of string;
      const ARows: array of TStringDynArray;
      const AWidths: array of Double);
    procedure AddPageBreak;
    procedure AddSpacer(AHeight: Double = 5);
    procedure AddHorizontalLine;
    procedure AddKeyValue(const AKey, AValue: string);
  end;

implementation

constructor TPDFReport.Create(const ATitle, ASubtitle: string);
begin
  inherited Create;
  FTitle := ATitle;
  FSubtitle := ASubtitle;
  FPDF := TPDFWriter.Create;
  FMargin := 15 * 72/25.4;  // 15mm
end;

destructor TPDFReport.Destroy;
begin
  FPDF.Free;
  inherited Destroy;
end;

procedure TPDFReport.BeginReport;
var
  TitleColor, SubColor: TPDFColor;
begin
  FPDF.BeginDocument;
  FPDF.AddPage(psA4);
  
  TitleColor := PDFColor(44, 62, 80);
  SubColor := PDFColor(127, 140, 141);
  
  // ชื่อ report
  FPDF.DrawText(FTitle, FMargin, 10 * 72/25.4, 20, True, TitleColor);
  FPDF.DrawText(FSubtitle, FMargin, 18 * 72/25.4, 11, False, SubColor);
  
  // เส้นคั่น
  FPDF.DrawLine(FMargin, 26 * 72/25.4, FPDF.PageWidth - FMargin, 26 * 72/25.4,
    2, PDFColor(52, 152, 219));
    
  // วันที่
  FPDF.DrawText('สร้างเมื่อ: ' + FormatDateTime('dd/mm/yyyy hh:nn', Now),
    FPDF.PageWidth - FMargin - 50 * 72/25.4, 10 * 72/25.4, 8, False, SubColor);
    
  FCurrentY := 32 * 72/25.4;
end;

procedure TPDFReport.EndReport(const AOutputFile: string);
begin
  FPDF.EndDocument;
  FPDF.SaveToFile(AOutputFile);
end;

procedure TPDFReport.AddSection(const ASectionTitle: string);
begin
  AddSpacer(5);
  
  // แถบสี
  FPDF.DrawRect(FMargin, FCurrentY, FPDF.PageWidth - 2 * FMargin, 8 * 72/25.4,
    PDFColor(52, 152, 219), PDFColor(52, 152, 219), True, False);
    
  FPDF.DrawText(ASectionTitle, FMargin + 3 * 72/25.4, FCurrentY + 1.5 * 72/25.4,
    10, True, PDFColor(255, 255, 255));
    
  FCurrentY := FCurrentY + 8 * 72/25.4 + 3 * 72/25.4;
end;

procedure TPDFReport.AddParagraph(const AText: string);
begin
  FPDF.DrawTextWrapped(AText, FMargin, FCurrentY, 
    FPDF.PageWidth - 2 * FMargin, 6 * 72/25.4, 10);
  // approximate height
  var Lines := Length(AText) div 80 + 1;
  FCurrentY := FCurrentY + Lines * 6 * 72/25.4 + 3 * 72/25.4;
end;

procedure TPDFReport.AddTable(const AHeaders: array of string;
  const ARows: array of TStringDynArray; const AWidths: array of Double);
var
  i, j: Integer;
  X: Double;
  RowH: Double;
  TotalW: Double;
begin
  RowH := 7 * 72/25.4;
  TotalW := 0;
  for i := 0 to High(AWidths) do TotalW := TotalW + AWidths[i] * 72/25.4;
  
  // Header
  FPDF.DrawRect(FMargin, FCurrentY, TotalW, RowH,
    PDFColor(44, 62, 80), PDFColor(44, 62, 80), True, False);
    
  X := FMargin;
  for i := 0 to High(AHeaders) do
  begin
    FPDF.DrawText(AHeaders[i], X + 2 * 72/25.4, FCurrentY + 1.5 * 72/25.4,
      9, True, PDFColor(255, 255, 255));
    X := X + AWidths[i] * 72/25.4;
  end;
  FCurrentY := FCurrentY + RowH;
  
  // Rows
  for i := 0 to High(ARows) do
  begin
    var BgColor := PDFColor(255, 255, 255);
    if i mod 2 = 1 then BgColor := PDFColor(236, 240, 241);
    
    FPDF.DrawRect(FMargin, FCurrentY, TotalW, RowH,
      PDFColor(200, 200, 200), BgColor, True, True);
      
    X := FMargin;
    for j := 0 to High(ARows[i]) do
    begin
      if j <= High(AWidths) then
      begin
        FPDF.DrawText(ARows[i][j], X + 2 * 72/25.4, FCurrentY + 1.5 * 72/25.4,
          9, False, PDFColor(0, 0, 0));
        X := X + AWidths[j] * 72/25.4;
      end;
    end;
    
    FCurrentY := FCurrentY + RowH;
  end;
  
  FCurrentY := FCurrentY + 5 * 72/25.4;
end;

procedure TPDFReport.AddPageBreak;
begin
  FPDF.AddPage(psA4);
  FCurrentY := FMargin;
end;

procedure TPDFReport.AddSpacer(AHeight: Double);
begin
  FCurrentY := FCurrentY + AHeight * 72/25.4;
end;

procedure TPDFReport.AddHorizontalLine;
begin
  FPDF.DrawLine(FMargin, FCurrentY, FPDF.PageWidth - FMargin, FCurrentY,
    0.5, PDFColor(200, 200, 200));
  FCurrentY := FCurrentY + 3 * 72/25.4;
end;

procedure TPDFReport.AddKeyValue(const AKey, AValue: string);
begin
  FPDF.DrawText(AKey + ':', FMargin, FCurrentY, 9, False, PDFColor(127, 140, 141));
  FPDF.DrawText(AValue, FMargin + 35 * 72/25.4, FCurrentY, 9, True, PDFColor(0, 0, 0));
  FCurrentY := FCurrentY + 6 * 72/25.4;
end;

end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **PDF Fundamentals** - โครงสร้าง PDF file format
2. **TPDFWriter** - เขียน PDF โดยตรงด้วย PDF operators
3. **PDF Invoice** - สร้างใบแจ้งหนี้ PDF ที่สวยงาม
4. **PDF Report** - Report Builder สำหรับสร้างรายงาน
5. **Watermark** - เพิ่ม watermark ลงใน PDF

PDF เป็นรูปแบบที่ดีสำหรับเอกสารที่ต้องการความสวยงามและพกพา FPReport ที่มากับ Lazarus เป็นตัวเลือกที่ดีสำหรับรายงานซับซ้อน
