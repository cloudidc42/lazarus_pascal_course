# ตอนที่ 34: Report Generation (การสร้างรายงาน)

## บทนำ

การสร้างรายงานเป็นส่วนสำคัญของแอปพลิเคชันธุรกิจทุกประเภท ใน Lazarus มีเครื่องมือหลายอย่างสำหรับสร้างรายงาน ตั้งแต่ LazReport (มาพร้อม Lazarus), FastReport, และ FPReport ในบทนี้เราจะเรียนรู้การสร้างรายงานแบบต่างๆ

## แนวคิดการออกแบบรายงาน

### โครงสร้างรายงาน
```
┌─────────────────────────────────────────┐
│           Title Band (แสดงครั้งเดียว)    │
│           ชื่อรายงาน, โลโก้บริษัท        │
├─────────────────────────────────────────┤
│      Page Header Band (ทุกหน้า)          │
│      หัวตาราง, วันที่พิมพ์              │
├─────────────────────────────────────────┤
│    Group Header Band (แสดงตาม group)    │
│    ชื่อกลุ่ม เช่น หมวดหมู่สินค้า       │
├─────────────────────────────────────────┤
│        Detail Band (ทุก record)          │
│        ข้อมูลแต่ละรายการ                │
├─────────────────────────────────────────┤
│    Group Footer Band (สรุปแต่ละกลุ่ม)  │
│    ยอดรวมของกลุ่ม                        │
├─────────────────────────────────────────┤
│      Page Footer Band (ทุกหน้า)          │
│      เลขหน้า, copyright                 │
├─────────────────────────────────────────┤
│      Summary Band (แสดงครั้งเดียว)       │
│      ยอดรวมทั้งหมด                       │
└─────────────────────────────────────────┘
```

---

## LazReport

### การติดตั้ง LazReport

LazReport มาพร้อมกับ Lazarus แต่ต้องติดตั้งเพิ่ม:

1. Package > Open Package File (.lpk)
2. เปิดไฟล์ `lazreport.lpk` ใน `<lazarus>/components/lazreport/`
3. Compile and Install
4. รีสตาร์ท Lazarus

**units ที่ต้องใช้:**
```pascal
uses
  LR_Class,   // TfrReport
  LR_DBSet,   // TfrDBDataSet
  LR_View;    // Report viewer
```

### การสร้างรายงานด้วย Code

```pascal
unit LazReportExample;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, LR_Class, LR_DBSet, LR_View,
  LR_Shape, LR_ChartReport,
  DB, ZDataSet, ZConnection;

type
  TReportGenerator = class
  private
    FReport: TfrReport;
    FDataSet: TfrDBDataSet;
    FQuery: TZQuery;
    FConnection: TZConnection;
    
    procedure CreateReportBands;
    procedure AddReportObjects;
    
  public
    constructor Create(AConnection: TZConnection);
    destructor Destroy; override;
    
    procedure GenerateProductReport;
    procedure GenerateInvoiceReport(OrderID: Integer);
    procedure GenerateSummaryReport(StartDate, EndDate: TDateTime);
    
    procedure Preview;
    procedure PrintReport;
    procedure ExportToPDF(const FileName: string);
  end;

implementation

constructor TReportGenerator.Create(AConnection: TZConnection);
begin
  FConnection := AConnection;
  
  FReport := TfrReport.Create(nil);
  FDataSet := TfrDBDataSet.Create(nil);
  FQuery := TZQuery.Create(nil);
  FQuery.Connection := FConnection;
  FDataSet.DataSet := FQuery;
end;

destructor TReportGenerator.Destroy;
begin
  FReport.Free;
  FDataSet.Free;
  FQuery.Free;
  inherited Destroy;
end;

procedure TReportGenerator.GenerateProductReport;
begin
  FReport.Clear;
  
  // โหลดข้อมูล
  FQuery.SQL.Text := 
    'SELECT p.product_id, p.sku, p.name, ' +
    '       p.price, p.stock_qty, ' +
    '       c.name AS category_name, ' +
    '       p.price * p.stock_qty AS stock_value ' +
    'FROM products p ' +
    'LEFT JOIN categories c ON p.category_id = c.category_id ' +
    'WHERE p.is_active = TRUE ' +
    'ORDER BY c.name, p.name';
  FQuery.Open;
  
  FDataSet.DataSet := FQuery;
  FReport.DataSets.Clear;
  FReport.DataSets.Add(FDataSet);
  
  // สร้าง report structure
  CreateReportBands;
  AddReportObjects;
  
  FQuery.Close;
end;

procedure TReportGenerator.Preview;
begin
  FReport.ShowReport;
end;

procedure TReportGenerator.PrintReport;
begin
  FReport.PrintReport;
end;

procedure TReportGenerator.ExportToPDF(const FileName: string);
begin
  // ใช้ PDF export ใน LazReport
  // ต้องติดตั้ง lazreportpdfexport package
  FReport.ExportTo('PDF', FileName);
end;

end.
```

---

## FPReport (Free Pascal Report)

FPReport เป็น report engine ที่มาพร้อมกับ Free Pascal และ Lazarus 1.8+

### การใช้งาน FPReport

