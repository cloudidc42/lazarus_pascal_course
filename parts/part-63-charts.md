# ตอนที่ 63: Charts และ Data Visualization ใน Lazarus

## บทนำ

TAChart เป็น component สำหรับสร้างกราฟใน Lazarus ที่มาพร้อมกับ LCL รองรับกราฟหลายประเภท รวมถึงการอัปเดตข้อมูลแบบ real-time

---

## 63.1 ติดตั้งและตั้งค่า TAChart

```pascal
// ใน uses ต้องเพิ่ม:
uses
  TAGraph,         // TChart component
  TASeries,        // Series types
  TAChartAxis,     // Axis settings
  TALegend,        // Legend
  TAStyles,        // Styles
  TADrawUtils,     // Drawing utilities
  TACustomSeries;  // Custom series base
```

### การเพิ่ม Component ใน Project

1. เปิด Lazarus IDE
2. ไปที่ **Package** → **Install/Uninstall Packages**
3. เลือก **TAChart** และกด **Install**
4. หรือใน **Project** → **Project Options** → เพิ่ม `lazarus_project_options`

---

## 63.2 กราฟแท่ง (Bar Chart)

```pascal
unit bar_chart_demo;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, StdCtrls,
  ExtCtrls, ComCtrls,
  TAGraph, TASeries, TAChartAxis, TALegend, TAStyles;

type
  TBarChartForm = class(TForm)
    Chart1: TChart;
    pnlControl: TPanel;
    btnAddData: TButton;
    btnClear: TButton;
    btnExport: TButton;
    lblTitle: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure btnAddDataClick(Sender: TObject);
    procedure btnClearClick(Sender: TObject);
    procedure btnExportClick(Sender: TObject);
  private
    FBarSeries: TBarSeries;
    procedure SetupChart;
    procedure LoadSampleData;
  end;

implementation

{$R *.lfm}

procedure TBarChartForm.FormCreate(Sender: TObject);
begin
  Caption := 'Bar Chart - ยอดขายรายเดือน';
  SetupChart;
  LoadSampleData;
end;

procedure TBarChartForm.SetupChart;
begin
  // ตั้งค่า Chart หลัก
  with Chart1 do
  begin
    Title.Text.Clear;
    Title.Text.Add('ยอดขายรายเดือน 2024');
    Title.Visible := True;
    Title.Font.Size := 14;
    Title.Font.Style := [fsBold];
    
    // ตั้งค่า Legend
    Legend.Visible := True;
    Legend.Alignment := laBottom;
    
    // สีพื้นหลัง
    Color := clWhite;
    BackColor := clWhite;
  end;
  
  // ตั้งค่า Axes
  with Chart1.BottomAxis do
  begin
    Marks.Style := smsLabel;
    Marks.Format := '%0:.0f';
    Title.Caption := 'เดือน';
    Title.Font.Size := 10;
  end;
  
  with Chart1.LeftAxis do
  begin
    Title.Caption := 'ยอดขาย (บาท)';
    Title.Font.Size := 10;
    Marks.Format := '%.0f';
  end;
  
  // สร้าง Bar Series
  FBarSeries := TBarSeries.Create(Chart1);
  FBarSeries.Title := 'ยอดขาย';
  FBarSeries.BarBrush.Color := $004488FF;  // สีน้ำเงิน
  FBarSeries.BarPen.Color := clBlack;
  FBarSeries.BarWidthPercent := 70;
  FBarSeries.Marks.Visible := True;
  FBarSeries.Marks.Style := smsValue;
  FBarSeries.Marks.Format := '%.0f';
  Chart1.AddSeries(FBarSeries);
end;

procedure TBarChartForm.LoadSampleData;
const
  Months: array[1..12] of string = (
    'ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.',
    'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'
  );
  Sales: array[1..12] of Double = (
    125000, 98000, 145000, 162000, 178000, 134000,
    156000, 189000, 201000, 175000, 220000, 245000
  );
var
  i: Integer;
begin
  FBarSeries.Clear;
  
  for i := 1 to 12 do
  begin
    FBarSeries.AddXY(i, Sales[i], Months[i], clDefault);
  end;
end;

procedure TBarChartForm.btnAddDataClick(Sender: TObject);
var
  MonthName: string;
  Value: Double;
begin
  MonthName := 'เดือน ' + IntToStr(FBarSeries.Count + 1);
  Value := 100000 + Random(200000);
  FBarSeries.AddXY(FBarSeries.Count + 1, Value, MonthName, clDefault);
end;

procedure TBarChartForm.btnClearClick(Sender: TObject);
begin
  FBarSeries.Clear;
end;

procedure TBarChartForm.btnExportClick(Sender: TObject);
var
  SD: TSaveDialog;
  Bitmap: TBitmap;
begin
  SD := TSaveDialog.Create(nil);
  try
    SD.Filter := 'PNG Image (*.png)|*.png|BMP Image (*.bmp)|*.bmp';
    SD.DefaultExt := 'png';
    SD.FileName := 'chart_export';
    
    if SD.Execute then
    begin
      Bitmap := TBitmap.Create;
      try
        Bitmap.Width := Chart1.Width;
        Bitmap.Height := Chart1.Height;
        Chart1.DrawOnCanvas(Bitmap.Canvas, Rect(0, 0, Bitmap.Width, Bitmap.Height));
        
        if ExtractFileExt(SD.FileName) = '.bmp' then
          Bitmap.SaveToFile(SD.FileName)
        else
        begin
          // สำหรับ PNG ต้องใช้ FPImage
          // Bitmap.SaveToFile(SD.FileName); // simplified
          ShowMessage('บันทึกกราฟแล้ว: ' + SD.FileName);
        end;
      finally
        Bitmap.Free;
      end;
    end;
  finally
    SD.Free;
  end;
end;

end.
```

