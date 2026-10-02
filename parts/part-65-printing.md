# ตอนที่ 65: Printing System ใน Lazarus/Pascal

## บทนำ

Lazarus มี TPrinter class สำหรับการพิมพ์ รองรับทั้ง Windows และ Linux (CUPS) บทนี้จะครอบคลุมการตั้งค่าเครื่องพิมพ์, dialog การพิมพ์, headers/footers และการ print preview

---

## 65.1 พื้นฐาน TPrinter

```pascal
program basic_print;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Printers, Graphics, Dialogs;

procedure PrintSimpleDocument;
var
  P: TPrinter;
  Canvas: TCanvas;
  PageNum: Integer;
begin
  P := Printer;
  
  WriteLn('เครื่องพิมพ์ที่ใช้งาน: ', P.PrinterName);
  WriteLn('จำนวนเครื่องพิมพ์: ', P.Printers.Count);
  
  // แสดงรายการเครื่องพิมพ์
  var i: Integer;
  for i := 0 to P.Printers.Count - 1 do
    WriteLn('  [', i, '] ', P.Printers[i]);
    
  // เริ่มพิมพ์
  P.Title := 'ทดสอบการพิมพ์';
  P.BeginDoc;
  try
    Canvas := P.Canvas;
    
    // ตั้งค่า font
    Canvas.Font.Name := 'TH Sarabun New';
    Canvas.Font.Size := 12;
    Canvas.Font.Color := clBlack;
    
    // พิมพ์หน้าที่ 1
    Canvas.TextOut(100, 100, 'ทดสอบการพิมพ์จาก Lazarus/Pascal');
    Canvas.TextOut(100, 130, 'วันที่: ' + FormatDateTime('dd/mm/yyyy', Now));
    Canvas.TextOut(100, 160, 'เวลา: ' + FormatDateTime('hh:nn:ss', Now));
    
    // วาดเส้น
    Canvas.Pen.Width := 1;
    Canvas.Pen.Color := clBlack;
    Canvas.MoveTo(100, 200);
    Canvas.LineTo(P.PageWidth - 100, 200);
    
    // เพิ่มหน้าใหม่
    P.NewPage;
    
    // พิมพ์หน้าที่ 2
    Canvas.TextOut(100, 100, 'หน้าที่ 2');
    
  finally
    P.EndDoc;
  end;
  
  WriteLn('พิมพ์เสร็จแล้ว');
end;

begin
  PrintSimpleDocument;
end.
```

---

## 65.2 Print Dialog และ Page Setup

```pascal
unit print_dialogs;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls,
  Printers, PrintersDlgs, Dialogs;

type
  TPrintManager = class
  private
    FPrinter: TPrinter;
    FPageWidth, FPageHeight: Integer;
    FMarginLeft, FMarginTop, FMarginRight, FMarginBottom: Integer;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    // แสดง dialogs
    function ShowPrintDialog: Boolean;
    function ShowPageSetupDialog: Boolean;
    function ShowPrinterSetupDialog: Boolean;
    
    // ดึงขนาดหน้ากระดาษ
    procedure GetPageSize(out AWidth, AHeight: Integer);
    
    // แปลง cm เป็น pixel ที่ใช้สำหรับการพิมพ์
    function CmToPixel(ACm: Double): Integer;
    function PixelToCm(APixel: Integer): Double;
    
    property MarginLeft: Integer read FMarginLeft write FMarginLeft;
    property MarginTop: Integer read FMarginTop write FMarginTop;
    property MarginRight: Integer read FMarginRight write FMarginRight;
    property MarginBottom: Integer read FMarginBottom write FMarginBottom;
  end;

implementation

constructor TPrintManager.Create;
begin
  inherited Create;
  FPrinter := Printer;
  
  // Margins ค่าเริ่มต้น (pixels)
  FMarginLeft := 150;
  FMarginTop := 150;
  FMarginRight := 150;
  FMarginBottom := 150;
end;

destructor TPrintManager.Destroy;
begin
  inherited Destroy;
end;

function TPrintManager.ShowPrintDialog: Boolean;
var
  Dlg: TPrintDialog;
begin
  Result := False;
  Dlg := TPrintDialog.Create(nil);
  try
    Dlg.Options := [poPageNums, poSelection, poWarning, poHelp];
    Dlg.Copies := 1;
    Dlg.FromPage := 1;
    Dlg.ToPage := 1;
    Dlg.MaxPage := 999;
    Dlg.MinPage := 1;
    
    Result := Dlg.Execute;
    
    if Result then
    begin
      WriteLn('จำนวนสำเนา: ', Dlg.Copies);
      WriteLn('หน้าที่: ', Dlg.FromPage, ' ถึง ', Dlg.ToPage);
      WriteLn('พิมพ์ทั้งหมด: ', Dlg.PrintRange = prAllPages);
    end;
  finally
    Dlg.Free;
  end;
end;

function TPrintManager.ShowPageSetupDialog: Boolean;
var
  Dlg: TPageSetupDialog;
begin
  Result := False;
  
  // TPageSetupDialog อาจต้องการ unit PrintersDlgs
  Dlg := TPageSetupDialog.Create(nil);
  try
    // ตั้งค่าเริ่มต้น
    Dlg.MarginLeft := FMarginLeft;
    Dlg.MarginTop := FMarginTop;
    Dlg.MarginRight := FMarginRight;
    Dlg.MarginBottom := FMarginBottom;
    
    Result := Dlg.Execute;
    
    if Result then
    begin
      FMarginLeft := Dlg.MarginLeft;
      FMarginTop := Dlg.MarginTop;
      FMarginRight := Dlg.MarginRight;
      FMarginBottom := Dlg.MarginBottom;
    end;
  finally
    Dlg.Free;
  end;
end;

function TPrintManager.ShowPrinterSetupDialog: Boolean;
var
  Dlg: TPrinterSetupDialog;
begin
  Dlg := TPrinterSetupDialog.Create(nil);
  try
    Result := Dlg.Execute;
  finally
    Dlg.Free;
  end;
end;

procedure TPrintManager.GetPageSize(out AWidth, AHeight: Integer);
begin
  AWidth := Printer.PageWidth;
  AHeight := Printer.PageHeight;
end;

function TPrintManager.CmToPixel(ACm: Double): Integer;
begin
  // 1 นิ้ว = 2.54 cm, 1 นิ้ว = DPI pixels
  Result := Round(ACm / 2.54 * Printer.XDPI);
end;

function TPrintManager.PixelToCm(APixel: Integer): Double;
begin
  Result := APixel / Printer.XDPI * 2.54;
end;

end.
```