```pascal
unit FPReportExample;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  fpreport,          // TFPReport, TFPReportPage
  fpreportDataset,   // TFPReportDataset
  fpreportexporter,  // Base exporter
  fpreporthtmlexport,  // HTML export
  fpreportpdfexport;   // PDF export (ถ้ามี)

type
  TFPReportExample = class
  private
    FReport: TFPReport;
    
    procedure OnGetValue(Sender: TFPReportElement; 
                         var AValue: TNullableBool);
    procedure OnBeforePrint(Sender: TFPReportElement);
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure CreateSimpleReport;
    procedure CreateGroupedReport;
    procedure CreateInvoiceReport;
    procedure ExportToHTML(const FileName: string);
    procedure Preview;
  end;

implementation

constructor TFPReportExample.Create;
begin
  FReport := TFPReport.Create(nil);
end;

destructor TFPReportExample.Destroy;
begin
  FReport.Free;
  inherited Destroy;
end;

procedure TFPReportExample.CreateSimpleReport;
var
  Page: TFPReportPage;
  TitleBand: TFPReportTitleBand;
  DetailBand: TFPReportDetailBand;
  FooterBand: TFPReportPageFooterBand;
  TextElement: TFPReportMemo;
  LineElement: TFPReportShape;
  Variable: TFPReportVariable;
begin
  FReport.Clear;
  FReport.Author := 'ระบบรายงาน';
  FReport.Subject := 'รายงานสินค้า';
  FReport.Title := 'รายการสินค้าทั้งหมด';
  
  // สร้างหน้า
  Page := TFPReportPage.Create(FReport.Pages);
  Page.Orientation := poPortrait;
  Page.PageSize.PaperName := 'A4';
  Page.Margins.Left := 15;
  Page.Margins.Right := 15;
  Page.Margins.Top := 15;
  Page.Margins.Bottom := 15;
  
  // ─── Title Band ───
  TitleBand := TFPReportTitleBand.Create(Page);
  TitleBand.Layout.Height := 20;
  
  // ชื่อรายงาน
  TextElement := TFPReportMemo.Create(TitleBand);
  TextElement.Layout.Left := 0;
  TextElement.Layout.Top := 2;
  TextElement.Layout.Width := Page.Layout.Width;
  TextElement.Layout.Height := 12;
  TextElement.Text := 'รายการสินค้าทั้งหมด';
  TextElement.Font.Size := 16;
  TextElement.Font.Style := [fsBold];
  TextElement.TextAlignment.Horizontal := taCenter;
  
  // วันที่พิมพ์
  TextElement := TFPReportMemo.Create(TitleBand);
  TextElement.Layout.Left := 0;
  TextElement.Layout.Top := 14;
  TextElement.Layout.Width := Page.Layout.Width;
  TextElement.Layout.Height := 6;
  TextElement.Text := 'วันที่พิมพ์: [FORMATDATETIME(''dd/mm/yyyy'', TODAY)]';
  TextElement.Font.Size := 9;
  TextElement.TextAlignment.Horizontal := taRight;
  
  // ─── Column Headers ───
  var ColHeaderBand := TFPReportColumnHeaderBand.Create(Page);
  ColHeaderBand.Layout.Height := 8;
  ColHeaderBand.Frame.Shape := [fsTop, fsBottom];
  ColHeaderBand.Frame.Width := 0.5;
  
  // Headers
  var HeaderData: array[0..4] of record
    Left, Width: Double;
    Caption: string;
  end;
  
  HeaderData[0] := (Left: 0;   Width: 20;  Caption: 'รหัส');
  HeaderData[1] := (Left: 22;  Width: 80;  Caption: 'ชื่อสินค้า');
  HeaderData[2] := (Left: 104; Width: 40;  Caption: 'หมวดหมู่');
  HeaderData[3] := (Left: 146; Width: 25;  Caption: 'ราคา');
  HeaderData[4] := (Left: 173; Width: 25;  Caption: 'คงเหลือ');
  
  for var i := 0 to 4 do
  begin
    TextElement := TFPReportMemo.Create(ColHeaderBand);
    TextElement.Layout.Left := HeaderData[i].Left;
    TextElement.Layout.Top := 1;
    TextElement.Layout.Width := HeaderData[i].Width;
    TextElement.Layout.Height := 6;
    TextElement.Text := HeaderData[i].Caption;
    TextElement.Font.Style := [fsBold];
    TextElement.Font.Size := 9;
  end;
  
  // ─── Detail Band ───
  DetailBand := TFPReportDetailBand.Create(Page);
  DetailBand.Layout.Height := 7;
  DetailBand.DataSet := nil;  // กำหนด dataset ที่นี่
  
  // Detail fields
  var DetailFields: array[0..4] of record
    Left, Width: Double;
    FieldExpr: string;
    Alignment: TAlignment;
  end;
  
  DetailFields[0] := (Left: 0;   Width: 20;  FieldExpr: '[product_id]'; Alignment: taCenter);
  DetailFields[1] := (Left: 22;  Width: 80;  FieldExpr: '[name]'; Alignment: taLeftJustify);
  DetailFields[2] := (Left: 104; Width: 40;  FieldExpr: '[category_name]'; Alignment: taLeftJustify);
  DetailFields[3] := (Left: 146; Width: 25;  FieldExpr: '[price]'; Alignment: taRightJustify);
  DetailFields[4] := (Left: 173; Width: 25;  FieldExpr: '[stock_qty]'; Alignment: taRightJustify);
  
  for var i := 0 to 4 do
  begin
    TextElement := TFPReportMemo.Create(DetailBand);
    TextElement.Layout.Left := DetailFields[i].Left;
    TextElement.Layout.Top := 1;
    TextElement.Layout.Width := DetailFields[i].Width;
    TextElement.Layout.Height := 5;
    TextElement.Text := DetailFields[i].FieldExpr;
    TextElement.Font.Size := 9;
    TextElement.TextAlignment.Horizontal := DetailFields[i].Alignment;
  end;
  
  // ─── Page Footer ───
  FooterBand := TFPReportPageFooterBand.Create(Page);
  FooterBand.Layout.Height := 8;
  FooterBand.Frame.Shape := [fsTop];
  
  TextElement := TFPReportMemo.Create(FooterBand);
  TextElement.Layout.Left := 0;
  TextElement.Layout.Top := 2;
  TextElement.Layout.Width := 80;
  TextElement.Layout.Height := 5;
  TextElement.Text := 'บริษัท ตัวอย่าง จำกัด';
  TextElement.Font.Size := 8;
  
  TextElement := TFPReportMemo.Create(FooterBand);
  TextElement.Layout.Left := Page.Layout.Width - 40;
  TextElement.Layout.Top := 2;
  TextElement.Layout.Width := 40;
  TextElement.Layout.Height := 5;
  TextElement.Text := 'หน้า [PAGENO] จาก [PAGECOUNT]';
  TextElement.Font.Size := 8;
  TextElement.TextAlignment.Horizontal := taRightJustify;
end;

procedure TFPReportExample.ExportToHTML(const FileName: string);
var
  Exporter: TFPReportHTMLExporter;
begin
  Exporter := TFPReportHTMLExporter.Create(nil);
  try
    Exporter.Report := FReport;
    Exporter.FileName := FileName;
    FReport.RunReport;
    Exporter.Execute;
    WriteLn('ส่งออก HTML แล้ว: ' + FileName);
  finally
    Exporter.Free;
  end;
end;

end.
```

---

## FastReport สำหรับ Lazarus

FastReport เป็น commercial report tool ที่มี Free Lazarus edition

### การใช้งาน FastReport

```pascal
unit FastReportExample;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, frxClass, frxDBSet, frxExportPDF, frxExportXLS;

type
  TFastReportHelper = class
  private
    FReport: TfrxReport;
    FDBDataset: TfrxDBDataSet;
    
    procedure SetupReport;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure LoadFromFile(const ReportFile: string);
    procedure PrepareData(AQuery: TZQuery);
    procedure Preview;
    procedure Print;
    procedure ExportToPDF(const FileName: string);
    procedure ExportToExcel(const FileName: string);
    procedure SetVariable(const Name: string; Value: Variant);
  end;

implementation

constructor TFastReportHelper.Create;
begin
  FReport := TfrxReport.Create(nil);
  FDBDataset := TfrxDBDataSet.Create(nil);
  
  // ซ่อน designer button ใน preview
  FReport.PreviewOptions.AllowDesign := False;
  FReport.PreviewOptions.Buttons := FReport.PreviewOptions.Buttons - [pbDesign];
  
  // ตั้งค่า PDF
  FReport.ExportOptions.ExportFileName := 'output.pdf';
  FReport.ExportOptions.OpenAfterExport := False;
end;

destructor TFastReportHelper.Destroy;
begin
  FReport.Free;
  FDBDataset.Free;
  inherited Destroy;
end;

procedure TFastReportHelper.LoadFromFile(const ReportFile: string);
begin
  if FileExists(ReportFile) then
    FReport.LoadFromFile(ReportFile)
  else
    raise Exception.Create('ไม่พบไฟล์รายงาน: ' + ReportFile);
end;

procedure TFastReportHelper.PrepareData(AQuery: TZQuery);
begin
  FDBDataset.DataSet := AQuery;
  FReport.Clear;
  FReport.DataSets.Add(FDBDataset);
end;

procedure TFastReportHelper.Preview;
begin
  FReport.ShowReport;
end;

procedure TFastReportHelper.Print;
begin
  FReport.PrintReport;
end;

procedure TFastReportHelper.ExportToPDF(const FileName: string);
var
  PDFExport: TfrxPDFExport;
begin
  PDFExport := TfrxPDFExport.Create(nil);
  try
    PDFExport.FileName := FileName;
    PDFExport.ShowDialog := False;
    PDFExport.Subject := FReport.ReportTitle;
    PDFExport.Creator := 'Hospital System';
    FReport.Export(PDFExport);
    WriteLn('PDF บันทึกแล้ว: ' + FileName);
  finally
    PDFExport.Free;
  end;
end;

procedure TFastReportHelper.ExportToExcel(const FileName: string);
var
  XLSExport: TfrxXLSExport;
begin
  XLSExport := TfrxXLSExport.Create(nil);
  try
    XLSExport.FileName := FileName;
    XLSExport.ShowDialog := False;
    FReport.Export(XLSExport);
    WriteLn('Excel บันทึกแล้ว: ' + FileName);
  finally
    XLSExport.Free;
  end;
end;

procedure TFastReportHelper.SetVariable(const Name: string; Value: Variant);
begin
  FReport.Variables[Name] := Value;
end;

end.
```

