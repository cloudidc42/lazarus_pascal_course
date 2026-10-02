# ตอนที่ 67: Excel Integration ใน Lazarus/Pascal

## บทนำ

บทนี้จะครอบคลุมการอ่าน/เขียน Excel ใน Lazarus ด้วย fpSpreadsheet library ซึ่งรองรับ XLSX, XLS, ODS และรูปแบบอื่นๆ โดยไม่ต้องติดตั้ง Microsoft Office

---

## 67.1 ติดตั้ง fpSpreadsheet

```bash
# ใน Lazarus: Package -> Install/Uninstall Packages
# เพิ่ม: fpspreadsheet (มักมาพร้อม Lazarus)
# หรือจาก OPM: lazarus-fpspreadsheet

# ใน .lpr ให้เพิ่ม:
uses
  fpspreadsheet,
  fpsAllFormats,   // โหลด format ทั้งหมด
  xlsxooxml;       // สำหรับ XLSX
```

---

## 67.2 การเขียน Excel

```pascal
unit excel_writer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics,
  fpspreadsheet,
  fpsTypes,
  xlsxooxml,
  fpsAllFormats;

type
  TCellStyle = record
    Bold: Boolean;
    Italic: Boolean;
    FontSize: Integer;
    FontColor: TsColor;
    BackColor: TsColor;
    HAlign: TsHorAlignment;
    VAlign: TsVertAlignment;
    Borders: TsCellBorders;
    WrapText: Boolean;
    NumberFormat: string;
  end;

  TExcelWriter = class
  private
    FWorkbook: TsWorkbook;
    FCurrentSheet: TsWorksheet;
    FSheetIndex: Integer;
    
    function MakeStyle(const AStyle: TCellStyle): Integer;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    // จัดการ Sheet
    function AddSheet(const AName: string): TsWorksheet;
    procedure SetActiveSheet(AIndex: Integer); overload;
    procedure SetActiveSheet(const AName: string); overload;
    
    // เขียนข้อมูล
    procedure WriteString(ARow, ACol: Integer; const AValue: string);
    procedure WriteNumber(ARow, ACol: Integer; AValue: Double);
    procedure WriteDate(ARow, ACol: Integer; AValue: TDateTime);
    procedure WriteBool(ARow, ACol: Integer; AValue: Boolean);
    procedure WriteFormula(ARow, ACol: Integer; const AFormula: string);
    
    // จัดรูปแบบ
    procedure SetCellFormat(ARow, ACol: Integer; const AStyle: TCellStyle);
    procedure SetColumnWidth(ACol: Integer; AWidth: Double);
    procedure SetRowHeight(ARow: Integer; AHeight: Double);
    procedure MergeCells(ARow1, ACol1, ARow2, ACol2: Integer);
    procedure FreezeRow(ARow: Integer);
    procedure FreezeColumn(ACol: Integer);
    
    // บันทึก
    procedure SaveToFile(const AFileName: string);
    procedure SaveToStream(AStream: TStream; AFormat: TsSpreadsheetFormat = sfOOXML);
    
    property Workbook: TsWorkbook read FWorkbook;
    property ActiveSheet: TsWorksheet read FCurrentSheet;
  end;

  // ฟังก์ชันสร้าง Style
  function MakeHeaderStyle: TCellStyle;
  function MakeDataStyle(ABold: Boolean = False): TCellStyle;
  function MakeNumberStyle(const AFormat: string = '#,##0.00'): TCellStyle;

implementation

function MakeHeaderStyle: TCellStyle;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.Bold := True;
  Result.FontSize := 11;
  Result.FontColor := $00FFFFFF;  // White
  Result.BackColor := $00336699;  // Dark blue
  Result.HAlign := haCenter;
  Result.VAlign := vaCenter;
  Result.Borders := [cbNorth, cbSouth, cbEast, cbWest];
  Result.WrapText := True;
end;

function MakeDataStyle(ABold: Boolean): TCellStyle;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.Bold := ABold;
  Result.FontSize := 10;
  Result.FontColor := $00000000;  // Black
  Result.BackColor := TsColor(-1);  // No fill
  Result.HAlign := haLeft;
  Result.VAlign := vaCenter;
end;

function MakeNumberStyle(const AFormat: string): TCellStyle;
begin
  Result := MakeDataStyle;
  Result.HAlign := haRight;
  Result.NumberFormat := AFormat;
end;

{ TExcelWriter }

constructor TExcelWriter.Create;
begin
  inherited Create;
  FWorkbook := TsWorkbook.Create;
  FSheetIndex := 0;
end;

destructor TExcelWriter.Destroy;
begin
  FWorkbook.Free;
  inherited Destroy;
end;

function TExcelWriter.AddSheet(const AName: string): TsWorksheet;
begin
  Result := FWorkbook.AddWorksheet(AName);
  FCurrentSheet := Result;
  Inc(FSheetIndex);
end;

procedure TExcelWriter.SetActiveSheet(AIndex: Integer);
begin
  FCurrentSheet := FWorkbook.GetWorksheetByIndex(AIndex);
end;

procedure TExcelWriter.SetActiveSheet(const AName: string);
begin
  FCurrentSheet := FWorkbook.GetWorksheetByName(AName);
end;

procedure TExcelWriter.WriteString(ARow, ACol: Integer; const AValue: string);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteUTF8Text(ARow, ACol, AValue);
end;

procedure TExcelWriter.WriteNumber(ARow, ACol: Integer; AValue: Double);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteNumber(ARow, ACol, AValue);
end;

procedure TExcelWriter.WriteDate(ARow, ACol: Integer; AValue: TDateTime);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteDateTime(ARow, ACol, AValue, nfShortDate);
end;

procedure TExcelWriter.WriteBool(ARow, ACol: Integer; AValue: Boolean);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteBoolValue(ARow, ACol, AValue);
end;

procedure TExcelWriter.WriteFormula(ARow, ACol: Integer; const AFormula: string);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteFormula(ARow, ACol, AFormula);
end;

procedure TExcelWriter.SetCellFormat(ARow, ACol: Integer; const AStyle: TCellStyle);
var
  Fmt: TsCellFormat;
begin
  if not Assigned(FCurrentSheet) then Exit;
  
  // ตั้งค่า font
  with FWorkbook.GetDefaultFont do
  begin
    // ใช้ default font เป็น base
  end;
  
  // Write formats
  if AStyle.Bold then
    FCurrentSheet.WriteFontStyle(ARow, ACol, [fssBold]);
  if AStyle.FontSize > 0 then
    FCurrentSheet.WriteFontSize(ARow, ACol, AStyle.FontSize);
  if AStyle.FontColor <> 0 then
    FCurrentSheet.WriteFontColor(ARow, ACol, AStyle.FontColor);
  if AStyle.BackColor <> TsColor(-1) then
    FCurrentSheet.WriteBackgroundColor(ARow, ACol, AStyle.BackColor);
  if AStyle.HAlign <> haDefault then
    FCurrentSheet.WriteHorAlignment(ARow, ACol, AStyle.HAlign);
  if AStyle.VAlign <> vaDefault then
    FCurrentSheet.WriteVertAlignment(ARow, ACol, AStyle.VAlign);
  if AStyle.WrapText then
    FCurrentSheet.WriteWordwrap(ARow, ACol, True);
  if AStyle.NumberFormat <> '' then
    FCurrentSheet.WriteNumberFormat(ARow, ACol, nfCustom, AStyle.NumberFormat);
end;

procedure TExcelWriter.SetColumnWidth(ACol: Integer; AWidth: Double);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteColWidth(ACol, AWidth, suMillimeters);
end;

procedure TExcelWriter.SetRowHeight(ARow: Integer; AHeight: Double);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.WriteRowHeight(ARow, AHeight, suMillimeters);
end;

procedure TExcelWriter.MergeCells(ARow1, ACol1, ARow2, ACol2: Integer);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.MergeCells(ARow1, ACol1, ARow2, ACol2);
end;

procedure TExcelWriter.FreezeRow(ARow: Integer);
begin
  if Assigned(FCurrentSheet) then
    FCurrentSheet.Options := FCurrentSheet.Options + [soHasFrozenPanes];
    // FCurrentSheet.TopPaneHeight := ARow; // approximate
end;

procedure TExcelWriter.SaveToFile(const AFileName: string);
var
  Format: TsSpreadsheetFormat;
  Ext: string;
begin
  Ext := LowerCase(ExtractFileExt(AFileName));
  
  if Ext = '.xlsx' then Format := sfOOXML
  else if Ext = '.xls' then Format := sfExcel8
  else if Ext = '.ods' then Format := sfOpenDocument
  else if Ext = '.csv' then Format := sfCSV
  else Format := sfOOXML;
  
  FWorkbook.WriteToFile(AFileName, Format, True);
end;

procedure TExcelWriter.SaveToStream(AStream: TStream; AFormat: TsSpreadsheetFormat);
begin
  FWorkbook.WriteToStream(AStream, AFormat);
end;

end.
```