---

## 65.3 Document Printer Class

```pascal
unit document_printer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Printers, Math;

type
  TTextAlign = (taLeft, taCenter, taRight);
  
  TPrintFont = record
    Name: string;
    Size: Integer;
    Style: TFontStyles;
    Color: TColor;
  end;

  THeaderFooter = record
    LeftText: string;
    CenterText: string;
    RightText: string;
    Font: TPrintFont;
    ShowLine: Boolean;
    LineColor: TColor;
  end;

  TDocumentPrinter = class
  private
    FCanvas: TCanvas;
    FPageWidth, FPageHeight: Integer;
    FMarginLeft, FMarginTop, FMarginRight, FMarginBottom: Integer;
    FCurrentY: Integer;
    FCurrentPage: Integer;
    FPageCount: Integer;
    FHeader: THeaderFooter;
    FFooter: THeaderFooter;
    FTitle: string;
    
    function GetPrintableWidth: Integer;
    function GetPrintableHeight: Integer;
    function GetPrintableLeft: Integer;
    function GetPrintableTop: Integer;
    function GetPrintableBottom: Integer;
    
    procedure DrawHeader;
    procedure DrawFooter;
    procedure CheckNewPage(ARequiredHeight: Integer = 0);
    function ReplaceTokens(const AText: string): string;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    // เริ่ม/จบการพิมพ์
    procedure BeginPrint(const ATitle: string = '');
    procedure EndPrint;
    procedure NewPage;
    
    // พิมพ์ข้อความ
    procedure PrintText(const AText: string; 
      const AFont: TPrintFont; AAlign: TTextAlign = taLeft;
      AX: Integer = -1; AY: Integer = -1);
    procedure PrintLine(const AText: string; 
      ALineHeight: Integer = 0; AAlign: TTextAlign = taLeft);
    procedure PrintWrappedText(const AText: string; AMaxWidth: Integer = 0);
    
    // พิมพ์ตาราง
    procedure BeginTable(const AWidths: array of Integer);
    procedure PrintTableRow(const ACells: array of string; 
      AIsHeader: Boolean = False);
    procedure EndTable;
    
    // วาด elements
    procedure DrawHLine(AY: Integer = -1; AThickness: Integer = 1);
    procedure DrawVLine(AX: Integer; AY1, AY2: Integer; AThickness: Integer = 1);
    procedure DrawRect(AX1, AY1, AX2, AY2: Integer; 
      ABrushColor: TColor = clNone; APenColor: TColor = clBlack);
    procedure DrawImage(ABitmap: TBitmap; AX, AY, AWidth, AHeight: Integer);
    
    // ย้าย cursor
    procedure MoveDown(APixels: Integer);
    procedure SetY(AY: Integer);
    
    // Header/Footer
    property Header: THeaderFooter read FHeader write FHeader;
    property Footer: THeaderFooter read FFooter write FFooter;
    
    // ขนาด
    property PageWidth: Integer read FPageWidth;
    property PageHeight: Integer read FPageHeight;
    property PrintableWidth: Integer read GetPrintableWidth;
    property CurrentPage: Integer read FCurrentPage;
    property CurrentY: Integer read FCurrentY;
    
    // Margins (pixels)
    property MarginLeft: Integer read FMarginLeft write FMarginLeft;
    property MarginTop: Integer read FMarginTop write FMarginTop;
    property MarginRight: Integer read FMarginRight write FMarginRight;
    property MarginBottom: Integer read FMarginBottom write FMarginBottom;
  end;

  // ฟังก์ชันสร้าง Font record
  function MakeFont(const AName: string; ASize: Integer; 
    AStyle: TFontStyles = []; AColor: TColor = clBlack): TPrintFont;

implementation

function MakeFont(const AName: string; ASize: Integer; 
  AStyle: TFontStyles; AColor: TColor): TPrintFont;
begin
  Result.Name := AName;
  Result.Size := ASize;
  Result.Style := AStyle;
  Result.Color := AColor;
end;

{ TDocumentPrinter }

constructor TDocumentPrinter.Create;
begin
  inherited Create;
  
  // Margins ค่าเริ่มต้น
  FMarginLeft := 150;
  FMarginTop := 150;
  FMarginRight := 150;
  FMarginBottom := 150;
  
  // Header default
  FHeader.LeftText := '';
  FHeader.CenterText := '{TITLE}';
  FHeader.RightText := 'หน้า {PAGE}/{PAGES}';
  FHeader.Font := MakeFont('TH Sarabun New', 9);
  FHeader.ShowLine := True;
  FHeader.LineColor := clGray;
  
  // Footer default
  FFooter.LeftText := 'พิมพ์เมื่อ: {DATE} {TIME}';
  FFooter.CenterText := '';
  FFooter.RightText := '';
  FFooter.Font := MakeFont('TH Sarabun New', 9);
  FFooter.ShowLine := True;
  FFooter.LineColor := clGray;
end;

destructor TDocumentPrinter.Destroy;
begin
  inherited Destroy;
end;

function TDocumentPrinter.GetPrintableWidth: Integer;
begin
  Result := FPageWidth - FMarginLeft - FMarginRight;
end;

function TDocumentPrinter.GetPrintableHeight: Integer;
begin
  Result := FPageHeight - FMarginTop - FMarginBottom;
end;

function TDocumentPrinter.GetPrintableLeft: Integer;
begin
  Result := FMarginLeft;
end;

function TDocumentPrinter.GetPrintableTop: Integer;
begin
  Result := FMarginTop + 80;  // เว้นที่สำหรับ header
end;

function TDocumentPrinter.GetPrintableBottom: Integer;
begin
  Result := FPageHeight - FMarginBottom - 80;  // เว้นที่สำหรับ footer
end;

function TDocumentPrinter.ReplaceTokens(const AText: string): string;
begin
  Result := AText;
  Result := StringReplace(Result, '{TITLE}', FTitle, [rfReplaceAll]);
  Result := StringReplace(Result, '{PAGE}', IntToStr(FCurrentPage), [rfReplaceAll]);
  Result := StringReplace(Result, '{PAGES}', IntToStr(FPageCount), [rfReplaceAll]);
  Result := StringReplace(Result, '{DATE}', FormatDateTime('dd/mm/yyyy', Now), [rfReplaceAll]);
  Result := StringReplace(Result, '{TIME}', FormatDateTime('hh:nn:ss', Now), [rfReplaceAll]);
end;

procedure TDocumentPrinter.DrawHeader;
var
  HeaderY: Integer;
  TextHeight: Integer;
begin
  if (FHeader.LeftText = '') and (FHeader.CenterText = '') and 
     (FHeader.RightText = '') then Exit;
     
  HeaderY := FMarginTop;
  
  // ตั้งค่า font
  with FCanvas.Font do
  begin
    Name := FHeader.Font.Name;
    Size := FHeader.Font.Size;
    Style := FHeader.Font.Style;
    Color := FHeader.Font.Color;
  end;
  
  TextHeight := FCanvas.TextHeight('X');
  
  // Left text
  if FHeader.LeftText <> '' then
    FCanvas.TextOut(FMarginLeft, HeaderY, ReplaceTokens(FHeader.LeftText));
    
  // Center text
  if FHeader.CenterText <> '' then
  begin
    var S := ReplaceTokens(FHeader.CenterText);
    var W := FCanvas.TextWidth(S);
    FCanvas.TextOut(FPageWidth div 2 - W div 2, HeaderY, S);
  end;
  
  // Right text
  if FHeader.RightText <> '' then
  begin
    var S := ReplaceTokens(FHeader.RightText);
    var W := FCanvas.TextWidth(S);
    FCanvas.TextOut(FPageWidth - FMarginRight - W, HeaderY, S);
  end;
  
  // เส้นใต้ header
  if FHeader.ShowLine then
  begin
    FCanvas.Pen.Color := FHeader.LineColor;
    FCanvas.Pen.Width := 1;
    FCanvas.MoveTo(FMarginLeft, HeaderY + TextHeight + 5);
    FCanvas.LineTo(FPageWidth - FMarginRight, HeaderY + TextHeight + 5);
  end;
end;

procedure TDocumentPrinter.DrawFooter;
var
  FooterY: Integer;
begin
  FooterY := FPageHeight - FMarginBottom - 50;
  
  // เส้นเหนือ footer
  if FFooter.ShowLine then
  begin
    FCanvas.Pen.Color := FFooter.LineColor;
    FCanvas.Pen.Width := 1;
    FCanvas.MoveTo(FMarginLeft, FooterY);
    FCanvas.LineTo(FPageWidth - FMarginRight, FooterY);
    Inc(FooterY, 10);
  end;
  
  with FCanvas.Font do
  begin
    Name := FFooter.Font.Name;
    Size := FFooter.Font.Size;
    Style := FFooter.Font.Style;
    Color := FFooter.Font.Color;
  end;
  
  if FFooter.LeftText <> '' then
    FCanvas.TextOut(FMarginLeft, FooterY, ReplaceTokens(FFooter.LeftText));
    
  if FFooter.CenterText <> '' then
  begin
    var S := ReplaceTokens(FFooter.CenterText);
    var W := FCanvas.TextWidth(S);
    FCanvas.TextOut(FPageWidth div 2 - W div 2, FooterY, S);
  end;
  
  if FFooter.RightText <> '' then
  begin
    var S := ReplaceTokens(FFooter.RightText);
    var W := FCanvas.TextWidth(S);
    FCanvas.TextOut(FPageWidth - FMarginRight - W, FooterY, S);
  end;
end;

procedure TDocumentPrinter.CheckNewPage(ARequiredHeight: Integer);
begin
  if (FCurrentY + ARequiredHeight) > GetPrintableBottom then
    NewPage;
end;

procedure TDocumentPrinter.BeginPrint(const ATitle: string);
begin
  FTitle := ATitle;
  FCurrentPage := 1;
  FPageCount := 1;  // จะอัปเดตภายหลัง
  
  Printer.Title := ATitle;
  Printer.BeginDoc;
  
  FCanvas := Printer.Canvas;
  FPageWidth := Printer.PageWidth;
  FPageHeight := Printer.PageHeight;
  
  FCurrentY := GetPrintableTop;
  
  DrawHeader;
end;

procedure TDocumentPrinter.EndPrint;
begin
  DrawFooter;
  Printer.EndDoc;
end;

procedure TDocumentPrinter.NewPage;
begin
  DrawFooter;
  Inc(FCurrentPage);
  Inc(FPageCount);
  Printer.NewPage;
  FCurrentY := GetPrintableTop;
  DrawHeader;
end;

procedure TDocumentPrinter.PrintText(const AText: string; 
  const AFont: TPrintFont; AAlign: TTextAlign; AX, AY: Integer);
var
  X, Y: Integer;
  TextWidth: Integer;
begin
  // ตั้งค่า font
  with FCanvas.Font do
  begin
    Name := AFont.Name;
    Size := AFont.Size;
    Style := AFont.Style;
    Color := AFont.Color;
  end;
  
  TextWidth := FCanvas.TextWidth(AText);
  
  if AX >= 0 then X := AX else X := FMarginLeft;
  if AY >= 0 then Y := AY else Y := FCurrentY;
  
  case AAlign of
    taLeft:   ;  // X ไม่เปลี่ยน
    taCenter: X := FPageWidth div 2 - TextWidth div 2;
    taRight:  X := FPageWidth - FMarginRight - TextWidth;
  end;
  
  FCanvas.TextOut(X, Y, AText);
  
  if AY < 0 then
    FCurrentY := Y + FCanvas.TextHeight(AText);
end;

procedure TDocumentPrinter.PrintLine(const AText: string; 
  ALineHeight: Integer; AAlign: TTextAlign);
var
  Height: Integer;
begin
  if ALineHeight > 0 then
    Height := ALineHeight
  else
    Height := FCanvas.TextHeight('X') + 4;
    
  CheckNewPage(Height);
  
  case AAlign of
    taLeft:
      FCanvas.TextOut(FMarginLeft, FCurrentY, AText);
    taCenter:
      begin
        var W := FCanvas.TextWidth(AText);
        FCanvas.TextOut(FPageWidth div 2 - W div 2, FCurrentY, AText);
      end;
    taRight:
      begin
        var W := FCanvas.TextWidth(AText);
        FCanvas.TextOut(FPageWidth - FMarginRight - W, FCurrentY, AText);
      end;
  end;
  
  FCurrentY := FCurrentY + Height;
end;

procedure TDocumentPrinter.PrintWrappedText(const AText: string; AMaxWidth: Integer);
var
  MaxWidth: Integer;
  Words: TStringList;
  Line, TestLine: string;
  i: Integer;
  LineHeight: Integer;
begin
  if AMaxWidth > 0 then MaxWidth := AMaxWidth else MaxWidth := GetPrintableWidth;
  LineHeight := FCanvas.TextHeight('X') + 4;
  
  Words := TStringList.Create;
  try
    Words.Delimiter := ' ';
    Words.DelimitedText := AText;
    
    Line := '';
    for i := 0 to Words.Count - 1 do
    begin
      if Line = '' then
        TestLine := Words[i]
      else
        TestLine := Line + ' ' + Words[i];
        
      if FCanvas.TextWidth(TestLine) <= MaxWidth then
        Line := TestLine
      else
      begin
        if Line <> '' then
        begin
          CheckNewPage(LineHeight);
          FCanvas.TextOut(FMarginLeft, FCurrentY, Line);
          FCurrentY := FCurrentY + LineHeight;
        end;
        Line := Words[i];
      end;
    end;
    
    if Line <> '' then
    begin
      CheckNewPage(LineHeight);
      FCanvas.TextOut(FMarginLeft, FCurrentY, Line);
      FCurrentY := FCurrentY + LineHeight;
    end;
  finally
    Words.Free;
  end;
end;

var
  FTableWidths: array of Integer;
  FTableStartX: Integer;

procedure TDocumentPrinter.BeginTable(const AWidths: array of Integer);
var
  i: Integer;
begin
  SetLength(FTableWidths, Length(AWidths));
  for i := 0 to High(AWidths) do
    FTableWidths[i] := AWidths[i];
  FTableStartX := FMarginLeft;
end;

procedure TDocumentPrinter.PrintTableRow(const ACells: array of string; 
  AIsHeader: Boolean);
var
  i, X, Y: Integer;
  CellHeight, ColWidth: Integer;
  TotalWidth: Integer;
begin
  CellHeight := FCanvas.TextHeight('X') + 10;
  CheckNewPage(CellHeight);
  
  Y := FCurrentY;
  X := FTableStartX;
  
  // คำนวณ total width
  TotalWidth := 0;
  for i := 0 to High(FTableWidths) do
    TotalWidth := TotalWidth + FTableWidths[i];
    
  // วาดพื้นหลัง header
  if AIsHeader then
  begin
    FCanvas.Brush.Color := $00DDDDDD;
    FCanvas.Brush.Style := bsSolid;
    FCanvas.FillRect(X, Y, X + TotalWidth, Y + CellHeight);
    FCanvas.Brush.Style := bsClear;
    FCanvas.Font.Style := [fsBold];
  end
  else
    FCanvas.Font.Style := [];
    
  // วาดเซลล์
  for i := 0 to High(ACells) do
  begin
    if i > High(FTableWidths) then Break;
    ColWidth := FTableWidths[i];
    
    // วาดกรอบเซลล์
    FCanvas.Pen.Color := clBlack;
    FCanvas.Pen.Width := 1;
    FCanvas.Rectangle(X, Y, X + ColWidth, Y + CellHeight);
    
    // วาดข้อความ
    if i < Length(ACells) then
      FCanvas.TextOut(X + 5, Y + 5, ACells[i]);
      
    X := X + ColWidth;
  end;
  
  FCurrentY := Y + CellHeight;
end;

procedure TDocumentPrinter.EndTable;
begin
  SetLength(FTableWidths, 0);
  MoveDown(5);
end;

procedure TDocumentPrinter.DrawHLine(AY: Integer; AThickness: Integer);
var
  Y: Integer;
begin
  if AY < 0 then Y := FCurrentY else Y := AY;
  
  FCanvas.Pen.Width := AThickness;
  FCanvas.Pen.Color := clBlack;
  FCanvas.MoveTo(FMarginLeft, Y);
  FCanvas.LineTo(FPageWidth - FMarginRight, Y);
  
  if AY < 0 then
    FCurrentY := Y + AThickness + 5;
end;

procedure TDocumentPrinter.DrawVLine(AX, AY1, AY2, AThickness: Integer);
begin
  FCanvas.Pen.Width := AThickness;
  FCanvas.Pen.Color := clBlack;
  FCanvas.MoveTo(AX, AY1);
  FCanvas.LineTo(AX, AY2);
end;

procedure TDocumentPrinter.DrawRect(AX1, AY1, AX2, AY2: Integer; 
  ABrushColor, APenColor: TColor);
begin
  FCanvas.Pen.Color := APenColor;
  if ABrushColor = clNone then
    FCanvas.Brush.Style := bsClear
  else
  begin
    FCanvas.Brush.Color := ABrushColor;
    FCanvas.Brush.Style := bsSolid;
  end;
  FCanvas.Rectangle(AX1, AY1, AX2, AY2);
  FCanvas.Brush.Style := bsClear;
end;

procedure TDocumentPrinter.DrawImage(ABitmap: TBitmap; AX, AY, AWidth, AHeight: Integer);
begin
  FCanvas.StretchDraw(Rect(AX, AY, AX + AWidth, AY + AHeight), ABitmap);
end;

procedure TDocumentPrinter.MoveDown(APixels: Integer);
begin
  FCurrentY := FCurrentY + APixels;
end;

procedure TDocumentPrinter.SetY(AY: Integer);
begin
  FCurrentY := AY;
end;

end.
```