---

## สร้างรายงาน Invoice (ใบแจ้งหนี้) ด้วย Code

```pascal
unit InvoiceReport;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Printers, 
  ZConnection, ZDataSet;

type
  TInvoiceData = record
    InvoiceNumber: string;
    InvoiceDate: TDateTime;
    CustomerName: string;
    CustomerAddress: string;
    CustomerPhone: string;
    TaxID: string;
  end;
  
  TInvoiceItem = record
    ItemNo: Integer;
    ProductName: string;
    Quantity: Integer;
    UnitPrice: Double;
    Total: Double;
  end;
  TInvoiceItems = array of TInvoiceItem;
  
  TInvoiceReportPrinter = class
  private
    FPrinter: TPrinter;
    FCanvas: TCanvas;
    FCurrentY: Integer;
    FPageWidth, FPageHeight: Integer;
    FMarginLeft, FMarginRight, FMarginTop, FMarginBottom: Integer;
    
    procedure NewPage;
    procedure DrawHeader(const Invoice: TInvoiceData);
    procedure DrawItemTable(const Items: TInvoiceItems);
    procedure DrawTableHeader;
    procedure DrawItem(const Item: TInvoiceItem);
    procedure DrawSummary(Subtotal, Tax, Total: Double);
    procedure DrawFooter(const Invoice: TInvoiceData);
    procedure DrawLine(Y: Integer);
    procedure DrawText(const Text: string; X, Y, Width: Integer; 
                      Alignment: TAlignment = taLeftJustify; 
                      Bold: Boolean = False; FontSize: Integer = 10);
    
  public
    constructor Create;
    procedure PrintInvoice(const Invoice: TInvoiceData; 
                          const Items: TInvoiceItems);
  end;

implementation

constructor TInvoiceReportPrinter.Create;
begin
  FPrinter := Printer;
  FMarginLeft := 20;
  FMarginRight := 20;
  FMarginTop := 15;
  FMarginBottom := 20;
end;

procedure TInvoiceReportPrinter.DrawText(const Text: string; 
                                         X, Y, Width: Integer;
                                         Alignment: TAlignment = taLeftJustify;
                                         Bold: Boolean = False; 
                                         FontSize: Integer = 10);
var
  TextX: Integer;
  TextWidth: Integer;
begin
  FCanvas.Font.Size := FontSize;
  FCanvas.Font.Bold := Bold;
  
  TextWidth := FCanvas.TextWidth(Text);
  
  case Alignment of
    taCenter: 
      TextX := X + (Width - TextWidth) div 2;
    taRightJustify: 
      TextX := X + Width - TextWidth - 2;
    else 
      TextX := X + 2;
  end;
  
  FCanvas.TextOut(TextX, Y, Text);
end;

procedure TInvoiceReportPrinter.DrawLine(Y: Integer);
begin
  FCanvas.Pen.Color := clBlack;
  FCanvas.Pen.Width := 1;
  FCanvas.MoveTo(FMarginLeft * FPrinter.XDPI div 25, Y);
  FCanvas.LineTo((FPageWidth div FPrinter.XDPI * 25 - FMarginRight) * FPrinter.XDPI div 25, Y);
end;

procedure TInvoiceReportPrinter.DrawHeader(const Invoice: TInvoiceData);
var
  Y: Integer;
begin
  Y := FMarginTop * FPrinter.YDPI div 25;
  
  // โลโก้บริษัท (สมมติ)
  FCanvas.Font.Size := 18;
  FCanvas.Font.Bold := True;
  FCanvas.Font.Color := clNavy;
  DrawText('บริษัท ตัวอย่าง จำกัด', 
           FMarginLeft * FPrinter.XDPI div 25, Y,
           (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25,
           taCenter, True, 18);
  
  FCanvas.Font.Color := clBlack;
  Y := Y + FPrinter.YDPI div 25 * 7;
  
  DrawText('123 ถ.สุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพฯ 10110',
           FMarginLeft * FPrinter.XDPI div 25, Y,
           (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25,
           taCenter, False, 9);
  
  Y := Y + FPrinter.YDPI div 25 * 5;
  DrawText('โทร: 02-123-4567  แฟกซ์: 02-123-4568  อีเมล: info@example.com',
           FMarginLeft * FPrinter.XDPI div 25, Y,
           (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25,
           taCenter, False, 9);
  
  Y := Y + FPrinter.YDPI div 25 * 5;
  DrawLine(Y);
  
  // ชื่อเอกสาร
  Y := Y + FPrinter.YDPI div 25 * 5;
  DrawText('ใบแจ้งหนี้ / INVOICE',
           FMarginLeft * FPrinter.XDPI div 25, Y,
           (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25,
           taCenter, True, 16);
  
  Y := Y + FPrinter.YDPI div 25 * 10;
  DrawLine(Y);
  Y := Y + FPrinter.YDPI div 25 * 3;
  
  // ข้อมูลลูกค้า
  var HalfWidth := (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 50;
  var XLeft := FMarginLeft * FPrinter.XDPI div 25;
  var XRight := XLeft + HalfWidth;
  
  DrawText('ชื่อลูกค้า: ' + Invoice.CustomerName, XLeft, Y, HalfWidth, taLeftJustify, False, 10);
  DrawText('เลขที่ใบแจ้งหนี้: ' + Invoice.InvoiceNumber, XRight, Y, HalfWidth, taRightJustify, True, 10);
  
  Y := Y + FPrinter.YDPI div 25 * 6;
  DrawText('ที่อยู่: ' + Invoice.CustomerAddress, XLeft, Y, HalfWidth, taLeftJustify, False, 9);
  DrawText('วันที่: ' + FormatDateTime('dd/mm/yyyy', Invoice.InvoiceDate), XRight, Y, HalfWidth, taRightJustify, False, 10);
  
  Y := Y + FPrinter.YDPI div 25 * 6;
  DrawText('โทร: ' + Invoice.CustomerPhone, XLeft, Y, HalfWidth, taLeftJustify, False, 9);
  DrawText('เลขประจำตัวผู้เสียภาษี: ' + Invoice.TaxID, XLeft, Y + FPrinter.YDPI div 25 * 5, HalfWidth, taLeftJustify, False, 9);
  
  FCurrentY := Y + FPrinter.YDPI div 25 * 15;
end;

procedure TInvoiceReportPrinter.DrawTableHeader;
var
  Y: Integer;
  ColWidths: array[0..4] of Integer;
  XLeft: Integer;
begin
  Y := FCurrentY;
  XLeft := FMarginLeft * FPrinter.XDPI div 25;
  var TotalWidth := (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25;
  
  // Column widths (เป็น %)
  ColWidths[0] := TotalWidth * 5 div 100;   // ลำดับ
  ColWidths[1] := TotalWidth * 45 div 100;  // รายการ
  ColWidths[2] := TotalWidth * 15 div 100;  // จำนวน
  ColWidths[3] := TotalWidth * 17 div 100;  // ราคาต่อหน่วย
  ColWidths[4] := TotalWidth * 18 div 100;  // จำนวนเงิน
  
  // เส้นบน
  DrawLine(Y);
  Y := Y + FPrinter.YDPI div 25 * 2;
  
  FCanvas.Brush.Color := $DDEEFF;
  FCanvas.FillRect(Rect(XLeft, Y, XLeft + TotalWidth, Y + FPrinter.YDPI div 25 * 7));
  FCanvas.Brush.Color := clWhite;
  
  var X := XLeft;
  DrawText('ลำดับ', X, Y + FPrinter.YDPI div 25 * 1, ColWidths[0], taCenter, True, 9); X := X + ColWidths[0];
  DrawText('รายการสินค้า/บริการ', X, Y + FPrinter.YDPI div 25 * 1, ColWidths[1], taCenter, True, 9); X := X + ColWidths[1];
  DrawText('จำนวน', X, Y + FPrinter.YDPI div 25 * 1, ColWidths[2], taCenter, True, 9); X := X + ColWidths[2];
  DrawText('ราคา/หน่วย', X, Y + FPrinter.YDPI div 25 * 1, ColWidths[3], taCenter, True, 9); X := X + ColWidths[3];
  DrawText('จำนวนเงิน', X, Y + FPrinter.YDPI div 25 * 1, ColWidths[4], taCenter, True, 9);
  
  Y := Y + FPrinter.YDPI div 25 * 8;
  DrawLine(Y);
  
  FCurrentY := Y + FPrinter.YDPI div 25 * 2;
end;

procedure TInvoiceReportPrinter.DrawItem(const Item: TInvoiceItem);
var
  Y: Integer;
  XLeft: Integer;
  TotalWidth: Integer;
  ColWidths: array[0..4] of Integer;
begin
  Y := FCurrentY;
  XLeft := FMarginLeft * FPrinter.XDPI div 25;
  TotalWidth := (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25;
  
  ColWidths[0] := TotalWidth * 5 div 100;
  ColWidths[1] := TotalWidth * 45 div 100;
  ColWidths[2] := TotalWidth * 15 div 100;
  ColWidths[3] := TotalWidth * 17 div 100;
  ColWidths[4] := TotalWidth * 18 div 100;
  
  var X := XLeft;
  DrawText(IntToStr(Item.ItemNo), X, Y, ColWidths[0], taCenter, False, 9); X := X + ColWidths[0];
  DrawText(Item.ProductName, X, Y, ColWidths[1], taLeftJustify, False, 9); X := X + ColWidths[1];
  DrawText(IntToStr(Item.Quantity), X, Y, ColWidths[2], taCenter, False, 9); X := X + ColWidths[2];
  DrawText(FormatFloat('#,##0.00', Item.UnitPrice), X, Y, ColWidths[3], taRightJustify, False, 9); X := X + ColWidths[3];
  DrawText(FormatFloat('#,##0.00', Item.Total), X, Y, ColWidths[4], taRightJustify, False, 9);
  
  FCurrentY := Y + FPrinter.YDPI div 25 * 6;
end;

procedure TInvoiceReportPrinter.DrawSummary(Subtotal, Tax, Total: Double);
var
  Y: Integer;
  XLeft: Integer;
  TotalWidth: Integer;
begin
  Y := FCurrentY + FPrinter.YDPI div 25 * 5;
  XLeft := FMarginLeft * FPrinter.XDPI div 25;
  TotalWidth := (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25;
  
  DrawLine(Y);
  Y := Y + FPrinter.YDPI div 25 * 3;
  
  var SummaryX := XLeft + TotalWidth * 60 div 100;
  var SummaryWidth := TotalWidth * 40 div 100;
  
  DrawText('ยอดรวมก่อน VAT:', SummaryX, Y, SummaryWidth * 60 div 100, taLeftJustify, False, 10);
  DrawText(FormatFloat('#,##0.00', Subtotal), SummaryX + SummaryWidth * 60 div 100, Y, SummaryWidth * 40 div 100, taRightJustify, False, 10);
  
  Y := Y + FPrinter.YDPI div 25 * 6;
  DrawText('ภาษีมูลค่าเพิ่ม 7%:', SummaryX, Y, SummaryWidth * 60 div 100, taLeftJustify, False, 10);
  DrawText(FormatFloat('#,##0.00', Tax), SummaryX + SummaryWidth * 60 div 100, Y, SummaryWidth * 40 div 100, taRightJustify, False, 10);
  
  Y := Y + FPrinter.YDPI div 25 * 4;
  DrawLine(Y);
  
  Y := Y + FPrinter.YDPI div 25 * 3;
  FCanvas.Font.Color := clRed;
  DrawText('จำนวนเงินรวมทั้งสิ้น:', SummaryX, Y, SummaryWidth * 60 div 100, taLeftJustify, True, 12);
  DrawText(FormatFloat('#,##0.00', Total) + ' บาท', SummaryX + SummaryWidth * 60 div 100, Y, SummaryWidth * 40 div 100, taRightJustify, True, 12);
  FCanvas.Font.Color := clBlack;
  
  // เงินเป็นตัวอักษร
  Y := Y + FPrinter.YDPI div 25 * 8;
  DrawText('( ' + NumberToThaiWords(Total) + ' )', 
           XLeft, Y, TotalWidth, taCenter, False, 10);
  
  FCurrentY := Y;
end;

procedure TInvoiceReportPrinter.DrawFooter(const Invoice: TInvoiceData);
var
  Y: Integer;
  XLeft, TotalWidth: Integer;
begin
  Y := FPageHeight - FMarginBottom * FPrinter.YDPI div 25;
  XLeft := FMarginLeft * FPrinter.XDPI div 25;
  TotalWidth := (FPageWidth - FMarginLeft - FMarginRight) * FPrinter.XDPI div 25;
  
  DrawLine(Y - FPrinter.YDPI div 25 * 20);
  
  // ลายเซ็น
  var SigWidth := TotalWidth div 3;
  
  DrawText('________________________', XLeft, Y - FPrinter.YDPI div 25 * 15, SigWidth, taCenter);
  DrawText('ผู้รับเงิน', XLeft, Y - FPrinter.YDPI div 25 * 9, SigWidth, taCenter, False, 9);
  DrawText('วันที่: ___/___/____', XLeft, Y - FPrinter.YDPI div 25 * 4, SigWidth, taCenter, False, 9);
  
  DrawText('________________________', XLeft + SigWidth * 2, Y - FPrinter.YDPI div 25 * 15, SigWidth, taCenter);
  DrawText('ผู้มีอำนาจลงนาม', XLeft + SigWidth * 2, Y - FPrinter.YDPI div 25 * 9, SigWidth, taCenter, False, 9);
  DrawText('ตราบริษัท', XLeft + SigWidth * 2, Y - FPrinter.YDPI div 25 * 4, SigWidth, taCenter, False, 9);
  
  DrawLine(Y);
  
  DrawText('หมายเหตุ: กรุณาชำระภายใน 30 วัน หากมีข้อสงสัยกรุณาติดต่อ 02-123-4567',
           XLeft, Y + FPrinter.YDPI div 25 * 3, TotalWidth, taCenter, False, 8);
end;

function NumberToThaiWords(Amount: Double): string;
// แปลงตัวเลขเป็นคำอ่านภาษาไทย (simplified version)
var
  IntPart: Int64;
  DecPart: Integer;
begin
  IntPart := Trunc(Amount);
  DecPart := Round((Amount - IntPart) * 100);
  
  // ใช้ library หรือเขียนเอง
  // simplified: แสดงแบบง่าย
  if DecPart = 0 then
    Result := Format('%s บาทถ้วน', [IntToStr(IntPart)])
  else
    Result := Format('%s บาท %d สตางค์', [IntToStr(IntPart), DecPart]);
end;

procedure TInvoiceReportPrinter.PrintInvoice(const Invoice: TInvoiceData;
                                             const Items: TInvoiceItems);
var
  Subtotal, Tax, Total: Double;
  i: Integer;
begin
  FPrinter.BeginDoc;
  try
    FCanvas := FPrinter.Canvas;
    FPageWidth := FPrinter.PageWidth;
    FPageHeight := FPrinter.PageHeight;
    
    FCurrentY := 0;
    
    DrawHeader(Invoice);
    DrawTableHeader;
    
    Subtotal := 0;
    for i := 0 to High(Items) do
    begin
      DrawItem(Items[i]);
      Subtotal := Subtotal + Items[i].Total;
    end;
    
    Tax := Subtotal * 0.07;
    Total := Subtotal + Tax;
    
    DrawSummary(Subtotal, Tax, Total);
    DrawFooter(Invoice);
    
  finally
    FPrinter.EndDoc;
  end;
end;

end.
```