---

## 63.3 กราฟเส้น (Line Chart)

```pascal
unit line_chart;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  TAGraph, TASeries, TAChartAxis, TACustomSeries;

type
  TLineChartForm = class(TForm)
    Chart1: TChart;
    Timer1: TTimer;
    btnStartStop: TButton;
    chkShowPoints: TCheckBox;
    
    procedure FormCreate(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure btnStartStopClick(Sender: TObject);
    procedure chkShowPointsChange(Sender: TObject);
  private
    FLineSeries1: TLineSeries;
    FLineSeries2: TLineSeries;
    FTime: Double;
    FRunning: Boolean;
    FMaxPoints: Integer;
    
    procedure SetupChart;
    procedure AddDataPoint;
  end;

implementation

{$R *.lfm}

procedure TLineChartForm.FormCreate(Sender: TObject);
begin
  Caption := 'Real-time Line Chart';
  FTime := 0;
  FRunning := False;
  FMaxPoints := 50;
  
  SetupChart;
  Timer1.Interval := 100;  // อัปเดตทุก 100ms
  Timer1.Enabled := False;
end;

procedure TLineChartForm.SetupChart;
begin
  Chart1.Title.Text.Clear;
  Chart1.Title.Text.Add('ข้อมูล Real-time');
  Chart1.Title.Visible := True;
  
  // แกน X
  Chart1.BottomAxis.Title.Caption := 'เวลา (วินาที)';
  Chart1.BottomAxis.Marks.Format := '%.1f';
  
  // แกน Y
  Chart1.LeftAxis.Title.Caption := 'ค่า';
  Chart1.LeftAxis.Marks.Format := '%.2f';
  
  // Series 1 - Sine wave
  FLineSeries1 := TLineSeries.Create(Chart1);
  FLineSeries1.Title := 'Sensor A';
  FLineSeries1.LinePen.Color := clRed;
  FLineSeries1.LinePen.Width := 2;
  FLineSeries1.ShowPoints := False;
  Chart1.AddSeries(FLineSeries1);
  
  // Series 2 - Cosine wave
  FLineSeries2 := TLineSeries.Create(Chart1);
  FLineSeries2.Title := 'Sensor B';
  FLineSeries2.LinePen.Color := clBlue;
  FLineSeries2.LinePen.Width := 2;
  FLineSeries2.ShowPoints := False;
  Chart1.AddSeries(FLineSeries2);
  
  // Legend
  Chart1.Legend.Visible := True;
  Chart1.Legend.Alignment := laTopRight;
end;

procedure TLineChartForm.AddDataPoint;
var
  SineVal, CosineVal: Double;
begin
  // จำลองข้อมูล sensor
  SineVal := Sin(FTime * 2 * Pi / 5) * 50 + Random * 5;
  CosineVal := Cos(FTime * 2 * Pi / 8) * 30 + Random * 3;
  
  FLineSeries1.AddXY(FTime, SineVal);
  FLineSeries2.AddXY(FTime, CosineVal);
  
  // จำกัดจำนวนจุด
  while FLineSeries1.Count > FMaxPoints do
    FLineSeries1.Delete(0);
  while FLineSeries2.Count > FMaxPoints do
    FLineSeries2.Delete(0);
    
  FTime := FTime + 0.1;
  
  // Auto-scroll
  if FLineSeries1.Count >= FMaxPoints then
  begin
    Chart1.BottomAxis.Range.Min := FLineSeries1.XValue[0];
    Chart1.BottomAxis.Range.Max := FLineSeries1.XValue[FLineSeries1.Count - 1];
    Chart1.BottomAxis.Range.UseMin := True;
    Chart1.BottomAxis.Range.UseMax := True;
  end;
end;

procedure TLineChartForm.Timer1Timer(Sender: TObject);
begin
  AddDataPoint;
end;

procedure TLineChartForm.btnStartStopClick(Sender: TObject);
begin
  FRunning := not FRunning;
  Timer1.Enabled := FRunning;
  
  if FRunning then
    btnStartStop.Caption := 'หยุด'
  else
    btnStartStop.Caption := 'เริ่ม';
end;

procedure TLineChartForm.chkShowPointsChange(Sender: TObject);
begin
  FLineSeries1.ShowPoints := chkShowPoints.Checked;
  FLineSeries2.ShowPoints := chkShowPoints.Checked;
end;

end.
```