---

## 65.4 ตัวอย่าง: พิมพ์ใบแจ้งหนี้

```pascal
unit print_invoice;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Printers,
  document_printer;

type
  TInvoiceItem = record
    Description: string;
    Quantity: Integer;
    UnitPrice: Double;
    Total: Double;
  end;

  TInvoice = record
    InvoiceNo: string;
    InvoiceDate: TDate;
    DueDate: TDate;
    
    // ข้อมูลลูกค้า
    CustomerName: string;
    CustomerAddress: string;
    CustomerPhone: string;
    CustomerTaxID: string;
    
    // ข้อมูลบริษัท
    CompanyName: string;
    CompanyAddress: string;
    CompanyPhone: string;
    CompanyTaxID: string;
    
    // รายการสินค้า
    Items: array of TInvoiceItem;
    
    // ยอดเงิน
    SubTotal: Double;
    DiscountPercent: Double;
    VATPercent: Double;
    Total: Double;
    
    Note: string;
  end;

procedure PrintInvoice(const AInvoice: TInvoice);
procedure PrintInvoicePreview(const AInvoice: TInvoice; APreviewForm: TForm);

implementation

procedure PrintInvoice(const AInvoice: TInvoice);
var
  DP: TDocumentPrinter;
  i: Integer;
  NormalFont, BoldFont, SmallFont, HeaderFont: TPrintFont;
  ColWidths: array[0..4] of Integer;
begin
  DP := TDocumentPrinter.Create;
  try
    // ตั้งค่า fonts
    NormalFont := MakeFont('TH Sarabun New', 11);
    BoldFont := MakeFont('TH Sarabun New', 11, [fsBold]);
    SmallFont := MakeFont('TH Sarabun New', 9);
    HeaderFont := MakeFont('TH Sarabun New', 14, [fsBold]);
    
    // ตั้งค่า header/footer
    DP.Header.LeftText := AInvoice.CompanyName;
    DP.Header.CenterText := '';
    DP.Header.RightText := 'ใบแจ้งหนี้ #{INV}';
    DP.Header.Font := SmallFont;
    DP.Footer.LeftText := 'พิมพ์เมื่อ: {DATE} {TIME}';
    DP.Footer.RightText := 'หน้า {PAGE}';
    
    // เริ่มพิมพ์
    DP.BeginPrint('ใบแจ้งหนี้ ' + AInvoice.InvoiceNo);
    
    // ชื่อบริษัท
    DP.PrintText(AInvoice.CompanyName, HeaderFont, taCenter);
    DP.MoveDown(5);
    DP.PrintText(AInvoice.CompanyAddress, SmallFont, taCenter);
    DP.PrintText('โทร: ' + AInvoice.CompanyPhone + 
                 '  เลขที่ผู้เสียภาษี: ' + AInvoice.CompanyTaxID, 
                 SmallFont, taCenter);
    DP.MoveDown(10);
    DP.DrawHLine(-1, 2);
    DP.MoveDown(10);
    
    // หัว "ใบแจ้งหนี้"
    DP.PrintText('ใบแจ้งหนี้ / TAX INVOICE', 
      MakeFont('TH Sarabun New', 18, [fsBold]), taCenter);
    DP.MoveDown(5);
    
    // เลขที่และวันที่
    var Y1 := DP.CurrentY;
    
    // ข้อมูลลูกค้า (ซ้าย)
    var PrintX := DP.MarginLeft;
    DP.PrintText('ชื่อลูกค้า: ' + AInvoice.CustomerName, NormalFont, taLeft, PrintX, Y1);
    DP.PrintText('ที่อยู่: ' + AInvoice.CustomerAddress, NormalFont, taLeft, PrintX, Y1 + 25);
    DP.PrintText('โทร: ' + AInvoice.CustomerPhone, NormalFont, taLeft, PrintX, Y1 + 50);
    DP.PrintText('เลขที่ผู้เสียภาษี: ' + AInvoice.CustomerTaxID, NormalFont, taLeft, PrintX, Y1 + 75);
    
    // เลขที่ใบแจ้งหนี้ (ขวา)
    var PrintW := DP.PrintableWidth;
    DP.PrintText('เลขที่: ' + AInvoice.InvoiceNo, BoldFont, taRight, -1, Y1);
    DP.PrintText('วันที่: ' + FormatDateTime('dd/mm/yyyy', AInvoice.InvoiceDate), 
      NormalFont, taRight, -1, Y1 + 25);
    DP.PrintText('ครบกำหนด: ' + FormatDateTime('dd/mm/yyyy', AInvoice.DueDate), 
      NormalFont, taRight, -1, Y1 + 50);
    
    DP.SetY(Y1 + 100);
    DP.MoveDown(10);
    DP.DrawHLine;
    
    // ตารางรายการ
    // ความกว้าง: ลำดับ, รายการ, จำนวน, ราคา/หน่วย, รวม
    var TW := DP.PrintableWidth;
    ColWidths[0] := 60;
    ColWidths[1] := TW - 60 - 80 - 120 - 120;
    ColWidths[2] := 80;
    ColWidths[3] := 120;
    ColWidths[4] := 120;
    
    DP.BeginTable(ColWidths);
    
    // Header row
    DP.PrintTableRow(['ลำดับ', 'รายการ', 'จำนวน', 'ราคา/หน่วย', 'รวม'], True);
    
    // Data rows
    for i := 0 to High(AInvoice.Items) do
    begin
      with AInvoice.Items[i] do
      begin
        DP.PrintTableRow([
          IntToStr(i + 1),
          Description,
          IntToStr(Quantity),
          FormatFloat('#,##0.00', UnitPrice),
          FormatFloat('#,##0.00', Total)
        ]);
      end;
    end;
    
    DP.EndTable;
    
    // ยอดรวม
    var SummaryX := DP.PageWidth - DP.MarginRight - 250;
    var SummaryY := DP.CurrentY;
    
    DP.PrintText('ยอดรวมก่อน VAT:', NormalFont, taLeft, SummaryX, SummaryY);
    DP.PrintText(FormatFloat('#,##0.00', AInvoice.SubTotal) + ' บาท',
      NormalFont, taRight, -1, SummaryY);
    SummaryY := SummaryY + 25;
    
    if AInvoice.DiscountPercent > 0 then
    begin
      DP.PrintText(Format('ส่วนลด (%.0f%%):', [AInvoice.DiscountPercent]),
        NormalFont, taLeft, SummaryX, SummaryY);
      DP.PrintText(FormatFloat('#,##0.00', 
        AInvoice.SubTotal * AInvoice.DiscountPercent / 100) + ' บาท',
        NormalFont, taRight, -1, SummaryY);
      SummaryY := SummaryY + 25;
    end;
    
    if AInvoice.VATPercent > 0 then
    begin
      DP.PrintText(Format('VAT (%.0f%%):', [AInvoice.VATPercent]),
        NormalFont, taLeft, SummaryX, SummaryY);
      var VATAmount := AInvoice.SubTotal * (1 - AInvoice.DiscountPercent/100) 
                       * AInvoice.VATPercent / 100;
      DP.PrintText(FormatFloat('#,##0.00', VATAmount) + ' บาท',
        NormalFont, taRight, -1, SummaryY);
      SummaryY := SummaryY + 25;
    end;
    
    // เส้นแยก
    DP.DrawHLine(SummaryY, 1);
    SummaryY := SummaryY + 5;
    
    // ยอดรวมสุทธิ
    DP.PrintText('ยอดรวมสุทธิ:', BoldFont, taLeft, SummaryX, SummaryY);
    DP.PrintText(FormatFloat('#,##0.00', AInvoice.Total) + ' บาท',
      MakeFont('TH Sarabun New', 11, [fsBold], clRed), taRight, -1, SummaryY);
    
    DP.SetY(SummaryY + 40);
    
    // หมายเหตุ
    if AInvoice.Note <> '' then
    begin
      DP.MoveDown(20);
      DP.DrawHLine;
      DP.MoveDown(5);
      DP.PrintText('หมายเหตุ:', BoldFont);
      DP.MoveDown(3);
      DP.PrintWrappedText(AInvoice.Note);
    end;
    
    // ลายเซ็น
    DP.MoveDown(50);
    var SignY := DP.CurrentY;
    var SignW := DP.PrintableWidth div 2 - 50;
    
    DP.DrawHLine(SignY + 60, 1);
    // เส้นลายเซ็น
    
    // ลายเซ็นลูกค้า
    DP.DrawVLine(DP.MarginLeft + SignW, SignY, SignY + 70);
    
    DP.PrintText('_____________________', NormalFont, taLeft, DP.MarginLeft + 50, SignY + 40);
    DP.PrintText('(ผู้รับสินค้า/ผู้ซื้อ)', SmallFont, taLeft, DP.MarginLeft + 50, SignY + 60);
    DP.PrintText('วันที่ ___________', SmallFont, taLeft, DP.MarginLeft + 50, SignY + 75);
    
    // ลายเซ็นบริษัท
    var CompanySignX := DP.PageWidth - DP.MarginRight - SignW;
    DP.PrintText('_____________________', NormalFont, taLeft, CompanySignX + 20, SignY + 40);
    DP.PrintText('(ผู้มีอำนาจลงนาม)', SmallFont, taLeft, CompanySignX + 30, SignY + 60);
    DP.PrintText('วันที่ ___________', SmallFont, taLeft, CompanySignX + 30, SignY + 75);
    
    DP.EndPrint;
    
  finally
    DP.Free;
  end;
end;

// ตัวอย่างสร้างและพิมพ์ใบแจ้งหนี้
procedure DemoPrintInvoice;
var
  Inv: TInvoice;
begin
  with Inv do
  begin
    InvoiceNo := 'INV-2024-00001';
    InvoiceDate := Today;
    DueDate := Today + 30;
    
    CompanyName := 'บริษัท ตัวอย่าง จำกัด';
    CompanyAddress := '123 ถนนตัวอย่าง แขวงตัวอย่าง เขตตัวอย่าง กรุงเทพ 10000';
    CompanyPhone := '02-123-4567';
    CompanyTaxID := '0105566012345';
    
    CustomerName := 'นาย ลูกค้า ตัวอย่าง';
    CustomerAddress := '456 ถนนลูกค้า กรุงเทพ 10100';
    CustomerPhone := '081-234-5678';
    CustomerTaxID := '1234567890123';
    
    SetLength(Items, 3);
    Items[0] := (Description: 'สินค้า A - รุ่นพิเศษ'; Quantity: 2; UnitPrice: 1500; Total: 3000);
    Items[1] := (Description: 'สินค้า B - บริการติดตั้ง'; Quantity: 1; UnitPrice: 500; Total: 500);
    Items[2] := (Description: 'สินค้า C - อะไหล่ทดแทน'; Quantity: 5; UnitPrice: 200; Total: 1000);
    
    SubTotal := 4500;
    DiscountPercent := 5;
    VATPercent := 7;
    Total := SubTotal * (1 - DiscountPercent/100) * (1 + VATPercent/100);
    
    Note := 'กรุณาชำระเงินภายใน 30 วัน โอนเงินได้ที่ ธนาคาร XXX เลขที่บัญชี 000-000-0000';
  end;
  
  PrintInvoice(Inv);
end;

end.
```