---

## การสร้างรายงานใบประมวลผลนักศึกษา (Student Transcript)

```pascal
unit StudentTranscript;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Printers, ZDataSet, ZConnection;

type
  TSubjectGrade = record
    SubjectCode: string;
    SubjectName: string;
    Credits: Integer;
    Grade: string;
    GradePoint: Double;
    Semester: Integer;
    AcademicYear: Integer;
  end;
  TSubjectGrades = array of TSubjectGrade;

  TStudentInfo = record
    StudentID: string;
    FullName: string;
    Faculty: string;
    Department: string;
    Program: string;
    EntryYear: Integer;
    AdmissionType: string;
  end;

  TTranscriptPrinter = class
  private
    FConn: TZConnection;
    FCanvas: TCanvas;
    FCurrentY: Integer;
    
    procedure DrawTranscriptHeader(const Student: TStudentInfo);
    procedure DrawSemesterBlock(const Grades: TSubjectGrades; 
                                Year, Semester: Integer);
    procedure DrawSummary(const Grades: TSubjectGrades);
    procedure DrawGPAChart(const Grades: TSubjectGrades);
    
    function CalculateSemesterGPA(const Grades: TSubjectGrades; 
                                  Year, Semester: Integer): Double;
    function CalculateCumulativeGPA(const Grades: TSubjectGrades): Double;
    function GradeToGPA(const Grade: string): Double;
    function GetHonorLevel(GPA: Double): string;
    
  public
    constructor Create(AConn: TZConnection);
    procedure PrintTranscript(const StudentID: string);
    procedure ExportTranscriptToPDF(const StudentID, OutputFile: string);
  end;

implementation

constructor TTranscriptPrinter.Create(AConn: TZConnection);
begin
  FConn := AConn;
end;

function TTranscriptPrinter.GradeToGPA(const Grade: string): Double;
begin
  if Grade = 'A'  then Result := 4.0
  else if Grade = 'B+' then Result := 3.5
  else if Grade = 'B'  then Result := 3.0
  else if Grade = 'C+' then Result := 2.5
  else if Grade = 'C'  then Result := 2.0
  else if Grade = 'D+' then Result := 1.5
  else if Grade = 'D'  then Result := 1.0
  else Result := 0.0;  // F, W, I
end;

function TTranscriptPrinter.CalculateSemesterGPA(const Grades: TSubjectGrades;
                                                  Year, Semester: Integer): Double;
var
  TotalPoints, TotalCredits: Double;
begin
  TotalPoints := 0;
  TotalCredits := 0;
  
  for var G in Grades do
  begin
    if (G.AcademicYear = Year) and (G.Semester = Semester) then
    begin
      if G.Grade <> 'W' then  // ไม่นับวิชาที่ถอน
      begin
        TotalPoints := TotalPoints + GradeToGPA(G.Grade) * G.Credits;
        TotalCredits := TotalCredits + G.Credits;
      end;
    end;
  end;
  
  if TotalCredits > 0 then
    Result := TotalPoints / TotalCredits
  else
    Result := 0;
end;

function TTranscriptPrinter.CalculateCumulativeGPA(const Grades: TSubjectGrades): Double;
var
  TotalPoints, TotalCredits: Double;
begin
  TotalPoints := 0;
  TotalCredits := 0;
  
  for var G in Grades do
  begin
    if G.Grade <> 'W' then
    begin
      TotalPoints := TotalPoints + GradeToGPA(G.Grade) * G.Credits;
      TotalCredits := TotalCredits + G.Credits;
    end;
  end;
  
  if TotalCredits > 0 then
    Result := TotalPoints / TotalCredits
  else
    Result := 0;
end;

function TTranscriptPrinter.GetHonorLevel(GPA: Double): string;
begin
  if GPA >= 3.60 then Result := 'เกียรตินิยมอันดับหนึ่ง'
  else if GPA >= 3.25 then Result := 'เกียรตินิยมอันดับสอง'
  else Result := '';
end;

procedure TTranscriptPrinter.PrintTranscript(const StudentID: string);
var
  Q: TZQuery;
  Student: TStudentInfo;
  Grades: TSubjectGrades;
  GradeCount: Integer;
begin
  // โหลดข้อมูลนักศึกษา
  Q := TZQuery.Create(nil);
  try
    Q.Connection := FConn;
    Q.SQL.Text := 
      'SELECT s.student_id, s.first_name || '' '' || s.last_name AS full_name, ' +
      '       f.faculty_name, d.department_name, p.program_name, ' +
      '       s.entry_year, s.admission_type ' +
      'FROM students s ' +
      'JOIN faculties f ON s.faculty_id = f.faculty_id ' +
      'JOIN departments d ON s.department_id = d.department_id ' +
      'JOIN programs p ON s.program_id = p.program_id ' +
      'WHERE s.student_id = :student_id';
    Q.ParamByName('student_id').AsString := StudentID;
    Q.Open;
    
    if Q.EOF then
    begin
      ShowMessage('ไม่พบนักศึกษารหัส: ' + StudentID);
      Q.Free;
      Exit;
    end;
    
    Student.StudentID := Q.FieldByName('student_id').AsString;
    Student.FullName := Q.FieldByName('full_name').AsString;
    Student.Faculty := Q.FieldByName('faculty_name').AsString;
    Student.Department := Q.FieldByName('department_name').AsString;
    Student.Program := Q.FieldByName('program_name').AsString;
    Student.EntryYear := Q.FieldByName('entry_year').AsInteger;
    Student.AdmissionType := Q.FieldByName('admission_type').AsString;
    
    Q.Close;
    
    // โหลดผลการเรียน
    Q.SQL.Text := 
      'SELECT sg.subject_code, sub.subject_name, sub.credits, ' +
      '       sg.grade, sg.semester, sg.academic_year ' +
      'FROM student_grades sg ' +
      'JOIN subjects sub ON sg.subject_code = sub.subject_code ' +
      'WHERE sg.student_id = :student_id ' +
      'ORDER BY sg.academic_year, sg.semester, sub.subject_code';
    Q.ParamByName('student_id').AsString := StudentID;
    Q.Open;
    
    GradeCount := 0;
    SetLength(Grades, 200);  // Reserve space
    
    while not Q.EOF do
    begin
      Grades[GradeCount].SubjectCode := Q.FieldByName('subject_code').AsString;
      Grades[GradeCount].SubjectName := Q.FieldByName('subject_name').AsString;
      Grades[GradeCount].Credits := Q.FieldByName('credits').AsInteger;
      Grades[GradeCount].Grade := Q.FieldByName('grade').AsString;
      Grades[GradeCount].GradePoint := GradeToGPA(Q.FieldByName('grade').AsString);
      Grades[GradeCount].Semester := Q.FieldByName('semester').AsInteger;
      Grades[GradeCount].AcademicYear := Q.FieldByName('academic_year').AsInteger;
      Inc(GradeCount);
      Q.Next;
    end;
    SetLength(Grades, GradeCount);
    
    Q.Close;
    
  finally
    Q.Free;
  end;
  
  // Print
  Printer.BeginDoc;
  try
    FCanvas := Printer.Canvas;
    FCurrentY := 20;
    
    DrawTranscriptHeader(Student);
    
    // หาปีการศึกษาทั้งหมด
    var Years := TStringList.Create;
    try
      for var G in Grades do
      begin
        var Key := Format('%d-%d', [G.AcademicYear, G.Semester]);
        if Years.IndexOf(Key) < 0 then
          Years.Add(Key);
      end;
      Years.Sort;
      
      for var Key in Years do
      begin
        var Parts := Key.Split(['-']);
        var Year := StrToInt(Parts[0]);
        var Semester := StrToInt(Parts[1]);
        DrawSemesterBlock(Grades, Year, Semester);
      end;
    finally
      Years.Free;
    end;
    
    DrawSummary(Grades);
    
  finally
    Printer.EndDoc;
  end;
end;

procedure TTranscriptPrinter.DrawTranscriptHeader(const Student: TStudentInfo);
begin
  var Canvas := FCanvas;
  
  Canvas.Font.Size := 14;
  Canvas.Font.Bold := True;
  Canvas.TextOut(Printer.PageWidth div 2 - 100, FCurrentY, 
    'ใบรายงานผลการศึกษา (TRANSCRIPT)');
  FCurrentY := FCurrentY + 20;
  
  Canvas.Font.Size := 12;
  Canvas.Font.Bold := False;
  Canvas.TextOut(Printer.PageWidth div 2 - 80, FCurrentY, 
    'มหาวิทยาลัยตัวอย่าง');
  FCurrentY := FCurrentY + 25;
  
  Canvas.Font.Size := 10;
  Canvas.TextOut(50, FCurrentY, 'รหัสนักศึกษา: ' + Student.StudentID);
  Canvas.TextOut(250, FCurrentY, 'ชื่อ-สกุล: ' + Student.FullName);
  FCurrentY := FCurrentY + 15;
  
  Canvas.TextOut(50, FCurrentY, 'คณะ: ' + Student.Faculty);
  Canvas.TextOut(250, FCurrentY, 'สาขา: ' + Student.Department);
  FCurrentY := FCurrentY + 15;
  
  Canvas.TextOut(50, FCurrentY, 'หลักสูตร: ' + Student.Program);
  Canvas.TextOut(250, FCurrentY, 'ปีที่เข้า: ' + IntToStr(Student.EntryYear));
  FCurrentY := FCurrentY + 25;
end;

procedure TTranscriptPrinter.DrawSemesterBlock(const Grades: TSubjectGrades; 
                                               Year, Semester: Integer);
var
  Canvas: TCanvas;
  HeaderY: Integer;
begin
  Canvas := FCanvas;
  HeaderY := FCurrentY;
  
  // Header
  Canvas.Font.Bold := True;
  Canvas.Font.Size := 10;
  Canvas.Brush.Color := $DDDDDD;
  Canvas.FillRect(Rect(40, FCurrentY, Printer.PageWidth - 40, FCurrentY + 16));
  Canvas.Brush.Color := clWhite;
  
  Canvas.TextOut(45, FCurrentY + 2, 
    Format('ภาคเรียนที่ %d ปีการศึกษา %d', [Semester, Year]));
  FCurrentY := FCurrentY + 18;
  
  // Column headers
  Canvas.Font.Bold := True;
  Canvas.Font.Size := 9;
  Canvas.TextOut(40,  FCurrentY, 'รหัสวิชา');
  Canvas.TextOut(100, FCurrentY, 'ชื่อวิชา');
  Canvas.TextOut(320, FCurrentY, 'หน่วยกิต');
  Canvas.TextOut(370, FCurrentY, 'เกรด');
  Canvas.TextOut(410, FCurrentY, 'จุดเกรด');
  FCurrentY := FCurrentY + 14;
  
  // Draw separator
  Canvas.MoveTo(40, FCurrentY);
  Canvas.LineTo(Printer.PageWidth - 40, FCurrentY);
  FCurrentY := FCurrentY + 3;
  
  // Subject rows
  Canvas.Font.Bold := False;
  var SemCredits := 0;
  var SemPoints := 0.0;
  
  for var G in Grades do
  begin
    if (G.AcademicYear = Year) and (G.Semester = Semester) then
    begin
      Canvas.TextOut(40,  FCurrentY, G.SubjectCode);
      Canvas.TextOut(100, FCurrentY, G.SubjectName);
      Canvas.TextOut(330, FCurrentY, IntToStr(G.Credits));
      Canvas.TextOut(375, FCurrentY, G.Grade);
      
      if G.Grade <> 'W' then
      begin
        Canvas.TextOut(410, FCurrentY, FormatFloat('0.00', G.GradePoint));
        Inc(SemCredits, G.Credits);
        SemPoints := SemPoints + G.GradePoint * G.Credits;
      end
      else
        Canvas.TextOut(410, FCurrentY, '-');
      
      FCurrentY := FCurrentY + 12;
    end;
  end;
  
  // Semester summary
  FCurrentY := FCurrentY + 3;
  Canvas.MoveTo(40, FCurrentY);
  Canvas.LineTo(Printer.PageWidth - 40, FCurrentY);
  FCurrentY := FCurrentY + 3;
  
  Canvas.Font.Bold := True;
  var SemGPA := 0.0;
  if SemCredits > 0 then SemGPA := SemPoints / SemCredits;
  
  Canvas.TextOut(300, FCurrentY, Format('รวม: %d หน่วยกิต  GPA: %.2f', 
    [SemCredits, SemGPA]));
  FCurrentY := FCurrentY + 20;
end;

procedure TTranscriptPrinter.DrawSummary(const Grades: TSubjectGrades);
var
  Canvas: TCanvas;
  TotalCredits: Integer;
  CumGPA: Double;
begin
  Canvas := FCanvas;
  
  TotalCredits := 0;
  for var G in Grades do
    if G.Grade <> 'W' then
      TotalCredits := TotalCredits + G.Credits;
  
  CumGPA := CalculateCumulativeGPA(Grades);
  
  FCurrentY := FCurrentY + 10;
  
  Canvas.Font.Bold := True;
  Canvas.Font.Size := 11;
  Canvas.Brush.Color := $AABBFF;
  Canvas.FillRect(Rect(40, FCurrentY, Printer.PageWidth - 40, FCurrentY + 30));
  Canvas.Brush.Color := clWhite;
  
  Canvas.TextOut(45, FCurrentY + 5, 
    Format('สรุปผลการเรียน: รวม %d หน่วยกิต  GPAX: %.2f  (%s)',
      [TotalCredits, CumGPA, GetHonorLevel(CumGPA)]));
  
  FCurrentY := FCurrentY + 40;
  
  // Seal
  Canvas.Font.Size := 9;
  Canvas.Font.Bold := False;
  Canvas.TextOut(Printer.PageWidth - 200, FCurrentY, 
    'ออกให้ ณ วันที่: ' + FormatDateTime('dd/mm/yyyy', Now));
end;

end.
```