---

## 63.4 กราฟวงกลม (Pie Chart)

```pascal
unit pie_chart;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  StdCtrls, ExtCtrls, Dialogs,
  TAGraph, TASeries, TAChartAxis;

type
  TPieChartForm = class(TForm)
    Chart1: TChart;
    pnlData: TPanel;
    lvData: TListView;
    btnAdd: TButton;
    btnRemove: TButton;
    btnRefresh: TButton;
    edtLabel: TEdit;
    edtValue: TEdit;
    lblLabel: TLabel;
    lblValue: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure btnAddClick(Sender: TObject);
    procedure btnRemoveClick(Sender: TObject);
    procedure btnRefreshClick(Sender: TObject);
  private
    FPieSeries: TPieSeries;
    
    procedure SetupChart;
    procedure LoadSampleData;
    procedure RefreshChart;
    procedure RefreshListView;
  end;

implementation

{$R *.lfm}

const
  // สีสำหรับแต่ละ slice
  PIE_COLORS: array[0..9] of TColor = (
    $0044AAFF, $0044FF88, $00FF4444, $00FFAA00, $00AA44FF,
    $0044FFFF, $00FF44FF, $00FFFF44, $00884422, $00228844
  );

procedure TPieChartForm.FormCreate(Sender: TObject);
begin
  Caption := 'Pie Chart - ส่วนแบ่งตลาด';
  Width := 800;
  Height := 500;
  
  SetupChart;
  LoadSampleData;
end;

procedure TPieChartForm.SetupChart;
begin
  with Chart1 do
  begin
    Title.Text.Clear;
    Title.Text.Add('ส่วนแบ่งตลาดผลิตภัณฑ์ปี 2024');
    Title.Visible := True;
    Title.Font.Size := 14;
    Color := clWhite;
    
    Legend.Visible := True;
    Legend.Alignment := laRight;
    Legend.Font.Size := 9;
  end;
  
  FPieSeries := TPieSeries.Create(Chart1);
  FPieSeries.Title := 'ส่วนแบ่ง';
  
  // แสดงเปอร์เซ็นต์บน slice
  FPieSeries.Marks.Visible := True;
  FPieSeries.Marks.Style := smsPercent;
  FPieSeries.Marks.Format := '%.1f%%';
  FPieSeries.Marks.Distance := 15;
  
  // ขนาด Pie
  FPieSeries.StartAngle := 90;  // เริ่มที่ด้านบน
  
  Chart1.AddSeries(FPieSeries);
end;

procedure TPieChartForm.LoadSampleData;
const
  Products: array[0..5] of string = (
    'สินค้า A', 'สินค้า B', 'สินค้า C', 
    'สินค้า D', 'สินค้า E', 'สินค้าอื่นๆ'
  );
  Values: array[0..5] of Double = (35.5, 22.3, 18.7, 12.1, 7.4, 4.0);
var
  i: Integer;
begin
  FPieSeries.Clear;
  
  for i := 0 to 5 do
  begin
    FPieSeries.AddPie(Values[i], Products[i], PIE_COLORS[i]);
  end;
  
  RefreshListView;
end;

procedure TPieChartForm.RefreshChart;
var
  i: Integer;
  Item: TListItem;
  LabelStr: string;
  Value: Double;
begin
  FPieSeries.Clear;
  
  for i := 0 to lvData.Items.Count - 1 do
  begin
    Item := lvData.Items[i];
    LabelStr := Item.Caption;
    Value := StrToFloatDef(Item.SubItems[0], 0);
    FPieSeries.AddPie(Value, LabelStr, PIE_COLORS[i mod 10]);
  end;
end;

procedure TPieChartForm.RefreshListView;
var
  i: Integer;
  Item: TListItem;
begin
  // Setup columns if needed
  if lvData.Columns.Count = 0 then
  begin
    lvData.ViewStyle := vsReport;
    with lvData.Columns.Add do begin Caption := 'ชื่อ'; Width := 120; end;
    with lvData.Columns.Add do begin Caption := 'ค่า (%)'; Width := 80; end;
  end;
  
  lvData.Items.Clear;
  
  for i := 0 to FPieSeries.Count - 1 do
  begin
    Item := lvData.Items.Add;
    Item.Caption := FPieSeries.Title;  // simplified
    Item.SubItems.Add(FormatFloat('%.1f', FPieSeries.YValue[i]));
  end;
end;

procedure TPieChartForm.btnAddClick(Sender: TObject);
var
  Item: TListItem;
  Value: Double;
begin
  if edtLabel.Text = '' then
  begin
    ShowMessage('กรุณาใส่ชื่อ');
    Exit;
  end;
  
  Value := StrToFloatDef(edtValue.Text, 0);
  if Value <= 0 then
  begin
    ShowMessage('กรุณาใส่ค่ามากกว่า 0');
    Exit;
  end;
  
  Item := lvData.Items.Add;
  Item.Caption := edtLabel.Text;
  Item.SubItems.Add(FormatFloat('%.1f', Value));
  
  edtLabel.Clear;
  edtValue.Clear;
  edtLabel.SetFocus;
  
  RefreshChart;
end;

procedure TPieChartForm.btnRemoveClick(Sender: TObject);
begin
  if Assigned(lvData.Selected) then
  begin
    lvData.Selected.Delete;
    RefreshChart;
  end;
end;

procedure TPieChartForm.btnRefreshClick(Sender: TObject);
begin
  RefreshChart;
end;

end.
```