---

## 67.3 การอ่าน Excel

```pascal
unit excel_reader;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  fpspreadsheet,
  fpsTypes,
  fpsAllFormats;

type
  TExcelCell = record
    Row, Col: Integer;
    CellType: TCellContentType;
    StringValue: string;
    NumberValue: Double;
    DateValue: TDateTime;
    BoolValue: Boolean;
    FormulaValue: string;
  end;

  TExcelDataTable = class
  private
    FHeaders: TStringList;
    FRows: TList;  // List of TStringList
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure AddHeader(const AName: string);
    procedure AddRow(ARow: TStringList);
    
    function GetValue(ARow: Integer; const AColName: string): string;
    function GetNumValue(ARow: Integer; const AColName: string): Double;
    
    property Headers: TStringList read FHeaders;
    property RowCount: Integer read (FRows.Count);
  end;

  TExcelReader = class
  private
    FWorkbook: TsWorkbook;
    FFileName: string;
    
    function CellToString(ACell: PCell; ASheet: TsWorksheet): string;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    function LoadFile(const AFileName: string): Boolean;
    procedure Close;
    
    // ข้อมูล Workbook
    function GetSheetCount: Integer;
    function GetSheetName(AIndex: Integer): string;
    
    // อ่านข้อมูล
    function GetCell(ASheetIndex, ARow, ACol: Integer): TExcelCell;
    function GetCellString(ASheetIndex, ARow, ACol: Integer): string;
    function GetCellNumber(ASheetIndex, ARow, ACol: Integer): Double;
    
    // อ่านเป็น DataTable
    function ReadAsTable(ASheetIndex: Integer; 
      AHasHeader: Boolean = True;
      AStartRow: Integer = 0): TExcelDataTable;
    
    // อ่านทั้ง sheet เป็น StringList
    function ReadSheetAsCSV(ASheetIndex: Integer): TStringList;
    
    property Workbook: TsWorkbook read FWorkbook;
    property FileName: string read FFileName;
  end;

implementation

{ TExcelDataTable }

constructor TExcelDataTable.Create;
begin
  inherited Create;
  FHeaders := TStringList.Create;
  FRows := TList.Create;
end;

destructor TExcelDataTable.Destroy;
var
  i: Integer;
begin
  for i := 0 to FRows.Count - 1 do
    TStringList(FRows[i]).Free;
  FRows.Free;
  FHeaders.Free;
  inherited Destroy;
end;

procedure TExcelDataTable.AddHeader(const AName: string);
begin
  FHeaders.Add(AName);
end;

procedure TExcelDataTable.AddRow(ARow: TStringList);
begin
  FRows.Add(ARow);
end;

function TExcelDataTable.GetValue(ARow: Integer; const AColName: string): string;
var
  ColIdx: Integer;
  Row: TStringList;
begin
  Result := '';
  ColIdx := FHeaders.IndexOf(AColName);
  if ColIdx < 0 then Exit;
  if (ARow < 0) or (ARow >= FRows.Count) then Exit;
  
  Row := TStringList(FRows[ARow]);
  if ColIdx < Row.Count then
    Result := Row[ColIdx];
end;

function TExcelDataTable.GetNumValue(ARow: Integer; const AColName: string): Double;
begin
  Result := StrToFloatDef(GetValue(ARow, AColName), 0);
end;

{ TExcelReader }

constructor TExcelReader.Create;
begin
  inherited Create;
  FWorkbook := TsWorkbook.Create;
end;

destructor TExcelReader.Destroy;
begin
  FWorkbook.Free;
  inherited Destroy;
end;

function TExcelReader.LoadFile(const AFileName: string): Boolean;
begin
  Result := False;
  if not FileExists(AFileName) then Exit;
  
  try
    FWorkbook.ReadFromFile(AFileName);
    FFileName := AFileName;
    Result := True;
  except
    on E: Exception do
      WriteLn('Error loading Excel: ', E.Message);
  end;
end;

procedure TExcelReader.Close;
begin
  FWorkbook.Clear;
  FFileName := '';
end;

function TExcelReader.GetSheetCount: Integer;
begin
  Result := FWorkbook.GetWorksheetCount;
end;

function TExcelReader.GetSheetName(AIndex: Integer): string;
var
  Sheet: TsWorksheet;
begin
  Result := '';
  Sheet := FWorkbook.GetWorksheetByIndex(AIndex);
  if Assigned(Sheet) then
    Result := Sheet.Name;
end;

function TExcelReader.CellToString(ACell: PCell; ASheet: TsWorksheet): string;
begin
  Result := '';
  if not Assigned(ACell) then Exit;
  
  case ACell^.ContentType of
    cctUTF8String: Result := ACell^.UTF8StringValue;
    cctNumber:     Result := FormatFloat('g', ACell^.NumberValue);
    cctDateTime:   Result := FormatDateTime('yyyy-mm-dd', ACell^.DateTimeValue);
    cctBool:       if ACell^.BoolValue then Result := 'TRUE' else Result := 'FALSE';
    cctFormula:    Result := ASheet.ReadAsUTF8Text(ACell^.Row, ACell^.Col);
    else           Result := '';
  end;
end;

function TExcelReader.GetCell(ASheetIndex, ARow, ACol: Integer): TExcelCell;
var
  Sheet: TsWorksheet;
  Cell: PCell;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.Row := ARow;
  Result.Col := ACol;
  
  Sheet := FWorkbook.GetWorksheetByIndex(ASheetIndex);
  if not Assigned(Sheet) then Exit;
  
  Cell := Sheet.FindCell(ARow, ACol);
  if not Assigned(Cell) then Exit;
  
  Result.CellType := Cell^.ContentType;
  
  case Cell^.ContentType of
    cctUTF8String: Result.StringValue := Cell^.UTF8StringValue;
    cctNumber:
    begin
      Result.NumberValue := Cell^.NumberValue;
      Result.StringValue := FormatFloat('g', Cell^.NumberValue);
    end;
    cctDateTime:
    begin
      Result.DateValue := Cell^.DateTimeValue;
      Result.StringValue := FormatDateTime('yyyy-mm-dd', Cell^.DateTimeValue);
    end;
    cctBool:
    begin
      Result.BoolValue := Cell^.BoolValue;
      if Cell^.BoolValue then Result.StringValue := 'TRUE' 
      else Result.StringValue := 'FALSE';
    end;
  end;
end;

function TExcelReader.GetCellString(ASheetIndex, ARow, ACol: Integer): string;
begin
  Result := GetCell(ASheetIndex, ARow, ACol).StringValue;
end;

function TExcelReader.GetCellNumber(ASheetIndex, ARow, ACol: Integer): Double;
begin
  Result := GetCell(ASheetIndex, ARow, ACol).NumberValue;
end;

function TExcelReader.ReadAsTable(ASheetIndex: Integer; 
  AHasHeader: Boolean; AStartRow: Integer): TExcelDataTable;
var
  Sheet: TsWorksheet;
  Cell: PCell;
  Row, Col: Integer;
  RowData: TStringList;
  MaxRow, MaxCol: Integer;
begin
  Result := TExcelDataTable.Create;
  
  Sheet := FWorkbook.GetWorksheetByIndex(ASheetIndex);
  if not Assigned(Sheet) then Exit;
  
  MaxRow := Sheet.GetLastRowIndex;
  MaxCol := Sheet.GetLastColIndex;
  
  Row := AStartRow;
  
  // อ่าน header
  if AHasHeader then
  begin
    for Col := 0 to MaxCol do
    begin
      Cell := Sheet.FindCell(Row, Col);
      Result.AddHeader(CellToString(Cell, Sheet));
    end;
    Inc(Row);
  end
  else
  begin
    // สร้าง header ชั่วคราว
    for Col := 0 to MaxCol do
      Result.AddHeader('Column' + IntToStr(Col + 1));
  end;
  
  // อ่านข้อมูล
  while Row <= MaxRow do
  begin
    RowData := TStringList.Create;
    for Col := 0 to MaxCol do
    begin
      Cell := Sheet.FindCell(Row, Col);
      RowData.Add(CellToString(Cell, Sheet));
    end;
    Result.AddRow(RowData);
    Inc(Row);
  end;
end;

function TExcelReader.ReadSheetAsCSV(ASheetIndex: Integer): TStringList;
var
  Sheet: TsWorksheet;
  Cell: PCell;
  Row, Col: Integer;
  MaxRow, MaxCol: Integer;
  RowStr: string;
begin
  Result := TStringList.Create;
  
  Sheet := FWorkbook.GetWorksheetByIndex(ASheetIndex);
  if not Assigned(Sheet) then Exit;
  
  MaxRow := Sheet.GetLastRowIndex;
  MaxCol := Sheet.GetLastColIndex;
  
  for Row := 0 to MaxRow do
  begin
    RowStr := '';
    for Col := 0 to MaxCol do
    begin
      if Col > 0 then RowStr := RowStr + ',';
      Cell := Sheet.FindCell(Row, Col);
      var S := CellToString(Cell, Sheet);
      // Quote ถ้ามี comma หรือ newline
      if (Pos(',', S) > 0) or (Pos(#13, S) > 0) or (Pos(#10, S) > 0) then
        S := '"' + StringReplace(S, '"', '""', [rfReplaceAll]) + '"';
      RowStr := RowStr + S;
    end;
    Result.Add(RowStr);
  end;
end;

end.
```