---

## Grouping Data ในรายงาน

```pascal
unit ReportGrouping;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Printers, ZDataSet, ZConnection;

type
  TGroupedReportPrinter = class
  private
    FConn: TZConnection;
    FCanvas: TCanvas;
    FCurrentY: Integer;
    FPageNo: Integer;
    
    FCurrentGroup: string;
    FGroupSubtotal: Double;
    FGroupCount: Integer;
    FGrandTotal: Double;
    FGrandCount: Integer;
    
    procedure StartPage;
    procedure EndPage;
    procedure NewPage;
    procedure PrintGroupHeader(const GroupName: string);
    procedure PrintGroupFooter;
    procedure PrintSummaryFooter;
    procedure PrintDataRow(const SKU, Name: string; Qty: Integer; 
                          Price, Total: Double);
    
  public
    constructor Create(AConn: TZConnection);
    procedure PrintSalesByCategory(Month, Year: Integer);
  end;

implementation

constructor TGroupedReportPrinter.Create(AConn: TZConnection);
begin
  FConn := AConn;
end;

procedure TGroupedReportPrinter.StartPage;
begin
  FCurrentY := 20;
  Inc(FPageNo);
  
  // Page header
  FCanvas.Font.Size := 8;
  FCanvas.Font.Bold := False;
  FCanvas.TextOut(Printer.PageWidth - 100, 5, 
    Format('หน้า %d  วันที่พิมพ์: %s', 
      [FPageNo, FormatDateTime('dd/mm/yyyy', Now)]));
end;

procedure TGroupedReportPrinter.EndPage;
begin
  // Page footer
  FCanvas.Font.Size := 8;
  FCanvas.MoveTo(40, Printer.PageHeight - 30);
  FCanvas.LineTo(Printer.PageWidth - 40, Printer.PageHeight - 30);
  
  FCanvas.TextOut(40, Printer.PageHeight - 25, 
    'รายงานยอดขายตามหมวดหมู่');
  FCanvas.TextOut(Printer.PageWidth - 120, Printer.PageHeight - 25, 
    Format('หน้า %d', [FPageNo]));
end;

procedure TGroupedReportPrinter.NewPage;
begin
  EndPage;
  Printer.NewPage;
  StartPage;
end;

procedure TGroupedReportPrinter.PrintGroupHeader(const GroupName: string);
begin
  FCurrentY := FCurrentY + 10;
  
  FCanvas.Font.Bold := True;
  FCanvas.Font.Size := 11;
  FCanvas.Brush.Color := $CCDDFF;
  FCanvas.FillRect(Rect(40, FCurrentY, Printer.PageWidth - 40, FCurrentY + 16));
  FCanvas.Brush.Color := clWhite;
  
  FCanvas.TextOut(45, FCurrentY + 2, 'หมวดหมู่: ' + GroupName);
  FCurrentY := FCurrentY + 18;
  
  // Column headers
  FCanvas.Font.Size := 9;
  FCanvas.TextOut(40, FCurrentY, 'SKU');
  FCanvas.TextOut(100, FCurrentY, 'ชื่อสินค้า');
  FCanvas.TextOut(280, FCurrentY, 'จำนวน');
  FCanvas.TextOut(320, FCurrentY, 'ราคา/หน่วย');
  FCanvas.TextOut(380, FCurrentY, 'รวม');
  FCurrentY := FCurrentY + 12;
  
  FCanvas.MoveTo(40, FCurrentY);
  FCanvas.LineTo(Printer.PageWidth - 40, FCurrentY);
  FCurrentY := FCurrentY + 3;
  
  FCurrentGroup := GroupName;
  FGroupSubtotal := 0;
  FGroupCount := 0;
end;

procedure TGroupedReportPrinter.PrintGroupFooter;
begin
  FCanvas.MoveTo(40, FCurrentY);
  FCanvas.LineTo(Printer.PageWidth - 40, FCurrentY);
  FCurrentY := FCurrentY + 3;
  
  FCanvas.Font.Bold := True;
  FCanvas.Font.Size := 9;
  FCanvas.TextOut(250, FCurrentY, Format('รวมหมวด "%s": %d รายการ  ยอดรวม: %s บาท',
    [FCurrentGroup, FGroupCount, FormatFloat('#,##0.00', FGroupSubtotal)]));
  
  FGrandTotal := FGrandTotal + FGroupSubtotal;
  FGrandCount := FGrandCount + FGroupCount;
  
  FCurrentY := FCurrentY + 15;
end;

procedure TGroupedReportPrinter.PrintSummaryFooter;
begin
  FCurrentY := FCurrentY + 10;
  
  FCanvas.Brush.Color := $AAFFAA;
  FCanvas.FillRect(Rect(40, FCurrentY, Printer.PageWidth - 40, FCurrentY + 25));
  FCanvas.Brush.Color := clWhite;
  
  FCanvas.Font.Bold := True;
  FCanvas.Font.Size := 12;
  FCanvas.TextOut(45, FCurrentY + 5, Format('ยอดรวมทั้งหมด: %d รายการ  รวมเงิน: %s บาท',
    [FGrandCount, FormatFloat('#,##0.00', FGrandTotal)]));
  
  FCurrentY := FCurrentY + 30;
end;

procedure TGroupedReportPrinter.PrintDataRow(const SKU, Name: string; 
                                             Qty: Integer; Price, Total: Double);
begin
  // ตรวจสอบว่าต้องขึ้นหน้าใหม่
  if FCurrentY > Printer.PageHeight - 80 then
  begin
    PrintGroupFooter;
    NewPage;
    PrintGroupHeader(FCurrentGroup);
  end;
  
  FCanvas.Font.Bold := False;
  FCanvas.Font.Size := 9;
  
  FCanvas.TextOut(40, FCurrentY, SKU);
  FCanvas.TextOut(100, FCurrentY, Name);
  FCanvas.TextOut(285, FCurrentY, IntToStr(Qty));
  FCanvas.TextOut(330, FCurrentY, FormatFloat('#,##0.00', Price));
  FCanvas.TextOut(370, FCurrentY, FormatFloat('#,##0.00', Total));
  
  FCurrentY := FCurrentY + 12;
  FGroupSubtotal := FGroupSubtotal + Total;
  Inc(FGroupCount);
end;

procedure TGroupedReportPrinter.PrintSalesByCategory(Month, Year: Integer);
var
  Q: TZQuery;
  LastCategory: string;
begin
  Q := TZQuery.Create(nil);
  try
    Q.Connection := FConn;
    Q.SQL.Text := 
      'SELECT c.name AS category_name, ' +
      '       p.sku, p.name AS product_name, ' +
      '       SUM(oi.quantity) AS total_qty, ' +
      '       AVG(oi.unit_price) AS avg_price, ' +
      '       SUM(oi.total_price) AS total_sales ' +
      'FROM order_items oi ' +
      'JOIN orders o ON oi.order_id = o.order_id ' +
      'JOIN products p ON oi.product_id = p.product_id ' +
      'JOIN categories c ON p.category_id = c.category_id ' +
      'WHERE EXTRACT(MONTH FROM o.ordered_at) = :month ' +
      '  AND EXTRACT(YEAR FROM o.ordered_at) = :year ' +
      '  AND o.status NOT IN (''cancelled'', ''refunded'') ' +
      'GROUP BY c.name, p.sku, p.name ' +
      'ORDER BY c.name, total_sales DESC';
    
    Q.ParamByName('month').AsInteger := Month;
    Q.ParamByName('year').AsInteger := Year;
    Q.Open;
    
    if Q.EOF then
    begin
      ShowMessage('ไม่มีข้อมูลในช่วงเวลาที่เลือก');
      Exit;
    end;
    
    FPageNo := 0;
    FGrandTotal := 0;
    FGrandCount := 0;
    LastCategory := '';
    
    Printer.BeginDoc;
    try
      FCanvas := Printer.Canvas;
      StartPage;
      
      // Title
      FCanvas.Font.Bold := True;
      FCanvas.Font.Size := 14;
      FCanvas.TextOut(Printer.PageWidth div 2 - 150, FCurrentY, 
        Format('รายงานยอดขายตามหมวดหมู่ ประจำเดือน %s %d',
          [FormatDateTime('mmmm', EncodeDate(Year, Month, 1)), Year]));
      FCurrentY := FCurrentY + 25;
      
      while not Q.EOF do
      begin
        var CategoryName := Q.FieldByName('category_name').AsString;
        
        if CategoryName <> LastCategory then
        begin
          if LastCategory <> '' then
            PrintGroupFooter;
          
          PrintGroupHeader(CategoryName);
          LastCategory := CategoryName;
        end;
        
        PrintDataRow(
          Q.FieldByName('sku').AsString,
          Q.FieldByName('product_name').AsString,
          Q.FieldByName('total_qty').AsInteger,
          Q.FieldByName('avg_price').AsFloat,
          Q.FieldByName('total_sales').AsFloat
        );
        
        Q.Next;
      end;
      
      // Print last group
      if LastCategory <> '' then
        PrintGroupFooter;
      
      PrintSummaryFooter;
      EndPage;
      
    finally
      Printer.EndDoc;
    end;
    
    Q.Close;
  finally
    Q.Free;
  end;
end;

end.
```