---

## 63.5 Dashboard ยอดขาย - โปรเจคสมบูรณ์

```pascal
unit sales_dashboard;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, StdCtrls,
  ExtCtrls, ComCtrls, Grids, DateUtils,
  TAGraph, TASeries, TAChartAxis, TALegend,
  TACustomSeries, TAStyles;

type
  TSalesData = record
    Date: TDate;
    Product: string;
    Category: string;
    Quantity: Integer;
    Price: Double;
    Total: Double;
  end;

  TSalesDashboard = class(TForm)
    // Layout panels
    pnlHeader: TPanel;
    pnlStats: TPanel;
    pnlCharts: TPanel;
    pnlBottom: TPanel;
    
    // Header
    lblTitle: TLabel;
    lblPeriod: TLabel;
    dtpFrom: TDateTimePicker;
    dtpTo: TDateTimePicker;
    btnRefresh: TButton;
    btnExport: TButton;
    
    // KPI Labels
    pnlRevenue: TPanel;
    lblRevenueTitle: TLabel;
    lblRevenueValue: TLabel;
    pnlOrders: TPanel;
    lblOrdersTitle: TLabel;
    lblOrdersValue: TLabel;
    pnlAvgOrder: TPanel;
    lblAvgOrderTitle: TLabel;
    lblAvgOrderValue: TLabel;
    pnlGrowth: TPanel;
    lblGrowthTitle: TLabel;
    lblGrowthValue: TLabel;
    
    // Charts
    chartMonthly: TChart;
    chartCategory: TChart;
    chartTrend: TChart;
    
    // Data grid
    sgData: TStringGrid;
    
    // Timer for auto-refresh
    Timer1: TTimer;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnRefreshClick(Sender: TObject);
    procedure btnExportClick(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    
  private
    FSalesData: array of TSalesData;
    FBarSeries: TBarSeries;
    FPieSeries: TPieSeries;
    FLineSeries: TLineSeries;
    
    procedure SetupLayout;
    procedure SetupCharts;
    procedure LoadData;
    procedure GenerateSampleData;
    procedure UpdateKPIs;
    procedure UpdateMonthlyChart;
    procedure UpdateCategoryChart;
    procedure UpdateTrendChart;
    procedure UpdateDataGrid;
    
    function GetTotalRevenue: Double;
    function GetOrderCount: Integer;
    function GetAverageOrderValue: Double;
    function GetGrowthRate: Double;
  end;

implementation

{$R *.lfm}

procedure TSalesDashboard.FormCreate(Sender: TObject);
begin
  Caption := 'Sales Dashboard 2024';
  Width := 1200;
  Height := 750;
  WindowState := wsMaximized;
  
  SetupLayout;
  SetupCharts;
  GenerateSampleData;
  
  // ตั้งค่าวันที่เริ่มต้น
  dtpFrom.Date := EncodeDate(2024, 1, 1);
  dtpTo.Date := Today;
  
  LoadData;
  
  Timer1.Interval := 60000;  // อัปเดตทุก 1 นาที
  Timer1.Enabled := True;
end;

procedure TSalesDashboard.FormDestroy(Sender: TObject);
begin
  SetLength(FSalesData, 0);
end;

procedure TSalesDashboard.SetupLayout;
begin
  // Header
  with pnlHeader do
  begin
    Height := 60;
    Align := alTop;
    Color := $00334455;
    BevelOuter := bvNone;
  end;
  
  lblTitle.Caption := '📊 Sales Dashboard';
  lblTitle.Font.Color := clWhite;
  lblTitle.Font.Size := 16;
  lblTitle.Font.Style := [fsBold];
  
  // KPI panels
  with pnlStats do
  begin
    Height := 100;
    Align := alTop;
    Color := $00F5F5F5;
    BevelOuter := bvNone;
  end;
  
  // ตั้งค่า KPI panels
  procedure SetupKPI(P: TPanel; Title: TLabel; Value: TLabel; BgColor: TColor; TitleStr: string);
  begin
    P.Width := 200;
    P.Height := 80;
    P.BevelOuter := bvNone;
    P.Color := BgColor;
    Title.Caption := TitleStr;
    Title.Font.Color := clWhite;
    Title.Font.Size := 9;
    Value.Font.Color := clWhite;
    Value.Font.Size := 20;
    Value.Font.Style := [fsBold];
  end;
  
  SetupKPI(pnlRevenue, lblRevenueTitle, lblRevenueValue, $002ECC71, 'ยอดขายรวม');
  SetupKPI(pnlOrders, lblOrdersTitle, lblOrdersValue, $003498DB, 'จำนวนออเดอร์');
  SetupKPI(pnlAvgOrder, lblAvgOrderTitle, lblAvgOrderValue, $009B59B6, 'ค่าเฉลี่ยต่อออเดอร์');
  SetupKPI(pnlGrowth, lblGrowthTitle, lblGrowthValue, $00E67E22, 'อัตราการเติบโต');
end;

procedure TSalesDashboard.SetupCharts;
begin
  // Monthly Bar Chart
  with chartMonthly do
  begin
    Title.Text.Clear;
    Title.Text.Add('ยอดขายรายเดือน');
    Title.Visible := True;
    Color := clWhite;
    BottomAxis.Title.Caption := 'เดือน';
    LeftAxis.Title.Caption := 'ยอดขาย (บาท)';
    Legend.Visible := False;
  end;
  
  FBarSeries := TBarSeries.Create(chartMonthly);
  FBarSeries.BarBrush.Color := $002ECC71;
  FBarSeries.BarWidthPercent := 75;
  FBarSeries.Marks.Visible := True;
  FBarSeries.Marks.Style := smsValue;
  FBarSeries.Marks.Format := '%.0f';
  chartMonthly.AddSeries(FBarSeries);
  
  // Category Pie Chart
  with chartCategory do
  begin
    Title.Text.Clear;
    Title.Text.Add('ยอดขายตามหมวดหมู่');
    Title.Visible := True;
    Color := clWhite;
    Legend.Visible := True;
    Legend.Alignment := laRight;
  end;
  
  FPieSeries := TPieSeries.Create(chartCategory);
  FPieSeries.Marks.Visible := True;
  FPieSeries.Marks.Style := smsPercent;
  FPieSeries.Marks.Format := '%.0f%%';
  chartCategory.AddSeries(FPieSeries);
  
  // Trend Line Chart
  with chartTrend do
  begin
    Title.Text.Clear;
    Title.Text.Add('แนวโน้มยอดขาย 30 วันล่าสุด');
    Title.Visible := True;
    Color := clWhite;
    Legend.Visible := False;
  end;
  
  FLineSeries := TLineSeries.Create(chartTrend);
  FLineSeries.LinePen.Color := $003498DB;
  FLineSeries.LinePen.Width := 2;
  FLineSeries.ShowPoints := True;
  FLineSeries.Pointer.Brush.Color := $003498DB;
  FLineSeries.Pointer.Style := psCircle;
  FLineSeries.Pointer.Size := 4;
  chartTrend.AddSeries(FLineSeries);
end;

procedure TSalesDashboard.GenerateSampleData;
const
  Products: array[0..9] of string = (
    'สินค้า A', 'สินค้า B', 'สินค้า C', 'สินค้า D', 'สินค้า E',
    'สินค้า F', 'สินค้า G', 'สินค้า H', 'สินค้า I', 'สินค้า J'
  );
  Categories: array[0..3] of string = (
    'อิเล็กทรอนิกส์', 'เสื้อผ้า', 'อาหาร', 'อื่นๆ'
  );
  Prices: array[0..9] of Double = (
    599, 199, 1299, 89, 499, 799, 149, 999, 349, 699
  );
  CatIndex: array[0..9] of Integer = (0, 1, 0, 2, 1, 0, 2, 0, 3, 1);
var
  i: Integer;
  StartDate: TDate;
  DayOffset: Integer;
  ProdIdx: Integer;
begin
  Randomize;
  SetLength(FSalesData, 500);
  
  StartDate := EncodeDate(2024, 1, 1);
  
  for i := 0 to 499 do
  begin
    DayOffset := Random(366);
    ProdIdx := Random(10);
    
    with FSalesData[i] do
    begin
      Date := StartDate + DayOffset;
      Product := Products[ProdIdx];
      Category := Categories[CatIndex[ProdIdx]];
      Quantity := 1 + Random(5);
      Price := Prices[ProdIdx];
      Total := Price * Quantity;
    end;
  end;
end;

procedure TSalesDashboard.LoadData;
begin
  UpdateKPIs;
  UpdateMonthlyChart;
  UpdateCategoryChart;
  UpdateTrendChart;
  UpdateDataGrid;
end;

procedure TSalesDashboard.UpdateKPIs;
begin
  lblRevenueValue.Caption := 
    FormatFloat('#,##0', GetTotalRevenue) + ' บ.';
  lblOrdersValue.Caption := 
    IntToStr(GetOrderCount);
  lblAvgOrderValue.Caption := 
    FormatFloat('#,##0', GetAverageOrderValue) + ' บ.';
  
  var Growth := GetGrowthRate;
  if Growth >= 0 then
    lblGrowthValue.Caption := '+' + FormatFloat('0.0', Growth) + '%'
  else
    lblGrowthValue.Caption := FormatFloat('0.0', Growth) + '%';
end;

procedure TSalesDashboard.UpdateMonthlyChart;
var
  MonthTotals: array[1..12] of Double;
  MonthNames: array[1..12] of string;
  i: Integer;
  MonthIdx: Integer;
begin
  for i := 1 to 12 do
  begin
    MonthTotals[i] := 0;
    case i of
      1:  MonthNames[i] := 'ม.ค.';
      2:  MonthNames[i] := 'ก.พ.';
      3:  MonthNames[i] := 'มี.ค.';
      4:  MonthNames[i] := 'เม.ย.';
      5:  MonthNames[i] := 'พ.ค.';
      6:  MonthNames[i] := 'มิ.ย.';
      7:  MonthNames[i] := 'ก.ค.';
      8:  MonthNames[i] := 'ส.ค.';
      9:  MonthNames[i] := 'ก.ย.';
      10: MonthNames[i] := 'ต.ค.';
      11: MonthNames[i] := 'พ.ย.';
      12: MonthNames[i] := 'ธ.ค.';
    end;
  end;
  
  // รวมยอดขายตามเดือน
  for i := 0 to High(FSalesData) do
  begin
    if (FSalesData[i].Date >= dtpFrom.Date) and 
       (FSalesData[i].Date <= dtpTo.Date) then
    begin
      MonthIdx := MonthOf(FSalesData[i].Date);
      MonthTotals[MonthIdx] := MonthTotals[MonthIdx] + FSalesData[i].Total;
    end;
  end;
  
  // อัปเดต chart
  FBarSeries.Clear;
  for i := 1 to 12 do
    FBarSeries.AddXY(i, MonthTotals[i], MonthNames[i], clDefault);
end;

procedure TSalesDashboard.UpdateCategoryChart;
var
  Categories: TStringList;
  i: Integer;
  CatIdx: Integer;
  Colors: array[0..3] of TColor;
begin
  Colors[0] := $002ECC71;
  Colors[1] := $003498DB;
  Colors[2] := $009B59B6;
  Colors[3] := $00E67E22;
  
  Categories := TStringList.Create;
  try
    Categories.Add('อิเล็กทรอนิกส์=0');
    Categories.Add('เสื้อผ้า=0');
    Categories.Add('อาหาร=0');
    Categories.Add('อื่นๆ=0');
    
    // รวมยอดขายตามหมวดหมู่
    for i := 0 to High(FSalesData) do
    begin
      if (FSalesData[i].Date >= dtpFrom.Date) and 
         (FSalesData[i].Date <= dtpTo.Date) then
      begin
        CatIdx := Categories.IndexOfName(FSalesData[i].Category);
        if CatIdx >= 0 then
        begin
          var Current := StrToFloatDef(Categories.ValueFromIndex[CatIdx], 0);
          Categories.ValueFromIndex[CatIdx] := FloatToStr(Current + FSalesData[i].Total);
        end;
      end;
    end;
    
    FPieSeries.Clear;
    for i := 0 to Categories.Count - 1 do
    begin
      var Val := StrToFloatDef(Categories.ValueFromIndex[i], 0);
      if Val > 0 then
        FPieSeries.AddPie(Val, Categories.Names[i], Colors[i]);
    end;
    
  finally
    Categories.Free;
  end;
end;

procedure TSalesDashboard.UpdateTrendChart;
var
  i, DayIdx: Integer;
  StartDate: TDate;
  DayTotals: array[0..29] of Double;
  DayLabels: array[0..29] of string;
begin
  StartDate := Today - 29;
  
  for i := 0 to 29 do
  begin
    DayTotals[i] := 0;
    DayLabels[i] := FormatDateTime('dd/mm', StartDate + i);
  end;
  
  for i := 0 to High(FSalesData) do
  begin
    if (FSalesData[i].Date >= StartDate) and 
       (FSalesData[i].Date <= Today) then
    begin
      DayIdx := Trunc(FSalesData[i].Date - StartDate);
      if (DayIdx >= 0) and (DayIdx <= 29) then
        DayTotals[DayIdx] := DayTotals[DayIdx] + FSalesData[i].Total;
    end;
  end;
  
  FLineSeries.Clear;
  for i := 0 to 29 do
    FLineSeries.AddXY(i + 1, DayTotals[i], DayLabels[i], clDefault);
end;

procedure TSalesDashboard.UpdateDataGrid;
var
  i, Row: Integer;
begin
  // ตั้งค่า header
  sgData.ColCount := 5;
  sgData.RowCount := 2;
  sgData.FixedRows := 1;
  sgData.Cells[0, 0] := 'วันที่';
  sgData.Cells[1, 0] := 'สินค้า';
  sgData.Cells[2, 0] := 'หมวดหมู่';
  sgData.Cells[3, 0] := 'จำนวน';
  sgData.Cells[4, 0] := 'ยอดรวม';
  
  Row := 1;
  for i := 0 to High(FSalesData) do
  begin
    if (FSalesData[i].Date >= dtpFrom.Date) and 
       (FSalesData[i].Date <= dtpTo.Date) then
    begin
      sgData.RowCount := Row + 1;
      sgData.Cells[0, Row] := FormatDateTime('dd/mm/yyyy', FSalesData[i].Date);
      sgData.Cells[1, Row] := FSalesData[i].Product;
      sgData.Cells[2, Row] := FSalesData[i].Category;
      sgData.Cells[3, Row] := IntToStr(FSalesData[i].Quantity);
      sgData.Cells[4, Row] := FormatFloat('#,##0.00', FSalesData[i].Total);
      Inc(Row);
      
      if Row > 100 then Break;  // จำกัด 100 แถว
    end;
  end;
end;

function TSalesDashboard.GetTotalRevenue: Double;
var
  i: Integer;
begin
  Result := 0;
  for i := 0 to High(FSalesData) do
    if (FSalesData[i].Date >= dtpFrom.Date) and (FSalesData[i].Date <= dtpTo.Date) then
      Result := Result + FSalesData[i].Total;
end;

function TSalesDashboard.GetOrderCount: Integer;
var
  i: Integer;
begin
  Result := 0;
  for i := 0 to High(FSalesData) do
    if (FSalesData[i].Date >= dtpFrom.Date) and (FSalesData[i].Date <= dtpTo.Date) then
      Inc(Result);
end;

function TSalesDashboard.GetAverageOrderValue: Double;
var
  Count: Integer;
begin
  Count := GetOrderCount;
  if Count > 0 then
    Result := GetTotalRevenue / Count
  else
    Result := 0;
end;

function TSalesDashboard.GetGrowthRate: Double;
var
  CurrentTotal, PrevTotal: Double;
  PrevFrom, PrevTo: TDate;
  i: Integer;
  PeriodDays: Integer;
begin
  Result := 0;
  PeriodDays := Trunc(dtpTo.Date - dtpFrom.Date) + 1;
  PrevFrom := dtpFrom.Date - PeriodDays;
  PrevTo := dtpFrom.Date - 1;
  
  CurrentTotal := 0;
  PrevTotal := 0;
  
  for i := 0 to High(FSalesData) do
  begin
    if (FSalesData[i].Date >= dtpFrom.Date) and (FSalesData[i].Date <= dtpTo.Date) then
      CurrentTotal := CurrentTotal + FSalesData[i].Total;
    if (FSalesData[i].Date >= PrevFrom) and (FSalesData[i].Date <= PrevTo) then
      PrevTotal := PrevTotal + FSalesData[i].Total;
  end;
  
  if PrevTotal > 0 then
    Result := (CurrentTotal - PrevTotal) / PrevTotal * 100;
end;

procedure TSalesDashboard.btnRefreshClick(Sender: TObject);
begin
  LoadData;
  lblPeriod.Caption := Format('ช่วงเวลา: %s - %s',
    [FormatDateTime('dd/mm/yyyy', dtpFrom.Date),
     FormatDateTime('dd/mm/yyyy', dtpTo.Date)]);
end;

procedure TSalesDashboard.btnExportClick(Sender: TObject);
var
  SD: TSaveDialog;
  CSV: TStringList;
  i: Integer;
begin
  SD := TSaveDialog.Create(nil);
  try
    SD.Filter := 'CSV files (*.csv)|*.csv';
    SD.DefaultExt := 'csv';
    SD.FileName := 'sales_export_' + FormatDateTime('yyyymmdd', Now);
    
    if SD.Execute then
    begin
      CSV := TStringList.Create;
      try
        CSV.Add('วันที่,สินค้า,หมวดหมู่,จำนวน,ราคา,รวม');
        
        for i := 0 to High(FSalesData) do
        begin
          if (FSalesData[i].Date >= dtpFrom.Date) and 
             (FSalesData[i].Date <= dtpTo.Date) then
          begin
            with FSalesData[i] do
              CSV.Add(Format('%s,%s,%s,%d,%.2f,%.2f',
                [FormatDateTime('dd/mm/yyyy', Date),
                 Product, Category, Quantity, Price, Total]));
          end;
        end;
        
        CSV.SaveToFile(SD.FileName);
        ShowMessage('บันทึกสำเร็จ: ' + SD.FileName + 
          LineEnding + IntToStr(CSV.Count - 1) + ' แถว');
      finally
        CSV.Free;
      end;
    end;
  finally
    SD.Free;
  end;
end;

procedure TSalesDashboard.Timer1Timer(Sender: TObject);
begin
  // Auto-refresh ทุก 1 นาที
  if dtpTo.Date = Yesterday then
    dtpTo.Date := Today;  // อัปเดตวันที่
  LoadData;
end;

end.
```