---

## 67.4 ตัวอย่างสมบูรณ์: Export ข้อมูลเป็น Excel

```pascal
unit excel_export_demo;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils,
  fpspreadsheet,
  fpsTypes,
  fpsAllFormats,
  xlsxooxml,
  excel_writer;

type
  TSalesRecord = record
    Date: TDate;
    InvoiceNo: string;
    CustomerName: string;
    Product: string;
    Category: string;
    Quantity: Integer;
    UnitPrice: Double;
    Discount: Double;
    Total: Double;
    Salesperson: string;
    Region: string;
  end;

  TSalesExporter = class
  private
    FExcel: TExcelWriter;
    
    procedure CreateSummarySheet(const AData: array of TSalesRecord);
    procedure CreateDetailSheet(const AData: array of TSalesRecord);
    procedure CreatePivotSheet(const AData: array of TSalesRecord);
    procedure CreateChartDataSheet(const AData: array of TSalesRecord);
    
    procedure ApplyHeaderStyle(ASheet: TsWorksheet; ARow, AFromCol, AToCol: Integer);
    procedure ApplyDataStyle(ASheet: TsWorksheet; ARow, AFromCol, AToCol: Integer; 
      AIsEven: Boolean);
    procedure AutoFitColumns(ASheet: TsWorksheet; AMaxCol: Integer);
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Export(const AData: array of TSalesRecord; const AOutputFile: string);
  end;

implementation

uses
  fpsUtils;

{ TSalesExporter }

constructor TSalesExporter.Create;
begin
  inherited Create;
  FExcel := TExcelWriter.Create;
end;

destructor TSalesExporter.Destroy;
begin
  FExcel.Free;
  inherited Destroy;
end;

procedure TSalesExporter.ApplyHeaderStyle(ASheet: TsWorksheet; 
  ARow, AFromCol, AToCol: Integer);
var
  Col: Integer;
begin
  for Col := AFromCol to AToCol do
  begin
    ASheet.WriteFontStyle(ARow, Col, [fssBold]);
    ASheet.WriteFontSize(ARow, Col, 10);
    ASheet.WriteFontColor(ARow, Col, $00FFFFFF);
    ASheet.WriteBackgroundColor(ARow, Col, $00336699);
    ASheet.WriteHorAlignment(ARow, Col, haCenter);
    ASheet.WriteVertAlignment(ARow, Col, vaCenter);
    ASheet.WriteWordwrap(ARow, Col, True);
  end;
  ASheet.WriteRowHeight(ARow, 15, suMillimeters);
end;

procedure TSalesExporter.ApplyDataStyle(ASheet: TsWorksheet; 
  ARow, AFromCol, AToCol: Integer; AIsEven: Boolean);
var
  Col: Integer;
  BgColor: TsColor;
begin
  BgColor := $00FFFFFF;  // White
  if AIsEven then BgColor := $00F0F0F0;  // Light gray
  
  for Col := AFromCol to AToCol do
  begin
    ASheet.WriteFontSize(ARow, Col, 9);
    ASheet.WriteBackgroundColor(ARow, Col, BgColor);
    ASheet.WriteVertAlignment(ARow, Col, vaCenter);
  end;
end;

procedure TSalesExporter.CreateSummarySheet(const AData: array of TSalesRecord);
var
  Sheet: TsWorksheet;
  Col, Row: Integer;
  TotalRevenue, TotalDiscount: Double;
  MonthlyTotals: array[1..12] of Double;
  CategoryTotals: TStringList;
  i: Integer;
begin
  Sheet := FExcel.Workbook.AddWorksheet('สรุป');
  FExcel.SetActiveSheet(0);
  
  // ชื่อ Report
  Sheet.MergeCells(0, 0, 0, 7);
  Sheet.WriteUTF8Text(0, 0, 'รายงานสรุปยอดขาย');
  Sheet.WriteFontStyle(0, 0, [fssBold]);
  Sheet.WriteFontSize(0, 0, 16);
  Sheet.WriteHorAlignment(0, 0, haCenter);
  Sheet.WriteBackgroundColor(0, 0, $00003366);
  Sheet.WriteFontColor(0, 0, $00FFFFFF);
  Sheet.WriteRowHeight(0, 12, suMillimeters);
  
  // วันที่สร้าง
  Sheet.MergeCells(1, 0, 1, 7);
  Sheet.WriteUTF8Text(1, 0, 'สร้างเมื่อ: ' + FormatDateTime('dd/mm/yyyy hh:nn', Now));
  Sheet.WriteHorAlignment(1, 0, haCenter);
  Sheet.WriteFontColor(1, 0, $00666666);
  Sheet.WriteFontSize(1, 0, 9);
  
  Row := 3;
  
  // คำนวณยอดรวม
  TotalRevenue := 0;
  TotalDiscount := 0;
  FillChar(MonthlyTotals, SizeOf(MonthlyTotals), 0);
  
  for i := 0 to High(AData) do
  begin
    TotalRevenue := TotalRevenue + AData[i].Total;
    TotalDiscount := TotalDiscount + AData[i].Discount * AData[i].UnitPrice * AData[i].Quantity / 100;
    MonthlyTotals[MonthOf(AData[i].Date)] := 
      MonthlyTotals[MonthOf(AData[i].Date)] + AData[i].Total;
  end;
  
  // KPI Cards
  Sheet.WriteUTF8Text(Row, 0, 'ยอดขายรวม');
  Sheet.WriteNumber(Row, 1, TotalRevenue);
  Sheet.WriteNumberFormat(Row, 1, nfCustom, '#,##0.00');
  Sheet.WriteFontStyle(Row, 1, [fssBold]);
  Sheet.WriteFontColor(Row, 1, $00006600);
  Inc(Row);
  
  Sheet.WriteUTF8Text(Row, 0, 'ส่วนลดรวม');
  Sheet.WriteNumber(Row, 1, TotalDiscount);
  Sheet.WriteNumberFormat(Row, 1, nfCustom, '#,##0.00');
  Sheet.WriteFontColor(Row, 1, $00CC0000);
  Inc(Row);
  
  Sheet.WriteUTF8Text(Row, 0, 'ยอดขายสุทธิ');
  Sheet.WriteNumber(Row, 1, TotalRevenue - TotalDiscount);
  Sheet.WriteNumberFormat(Row, 1, nfCustom, '#,##0.00');
  Sheet.WriteFontStyle(Row, 1, [fssBold]);
  Sheet.WriteFontColor(Row, 1, $00000099);
  Inc(Row);
  
  Sheet.WriteUTF8Text(Row, 0, 'จำนวนรายการ');
  Sheet.WriteNumber(Row, 1, Length(AData));
  Inc(Row);
  
  Inc(Row);
  
  // ยอดขายรายเดือน
  Sheet.WriteUTF8Text(Row, 0, 'ยอดขายรายเดือน');
  Sheet.WriteFontStyle(Row, 0, [fssBold]);
  Sheet.WriteFontSize(Row, 0, 11);
  Inc(Row);
  
  ApplyHeaderStyle(Sheet, Row, 0, 2);
  Sheet.WriteUTF8Text(Row, 0, 'เดือน');
  Sheet.WriteUTF8Text(Row, 1, 'ยอดขาย (บาท)');
  Sheet.WriteUTF8Text(Row, 2, '% ของทั้งหมด');
  Inc(Row);
  
  const MonthNames: array[1..12] of string = (
    'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
    'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
    'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'
  );
  
  for i := 1 to 12 do
  begin
    ApplyDataStyle(Sheet, Row, 0, 2, i mod 2 = 0);
    Sheet.WriteUTF8Text(Row, 0, MonthNames[i]);
    Sheet.WriteNumber(Row, 1, MonthlyTotals[i]);
    Sheet.WriteNumberFormat(Row, 1, nfCustom, '#,##0.00');
    
    if TotalRevenue > 0 then
    begin
      Sheet.WriteNumber(Row, 2, MonthlyTotals[i] / TotalRevenue * 100);
      Sheet.WriteNumberFormat(Row, 2, nfCustom, '0.0"%"');
    end;
    
    Inc(Row);
  end;
  
  // ตั้งค่าความกว้าง columns
  Sheet.WriteColWidth(0, 35, suMillimeters);
  Sheet.WriteColWidth(1, 35, suMillimeters);
  Sheet.WriteColWidth(2, 25, suMillimeters);
end;

procedure TSalesExporter.CreateDetailSheet(const AData: array of TSalesRecord);
var
  Sheet: TsWorksheet;
  Row, i: Integer;
  Headers: array[0..10] of string;
begin
  Sheet := FExcel.Workbook.AddWorksheet('รายละเอียด');
  
  Headers[0] := 'วันที่';
  Headers[1] := 'เลขที่ใบแจ้งหนี้';
  Headers[2] := 'ลูกค้า';
  Headers[3] := 'สินค้า';
  Headers[4] := 'หมวดหมู่';
  Headers[5] := 'จำนวน';
  Headers[6] := 'ราคา/หน่วย';
  Headers[7] := 'ส่วนลด (%)';
  Headers[8] := 'รวม (บาท)';
  Headers[9] := 'พนักงาน';
  Headers[10] := 'ภูมิภาค';
  
  // Header row
  Row := 0;
  ApplyHeaderStyle(Sheet, Row, 0, 10);
  for i := 0 to 10 do
    Sheet.WriteUTF8Text(Row, i, Headers[i]);
  Inc(Row);
  
  // Data rows
  for i := 0 to High(AData) do
  begin
    ApplyDataStyle(Sheet, Row, 0, 10, i mod 2 = 0);
    
    Sheet.WriteDateTime(Row, 0, AData[i].Date, nfShortDate);
    Sheet.WriteUTF8Text(Row, 1, AData[i].InvoiceNo);
    Sheet.WriteUTF8Text(Row, 2, AData[i].CustomerName);
    Sheet.WriteUTF8Text(Row, 3, AData[i].Product);
    Sheet.WriteUTF8Text(Row, 4, AData[i].Category);
    Sheet.WriteNumber(Row, 5, AData[i].Quantity);
    Sheet.WriteNumber(Row, 6, AData[i].UnitPrice);
    Sheet.WriteNumberFormat(Row, 6, nfCustom, '#,##0.00');
    Sheet.WriteNumber(Row, 7, AData[i].Discount);
    Sheet.WriteNumberFormat(Row, 7, nfCustom, '0.0"%"');
    Sheet.WriteNumber(Row, 8, AData[i].Total);
    Sheet.WriteNumberFormat(Row, 8, nfCustom, '#,##0.00');
    Sheet.WriteUTF8Text(Row, 9, AData[i].Salesperson);
    Sheet.WriteUTF8Text(Row, 10, AData[i].Region);
    
    Inc(Row);
  end;
  
  // Total row
  ApplyHeaderStyle(Sheet, Row, 0, 10);
  Sheet.WriteUTF8Text(Row, 0, 'รวมทั้งหมด');
  Sheet.WriteUTF8Text(Row, 2, IntToStr(Length(AData)) + ' รายการ');
  
  // SUM formula
  Sheet.WriteFormula(Row, 8, 
    Format('=SUM(I2:I%d)', [Row]));
  Sheet.WriteNumberFormat(Row, 8, nfCustom, '#,##0.00');
  
  // ตั้งค่าความกว้าง
  Sheet.WriteColWidth(0, 25, suMillimeters);
  Sheet.WriteColWidth(1, 30, suMillimeters);
  Sheet.WriteColWidth(2, 40, suMillimeters);
  Sheet.WriteColWidth(3, 40, suMillimeters);
  Sheet.WriteColWidth(4, 30, suMillimeters);
  Sheet.WriteColWidth(5, 15, suMillimeters);
  Sheet.WriteColWidth(6, 25, suMillimeters);
  Sheet.WriteColWidth(7, 20, suMillimeters);
  Sheet.WriteColWidth(8, 25, suMillimeters);
  Sheet.WriteColWidth(9, 30, suMillimeters);
  Sheet.WriteColWidth(10, 25, suMillimeters);
  
  // Freeze header row
  // Sheet freeze ผ่าน options
end;

procedure TSalesExporter.CreatePivotSheet(const AData: array of TSalesRecord);
var
  Sheet: TsWorksheet;
  RegionTotals: TStringList;
  CategoryTotals: TStringList;
  i: Integer;
  Row: Integer;
  Key: string;
  Current: Double;
begin
  Sheet := FExcel.Workbook.AddWorksheet('Pivot');
  Row := 0;
  
  // หัว
  Sheet.MergeCells(0, 0, 0, 4);
  Sheet.WriteUTF8Text(0, 0, 'ตารางวิเคราะห์ยอดขาย');
  Sheet.WriteFontStyle(0, 0, [fssBold]);
  Sheet.WriteFontSize(0, 0, 14);
  Sheet.WriteHorAlignment(0, 0, haCenter);
  Row := 2;
  
  // ยอดขายตามภูมิภาค
  Sheet.WriteUTF8Text(Row, 0, 'ยอดขายตามภูมิภาค');
  Sheet.WriteFontStyle(Row, 0, [fssBold]);
  Inc(Row);
  
  ApplyHeaderStyle(Sheet, Row, 0, 2);
  Sheet.WriteUTF8Text(Row, 0, 'ภูมิภาค');
  Sheet.WriteUTF8Text(Row, 1, 'ยอดขาย');
  Sheet.WriteUTF8Text(Row, 2, 'จำนวนรายการ');
  Inc(Row);
  
  RegionTotals := TStringList.Create;
  try
    for i := 0 to High(AData) do
    begin
      Key := AData[i].Region;
      if RegionTotals.IndexOfName(Key) < 0 then
        RegionTotals.Values[Key] := '0|0'
      else
      begin
        var Parts := RegionTotals.Values[Key].Split(['|']);
        var Total := StrToFloatDef(Parts[0], 0) + AData[i].Total;
        var Count := StrToIntDef(Parts[1], 0) + 1;
        RegionTotals.Values[Key] := FormatFloat('0.##', Total) + '|' + IntToStr(Count);
      end;
    end;
    
    for i := 0 to RegionTotals.Count - 1 do
    begin
      var Parts := RegionTotals.ValueFromIndex[i].Split(['|']);
      ApplyDataStyle(Sheet, Row, 0, 2, i mod 2 = 0);
      Sheet.WriteUTF8Text(Row, 0, RegionTotals.Names[i]);
      Sheet.WriteNumber(Row, 1, StrToFloatDef(Parts[0], 0));
      Sheet.WriteNumberFormat(Row, 1, nfCustom, '#,##0.00');
      Sheet.WriteNumber(Row, 2, StrToIntDef(Parts[1], 0));
      Inc(Row);
    end;
  finally
    RegionTotals.Free;
  end;
  
  // ตั้งค่าความกว้าง
  Sheet.WriteColWidth(0, 35, suMillimeters);
  Sheet.WriteColWidth(1, 30, suMillimeters);
  Sheet.WriteColWidth(2, 30, suMillimeters);
end;

procedure TSalesExporter.CreateChartDataSheet(const AData: array of TSalesRecord);
var
  Sheet: TsWorksheet;
  i: Integer;
  MonthlyTotals: array[1..12] of Double;
begin
  Sheet := FExcel.Workbook.AddWorksheet('Chart Data');
  
  FillChar(MonthlyTotals, SizeOf(MonthlyTotals), 0);
  for i := 0 to High(AData) do
    MonthlyTotals[MonthOf(AData[i].Date)] := 
      MonthlyTotals[MonthOf(AData[i].Date)] + AData[i].Total;
  
  ApplyHeaderStyle(Sheet, 0, 0, 1);
  Sheet.WriteUTF8Text(0, 0, 'เดือน');
  Sheet.WriteUTF8Text(0, 1, 'ยอดขาย');
  
  const Months: array[1..12] of string = (
    'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
    'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'
  );
  
  for i := 1 to 12 do
  begin
    Sheet.WriteUTF8Text(i, 0, Months[i]);
    Sheet.WriteNumber(i, 1, MonthlyTotals[i]);
    Sheet.WriteNumberFormat(i, 1, nfCustom, '#,##0.00');
  end;
end;

procedure TSalesExporter.Export(const AData: array of TSalesRecord; 
  const AOutputFile: string);
begin
  WriteLn('กำลังสร้าง Excel...');
  
  CreateSummarySheet(AData);
  CreateDetailSheet(AData);
  CreatePivotSheet(AData);
  CreateChartDataSheet(AData);
  
  FExcel.SaveToFile(AOutputFile);
  
  WriteLn('บันทึก Excel สำเร็จ: ' + AOutputFile);
  WriteLn('จำนวน Sheets: ', FExcel.Workbook.GetWorksheetCount);
end;

// สร้างข้อมูลตัวอย่าง
procedure DemoExcelExport;
var
  Data: array of TSalesRecord;
  Exporter: TSalesExporter;
  Products: array[0..4] of string;
  Categories: array[0..4] of string;
  Customers: array[0..4] of string;
  Regions: array[0..3] of string;
  Salespersons: array[0..2] of string;
  i: Integer;
begin
  Products[0] := 'Laptop Pro'; Products[1] := 'Wireless Mouse';
  Products[2] := 'USB Hub'; Products[3] := 'Monitor 27"'; Products[4] := 'Keyboard';
  
  Categories[0] := 'Laptop'; Categories[1] := 'Peripheral';
  Categories[2] := 'Peripheral'; Categories[3] := 'Monitor'; Categories[4] := 'Peripheral';
  
  Customers[0] := 'บริษัท A'; Customers[1] := 'บริษัท B';
  Customers[2] := 'คุณ C'; Customers[3] := 'ห้าง D'; Customers[4] := 'โรงเรียน E';
  
  Regions[0] := 'กรุงเทพฯ'; Regions[1] := 'ภาคเหนือ';
  Regions[2] := 'ภาคใต้'; Regions[3] := 'ภาคอีสาน';
  
  Salespersons[0] := 'สมชาย'; Salespersons[1] := 'สมหญิง'; Salespersons[2] := 'วิชัย';
  
  Randomize;
  SetLength(Data, 100);
  
  for i := 0 to 99 do
  begin
    var ProdIdx := Random(5);
    var Prices: array[0..4] of Double;
    Prices[0] := 35000; Prices[1] := 990; Prices[2] := 590; Prices[3] := 12000; Prices[4] := 1500;
    
    with Data[i] do
    begin
      Date := EncodeDate(2024, 1 + Random(12), 1 + Random(28));
      InvoiceNo := Format('INV-2024-%04d', [i + 1]);
      CustomerName := Customers[Random(5)];
      Product := Products[ProdIdx];
      Category := Categories[ProdIdx];
      Quantity := 1 + Random(5);
      UnitPrice := Prices[ProdIdx];
      Discount := Random(10) * 2.0;  // 0, 2, 4, ..., 18%
      Total := UnitPrice * Quantity * (1 - Discount/100);
      Salesperson := Salespersons[Random(3)];
      Region := Regions[Random(4)];
    end;
  end;
  
  Exporter := TSalesExporter.Create;
  try
    Exporter.Export(Data, 'sales_report_2024.xlsx');
    WriteLn('ส่งออก Excel สำเร็จ!');
  finally
    Exporter.Free;
  end;
end;

end.
```