---

## แบบฝึกหัด 10 ข้อ

**ข้อ 1:** สร้างรายงานรายชื่อพนักงาน
```pascal
// ข้อมูล: รหัส, ชื่อ, แผนก, ตำแหน่ง, เงินเดือน
// จัดกลุ่มตามแผนก
// แสดงค่าเฉลี่ยเงินเดือนแต่ละแผนก
// ยอดรวมเงินเดือนทั้งหมด
```

**ข้อ 2:** สร้างใบเสร็จรับเงินอย่างง่าย
```pascal
// ข้อมูล: เลขที่ใบเสร็จ, วันที่, ชื่อผู้ชำระ
// รายการชำระ
// จำนวนเงิน, ภาษี, ยอดสุทธิ
// ตัวอักษรแทนเงิน
```

**ข้อ 3:** สร้างรายงาน inventory (รายงานสต็อกสินค้า)
```pascal
// แสดงสินค้าใกล้หมด (stock < threshold)
// สินค้าที่ไม่มีการขายใน 30 วัน
// มูลค่าสต็อกรวม
// แสดง bar chart ขนาดสต็อก
```

**ข้อ 4:** สร้างรายงานสรุปการขายรายวัน
```pascal
// ยอดขายทุกชั่วโมง
// ยอดขายแยกตาม payment method
// เปรียบเทียบกับวันเดียวกันของสัปดาห์ที่แล้ว
```