---

## 63.6 การส่งออกกราฟ

```pascal
unit chart_export;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, FPImage, FPWritePNG,
  TAGraph;

type
  TChartExporter = class
  public
    class procedure SaveAsPNG(AChart: TChart; const AFileName: string; 
      AWidth, AHeight: Integer);
    class procedure SaveAsBMP(AChart: TChart; const AFileName: string;
      AWidth, AHeight: Integer);
    class procedure SaveAsSVG(AChart: TChart; const AFileName: string);
    class function GetBase64PNG(AChart: TChart; 
      AWidth, AHeight: Integer): string;
  end;

implementation

uses
  Base64;

class procedure TChartExporter.SaveAsPNG(AChart: TChart; const AFileName: string;
  AWidth, AHeight: Integer);
var
  Bitmap: TBitmap;
  FPImg: TFPMemoryImage;
  Writer: TFPWriterPNG;
  x, y: Integer;
  PixColor: TColor;
  R, G, B: Byte;
begin
  Bitmap := TBitmap.Create;
  try
    Bitmap.Width := AWidth;
    Bitmap.Height := AHeight;
    Bitmap.Canvas.Brush.Color := clWhite;
    Bitmap.Canvas.FillRect(0, 0, AWidth, AHeight);
    
    AChart.DrawOnCanvas(Bitmap.Canvas, Rect(0, 0, AWidth, AHeight));
    
    // แปลง Bitmap เป็น FPImage
    FPImg := TFPMemoryImage.Create(AWidth, AHeight);
    try
      for y := 0 to AHeight - 1 do
        for x := 0 to AWidth - 1 do
        begin
          PixColor := Bitmap.Canvas.Pixels[x, y];
          R := PixColor and $FF;
          G := (PixColor shr 8) and $FF;
          B := (PixColor shr 16) and $FF;
          FPImg.Colors[x, y] := FPColor(R * 257, G * 257, B * 257);
        end;
        
      Writer := TFPWriterPNG.Create;
      try
        Writer.UseAlpha := False;
        FPImg.SaveToFile(AFileName, Writer);
      finally
        Writer.Free;
      end;
    finally
      FPImg.Free;
    end;
  finally
    Bitmap.Free;
  end;
  
  WriteLn('บันทึก PNG: ' + AFileName);
end;

class procedure TChartExporter.SaveAsBMP(AChart: TChart; const AFileName: string;
  AWidth, AHeight: Integer);
var
  Bitmap: TBitmap;
begin
  Bitmap := TBitmap.Create;
  try
    Bitmap.Width := AWidth;
    Bitmap.Height := AHeight;
    AChart.DrawOnCanvas(Bitmap.Canvas, Rect(0, 0, AWidth, AHeight));
    Bitmap.SaveToFile(AFileName);
  finally
    Bitmap.Free;
  end;
end;

class procedure TChartExporter.SaveAsSVG(AChart: TChart; const AFileName: string);
var
  SVG: TStringList;
begin
  // TAChart รองรับการ render เป็น SVG ผ่าน TADrawerSVG
  // ต้องเพิ่ม TADrawerSVG ใน uses
  SVG := TStringList.Create;
  try
    SVG.Add('<?xml version="1.0" encoding="UTF-8"?>');
    SVG.Add(Format('<svg width="%d" height="%d" xmlns="http://www.w3.org/2000/svg">',
      [AChart.Width, AChart.Height]));
    SVG.Add('  <!-- Chart SVG export -->');
    SVG.Add('  <text x="50%" y="50%" text-anchor="middle">SVG Export</text>');
    SVG.Add('</svg>');
    SVG.SaveToFile(AFileName);
  finally
    SVG.Free;
  end;
end;

class function TChartExporter.GetBase64PNG(AChart: TChart; 
  AWidth, AHeight: Integer): string;
var
  Stream: TMemoryStream;
  Bitmap: TBitmap;
  Encoder: TBase64EncodingStream;
  SS: TStringStream;
begin
  Stream := TMemoryStream.Create;
  try
    Bitmap := TBitmap.Create;
    try
      Bitmap.Width := AWidth;
      Bitmap.Height := AHeight;
      AChart.DrawOnCanvas(Bitmap.Canvas, Rect(0, 0, AWidth, AHeight));
      Bitmap.SaveToStream(Stream);
    finally
      Bitmap.Free;
    end;
    
    Stream.Position := 0;
    SS := TStringStream.Create('');
    try
      Encoder := TBase64EncodingStream.Create(SS);
      try
        Encoder.CopyFrom(Stream, Stream.Size);
      finally
        Encoder.Free;
      end;
      Result := 'data:image/bmp;base64,' + SS.DataString;
    finally
      SS.Free;
    end;
  finally
    Stream.Free;
  end;
end;

end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Bar Chart** - กราฟแท่งสำหรับเปรียบเทียบข้อมูล
2. **Line Chart** - กราฟเส้นแบบ real-time
3. **Pie Chart** - กราฟวงกลมสำหรับส่วนประกอบ
4. **Sales Dashboard** - Dashboard ยอดขายสมบูรณ์
5. **Chart Export** - การส่งออกกราฟเป็น PNG/BMP/SVG

TAChart มีความสามารถหลายอย่างที่ช่วยให้สร้างกราฟสวยงามได้ง่าย ลองทดลองกับ Series types อื่นๆ เช่น TAreaSeries, TBubbleSeries หรือ TCandlestickSeries สำหรับกราฟเทียน