---

## 67.5 อ่าน Excel และประมวลผล

```pascal
procedure DemoReadExcel;
var
  Reader: TExcelReader;
  Table: TExcelDataTable;
  i: Integer;
  Total: Double;
begin
  Reader := TExcelReader.Create;
  try
    if Reader.LoadFile('sales_report_2024.xlsx') then
    begin
      WriteLn('Sheets: ', Reader.GetSheetCount);
      for i := 0 to Reader.GetSheetCount - 1 do
        WriteLn('  [', i, '] ', Reader.GetSheetName(i));
      
      // อ่าน sheet "รายละเอียด" (index 1)
      Table := Reader.ReadAsTable(1, True);
      try
        WriteLn('Columns: ', Table.Headers.Count);
        WriteLn('Rows: ', Table.RowCount);
        
        // คำนวณยอดรวม
        Total := 0;
        for i := 0 to Table.RowCount - 1 do
          Total := Total + Table.GetNumValue(i, 'รวม (บาท)');
          
        WriteLn('ยอดรวมจากการอ่าน: ', FormatFloat('#,##0.00', Total));
        
        // แสดง 5 แถวแรก
        WriteLn('5 แถวแรก:');
        for i := 0 to Min(4, Table.RowCount - 1) do
        begin
          WriteLn(Format('  %s | %s | %s | %s',
            [Table.GetValue(i, 'วันที่'),
             Table.GetValue(i, 'ลูกค้า'),
             Table.GetValue(i, 'สินค้า'),
             Table.GetValue(i, 'รวม (บาท)')]));
        end;
      finally
        Table.Free;
      end;
    end
    else
      WriteLn('ไม่สามารถเปิดไฟล์');
  finally
    Reader.Free;
  end;
end;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **TExcelWriter** - เขียนข้อมูลลง Excel ด้วย fpSpreadsheet
2. **TExcelReader** - อ่านข้อมูลจาก Excel
3. **Styles & Formatting** - จัดรูปแบบเซลล์, สี, font
4. **Multiple Sheets** - จัดการหลาย worksheets
5. **Formulas** - เพิ่ม Excel formulas
6. **Sales Export** - ส่งออกรายงานขายสมบูรณ์

fpSpreadsheet เป็น library ที่ทรงพลังและไม่ต้องพึ่งพา Microsoft Office ทำให้สามารถส่งออก Excel บน Windows, Linux และ macOS ได้เหมือนกัน