**ข้อ 5:** สร้างรายงาน statement ลูกค้า
```pascal
// ยอดค้างชำระ
// ประวัติการซื้อ
// Aging report (0-30, 31-60, 61-90, 90+ วัน)
```

**ข้อ 6:** สร้าง Mailing Label report
```pascal
// แสดงที่อยู่ในรูปแบบ label
// 3 คอลัมน์ต่อหน้า
// Print หลาย labels พร้อมกัน
```

**ข้อ 7:** สร้างรายงานประจำเดือน (Monthly Report)
```pascal
// ยอดขาย, ค่าใช้จ่าย, กำไร
// เปรียบเทียบกับเดือนที่แล้ว
// Line chart แสดงแนวโน้ม
```

**ข้อ 8:** สร้างรายงาน audit trail
```pascal
// การเปลี่ยนแปลงข้อมูลทั้งหมด
// ใคร, เมื่อไร, เปลี่ยนอะไร
// กรองตามช่วงวันเวลา
// กรองตามผู้ใช้
```

**ข้อ 9:** สร้างรายงาน chart ด้วย TChart
```pascal
// Bar chart ยอดขาย 12 เดือน
// Pie chart สัดส่วนยอดขายตามหมวดหมู่
// Line chart แนวโน้มลูกค้าใหม่
// Export chart เป็น image
```

**ข้อ 10:** โปรเจกต์: ระบบรายงานครบชุด
```pascal
// Report manager form
// เลือกรายงาน, กำหนด parameters
// Preview, Print, Export
// Schedule reports (ส่ง email อัตโนมัติ)
// บันทึก report history
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- แนวคิดการออกแบบรายงาน (Bands)
- LazReport สำหรับการสร้างรายงาน
- FPReport (ฟรี มาพร้อม Lazarus)
- FastReport
- การสร้างรายงาน Invoice ด้วย code
- Transcript รายงานผลการเรียน
- Grouped reports
- Export PDF, HTML, Excel

การสร้างรายงานที่ดีต้องคำนึงถึงความสวยงาม, อ่านง่าย และมีข้อมูลครบถ้วน