---

## 65.5 Print Preview

```pascal
unit print_preview;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  StdCtrls, ExtCtrls, ComCtrls, Dialogs,
  Printers;

type
  TPrintPreviewForm = class(TForm)
    pnlTop: TPanel;
    btnPrint: TButton;
    btnClose: TButton;
    btnZoomIn: TButton;
    btnZoomOut: TButton;
    lblZoom: TLabel;
    ScrollBox: TScrollBox;
    imgPreview: TImage;
    
    procedure FormCreate(Sender: TObject);
    procedure btnPrintClick(Sender: TObject);
    procedure btnZoomInClick(Sender: TObject);
    procedure btnZoomOutClick(Sender: TObject);
    
  private
    FPages: TList;  // TBitmap list
    FCurrentPage: Integer;
    FZoom: Double;
    
    procedure RenderPreview;
    procedure UpdateZoomLabel;
    
  public
    destructor Destroy; override;
    
    procedure AddPage(ABitmap: TBitmap);
    procedure ClearPages;
    procedure ShowPage(APageNum: Integer);
    
    property CurrentPage: Integer read FCurrentPage;
    property PageCount: Integer read (FPages.Count);
  end;

  // ฟังก์ชันสร้าง preview จาก document
  procedure CreatePrintPreview(APreviewForm: TPrintPreviewForm; 
    APrintProc: TNotifyEvent);

implementation

procedure TPrintPreviewForm.FormCreate(Sender: TObject);
begin
  Caption := 'Print Preview';
  Width := 700;
  Height := 900;
  FPages := TList.Create;
  FCurrentPage := 0;
  FZoom := 1.0;
end;

destructor TPrintPreviewForm.Destroy;
begin
  ClearPages;
  FPages.Free;
  inherited Destroy;
end;

procedure TPrintPreviewForm.ClearPages;
var
  i: Integer;
begin
  for i := 0 to FPages.Count - 1 do
    TBitmap(FPages[i]).Free;
  FPages.Clear;
  FCurrentPage := 0;
end;

procedure TPrintPreviewForm.AddPage(ABitmap: TBitmap);
begin
  FPages.Add(ABitmap);
end;

procedure TPrintPreviewForm.ShowPage(APageNum: Integer);
begin
  if (APageNum >= 0) and (APageNum < FPages.Count) then
  begin
    FCurrentPage := APageNum;
    RenderPreview;
  end;
end;

procedure TPrintPreviewForm.RenderPreview;
var
  SrcBitmap: TBitmap;
  W, H: Integer;
begin
  if FPages.Count = 0 then Exit;
  
  SrcBitmap := TBitmap(FPages[FCurrentPage]);
  W := Round(SrcBitmap.Width * FZoom);
  H := Round(SrcBitmap.Height * FZoom);
  
  imgPreview.Width := W;
  imgPreview.Height := H;
  
  imgPreview.Picture.Bitmap.Width := W;
  imgPreview.Picture.Bitmap.Height := H;
  
  imgPreview.Picture.Bitmap.Canvas.StretchDraw(
    Rect(0, 0, W, H), SrcBitmap);
end;

procedure TPrintPreviewForm.UpdateZoomLabel;
begin
  lblZoom.Caption := Format('%.0f%%', [FZoom * 100]);
end;

procedure TPrintPreviewForm.btnPrintClick(Sender: TObject);
begin
  // พิมพ์จริง
  if MessageDlg('พิมพ์', 'ต้องการพิมพ์เอกสารหรือไม่?',
    mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    // ... เรียก print function
    Close;
  end;
end;

procedure TPrintPreviewForm.btnZoomInClick(Sender: TObject);
begin
  if FZoom < 2.0 then
  begin
    FZoom := FZoom + 0.1;
    RenderPreview;
    UpdateZoomLabel;
  end;
end;

procedure TPrintPreviewForm.btnZoomOutClick(Sender: TObject);
begin
  if FZoom > 0.2 then
  begin
    FZoom := FZoom - 0.1;
    RenderPreview;
    UpdateZoomLabel;
  end;
end;

procedure CreatePrintPreview(APreviewForm: TPrintPreviewForm; 
  APrintProc: TNotifyEvent);
var
  OldPrinter: string;
  Bitmap: TBitmap;
begin
  // สร้าง preview โดยพิมพ์ไปยัง Bitmap
  // (ใน Lazarus ทำผ่าน TPostScript หรือ TPDFDocument)
  
  Bitmap := TBitmap.Create;
  try
    Bitmap.Width := 794;   // A4 at 96 DPI
    Bitmap.Height := 1123;
    Bitmap.Canvas.Brush.Color := clWhite;
    Bitmap.Canvas.FillRect(0, 0, Bitmap.Width, Bitmap.Height);
    
    // จำลองการวาด
    Bitmap.Canvas.Font.Name := 'TH Sarabun New';
    Bitmap.Canvas.Font.Size := 16;
    Bitmap.Canvas.TextOut(50, 50, 'Print Preview - หน้าตัวอย่าง');
    Bitmap.Canvas.Font.Size := 12;
    Bitmap.Canvas.TextOut(50, 100, 'เนื้อหาของเอกสาร...');
    
    APreviewForm.AddPage(Bitmap);
    Bitmap := nil;  // ownership transferred
    
  finally
    Bitmap.Free;
  end;
  
  APreviewForm.ShowPage(0);
end;

end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **TPrinter** - การใช้งานเบื้องต้น
2. **Print Dialog** - แสดง dialog เลือกเครื่องพิมพ์
3. **Document Printer** - คลาสช่วยพิมพ์เอกสาร
4. **Headers/Footers** - ส่วนหัวและส่วนท้ายของหน้า
5. **Invoice Printing** - พิมพ์ใบแจ้งหนี้สมบูรณ์
6. **Print Preview** - ดูตัวอย่างก่อนพิมพ์

ระบบการพิมพ์ใน Lazarus มีความยืดหยุ่นสูง สามารถวาด graphics, ตาราง และข้อความได้โดยตรงบน Canvas ของเครื่องพิมพ์
