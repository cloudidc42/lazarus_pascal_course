# ตอนที่ 35: Graphics Programming (การเขียนโปรแกรมกราฟิก)

## บทนำ

การเขียนโปรแกรมกราฟิกใน Lazarus/Pascal ใช้ระบบ TCanvas ซึ่งเป็น abstraction layer เหนือ system graphics API ทำให้โค้ดทำงานได้บนทุกแพลตฟอร์ม (Windows, Linux, macOS) TCanvas ให้ความสามารถในการวาดเส้น, รูปทรง, ข้อความ, และจัดการรูปภาพ

## TCanvas Overview

```pascal
// TCanvas พบได้ใน:
// - TForm.Canvas           // วาดโดยตรงบน Form
// - TPaintBox.Canvas       // Control เฉพาะสำหรับการวาด
// - TBitmap.Canvas         // วาดบน Bitmap ใน memory
// - TPrinter.Canvas        // วาดสำหรับการพิมพ์
// - TImage.Canvas          // วาดบน TImage control

// การเข้าถึง Canvas
procedure TForm1.PaintBoxPaint(Sender: TObject);
begin
  // ใช้ Canvas ของ PaintBox
  with PaintBox1.Canvas do
  begin
    Pen.Color := clBlue;
    MoveTo(0, 0);
    LineTo(100, 100);
  end;
end;

// หรือวาดเมื่อ Form ต้องการ repaint
procedure TForm1.FormPaint(Sender: TObject);
begin
  Canvas.TextOut(10, 10, 'Hello, World!');
end;
```

---

## Coordinate System

```pascal
// ระบบพิกัดของ Lazarus
// (0,0) อยู่ที่มุมซ้ายบน
// X เพิ่มไปทางขวา
// Y เพิ่มลงมาล่าง

// (0,0)--------> X
//   |
//   |
//   v
//   Y

// หน่วยเป็น pixels
// ขนาดของ Canvas = ขนาดของ Control

procedure TForm1.FormPaint(Sender: TObject);
begin
  // มุมซ้ายบน
  Canvas.TextOut(0, 0, 'มุมซ้ายบน');
  
  // กลางหน้าจอ
  var CX := ClientWidth div 2;
  var CY := ClientHeight div 2;
  Canvas.TextOut(CX, CY, 'กลาง');
  
  // มุมขวาล่าง (ต้องลบขนาด text)
  var TextWidth := Canvas.TextWidth('มุมขวาล่าง');
  Canvas.TextOut(ClientWidth - TextWidth, ClientHeight - 20, 'มุมขวาล่าง');
end;
```

---

## TPen (ปากกา)

```pascal
unit PenExamples;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Graphics, Controls, ExtCtrls;

procedure DrawPenExamples(Canvas: TCanvas; Width, Height: Integer);

implementation

procedure DrawPenExamples(Canvas: TCanvas; Width, Height: Integer);
var
  Y: Integer;
begin
  Y := 20;
  
  // ─── Pen Color ───
  Canvas.Font.Size := 10;
  Canvas.TextOut(10, Y, 'Pen Colors:');
  Y := Y + 20;
  
  var Colors: array[0..5] of record Color: TColor; Name: string; end;
  Colors[0] := (Color: clRed;     Name: 'Red');
  Colors[1] := (Color: clGreen;   Name: 'Green');
  Colors[2] := (Color: clBlue;    Name: 'Blue');
  Colors[3] := (Color: clOrange;  Name: 'Orange');
  Colors[4] := (Color: clPurple;  Name: 'Purple');
  Colors[5] := (Color: clBlack;   Name: 'Black');
  
  for var i := 0 to High(Colors) do
  begin
    Canvas.Pen.Color := Colors[i].Color;
    Canvas.Pen.Width := 3;
    Canvas.MoveTo(30, Y + 7);
    Canvas.LineTo(120, Y + 7);
    Canvas.TextOut(130, Y, Colors[i].Name);
    Y := Y + 20;
  end;
  
  Y := Y + 10;
  
  // ─── Pen Width ───
  Canvas.TextOut(10, Y, 'Pen Width:');
  Y := Y + 20;
  
  Canvas.Pen.Color := clBlack;
  for var W := 1 to 8 do
  begin
    Canvas.Pen.Width := W;
    Canvas.MoveTo(30, Y);
    Canvas.LineTo(200, Y);
    Canvas.TextOut(210, Y - 5, Format('Width = %d', [W]));
    Y := Y + W + 8;
  end;
  
  Y := Y + 10;
  
  // ─── Pen Style ───
  Canvas.TextOut(10, Y, 'Pen Style:');
  Y := Y + 20;
  
  Canvas.Pen.Width := 1;
  Canvas.Pen.Color := clBlack;
  
  var Styles: array[0..5] of record Style: TPenStyle; Name: string; end;
  Styles[0] := (Style: psSolid;       Name: 'Solid');
  Styles[1] := (Style: psDash;        Name: 'Dash');
  Styles[2] := (Style: psDot;         Name: 'Dot');
  Styles[3] := (Style: psDashDot;     Name: 'DashDot');
  Styles[4] := (Style: psDashDotDot;  Name: 'DashDotDot');
  Styles[5] := (Style: psClear;       Name: 'Clear (invisible)');
  
  for var i := 0 to High(Styles) do
  begin
    Canvas.Pen.Style := Styles[i].Style;
    Canvas.MoveTo(30, Y + 7);
    Canvas.LineTo(200, Y + 7);
    Canvas.TextOut(210, Y, Styles[i].Name);
    Y := Y + 22;
  end;
  
  // ─── Pen Mode ───
  Y := Y + 10;
  Canvas.TextOut(10, Y, 'Pen Mode (XOR effect):');
  Y := Y + 20;
  
  // วาดสีพื้น
  Canvas.Brush.Color := clYellow;
  Canvas.FillRect(Rect(30, Y, 200, Y + 40));
  
  // วาดเส้นทับด้วย pmXOR
  Canvas.Pen.Mode := pmXOR;
  Canvas.Pen.Color := clBlue;
  Canvas.Pen.Width := 5;
  for var X := 30 to 200 do
  begin
    Canvas.MoveTo(X, Y);
    Canvas.LineTo(X, Y + 40);
  end;
  Canvas.Pen.Mode := pmCopy;  // รีเซ็ต
end;

end.
```

---

## TBrush (แปรง/พู่กัน)

```pascal
unit BrushExamples;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics;

procedure DrawBrushExamples(Canvas: TCanvas);

implementation

procedure DrawBrushExamples(Canvas: TCanvas);
var
  Y, X: Integer;
  R: TRect;
begin
  Y := 20;
  
  Canvas.Font.Size := 10;
  Canvas.Font.Bold := True;
  Canvas.TextOut(10, Y, 'Brush Styles:');
  Y := Y + 25;
  Canvas.Font.Bold := False;
  
  // Brush Styles
  var BrushData: array[0..7] of record 
    Style: TBrushStyle; 
    Name: string; 
  end;
  
  BrushData[0] := (Style: bsSolid;        Name: 'Solid');
  BrushData[1] := (Style: bsClear;        Name: 'Clear');
  BrushData[2] := (Style: bsHorizontal;   Name: 'Horizontal');
  BrushData[3] := (Style: bsVertical;     Name: 'Vertical');
  BrushData[4] := (Style: bsFDiagonal;    Name: 'FDiagonal');
  BrushData[5] := (Style: bsBDiagonal;    Name: 'BDiagonal');
  BrushData[6] := (Style: bsCross;        Name: 'Cross');
  BrushData[7] := (Style: bsDiagCross;    Name: 'DiagCross');
  
  X := 10;
  for var i := 0 to High(BrushData) do
  begin
    Canvas.Pen.Color := clBlack;
    Canvas.Pen.Width := 1;
    Canvas.Brush.Style := BrushData[i].Style;
    Canvas.Brush.Color := clBlue;
    
    R := Rect(X, Y, X + 60, Y + 50);
    Canvas.Rectangle(R);
    Canvas.TextOut(X, Y + 55, BrushData[i].Name);
    
    X := X + 75;
    if X > 600 then
    begin
      X := 10;
      Y := Y + 85;
    end;
  end;
  
  // Gradient Brush (manual)
  Y := Y + 100;
  Canvas.Font.Bold := True;
  Canvas.TextOut(10, Y, 'Gradient Fill (manual):');
  Y := Y + 20;
  Canvas.Font.Bold := False;
  
  // Horizontal gradient สีน้ำเงินไปแดง
  var GradWidth := 300;
  var GradHeight := 60;
  Canvas.Pen.Style := psClear;
  
  for var gx := 0 to GradWidth - 1 do
  begin
    var R_val := Round(255 * gx / GradWidth);
    var B_val := Round(255 * (1 - gx / GradWidth));
    Canvas.Brush.Color := RGB(R_val, 0, B_val);
    Canvas.FillRect(Rect(gx + 10, Y, gx + 11, Y + GradHeight));
  end;
  Canvas.Pen.Style := psSolid;
  Canvas.Pen.Color := clBlack;
  Canvas.Brush.Style := bsClear;
  Canvas.Rectangle(10, Y, GradWidth + 10, Y + GradHeight);
end;

end.
```

---

## การวาดรูปทรง

```pascal
unit ShapeDrawing;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Math;

type
  TShapeDrawer = class
  private
    FCanvas: TCanvas;
  public
    constructor Create(ACanvas: TCanvas);
    
    // Basic shapes
    procedure DrawLine(X1, Y1, X2, Y2: Integer; Color: TColor; Width: Integer = 1);
    procedure DrawRectangle(X, Y, W, H: Integer; PenColor, BrushColor: TColor);
    procedure DrawRoundRect(X, Y, W, H, RX, RY: Integer; PenColor, BrushColor: TColor);
    procedure DrawEllipse(X, Y, W, H: Integer; PenColor, BrushColor: TColor);
    procedure DrawArc(X, Y, W, H, StartAngle, SweepAngle: Integer; Color: TColor);
    procedure DrawChord(X, Y, W, H, StartAngle, SweepAngle: Integer; PenColor, BrushColor: TColor);
    procedure DrawPie(X, Y, W, H, StartAngle, SweepAngle: Integer; PenColor, BrushColor: TColor);
    
    // Advanced shapes
    procedure DrawPolygon(Points: array of TPoint; PenColor, BrushColor: TColor);
    procedure DrawRegularPolygon(CX, CY, Radius, Sides: Integer; 
                                 Angle: Double; PenColor, BrushColor: TColor);
    procedure DrawArrow(X1, Y1, X2, Y2: Integer; Color: TColor; ArrowSize: Integer = 10);
    procedure DrawStar(CX, CY, OuterRadius, InnerRadius, Points: Integer;
                      Angle: Double; PenColor, BrushColor: TColor);
    procedure DrawGrid(X, Y, W, H, CellW, CellH: Integer; 
                      PenColor: TColor; ShowLabels: Boolean = False);
  end;

implementation

constructor TShapeDrawer.Create(ACanvas: TCanvas);
begin
  FCanvas := ACanvas;
end;

procedure TShapeDrawer.DrawLine(X1, Y1, X2, Y2: Integer; Color: TColor; Width: Integer = 1);
begin
  FCanvas.Pen.Color := Color;
  FCanvas.Pen.Width := Width;
  FCanvas.MoveTo(X1, Y1);
  FCanvas.LineTo(X2, Y2);
end;

procedure TShapeDrawer.DrawRectangle(X, Y, W, H: Integer; PenColor, BrushColor: TColor);
begin
  FCanvas.Pen.Color := PenColor;
  FCanvas.Pen.Width := 1;
  FCanvas.Brush.Color := BrushColor;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.Rectangle(X, Y, X + W, Y + H);
end;

procedure TShapeDrawer.DrawRoundRect(X, Y, W, H, RX, RY: Integer; PenColor, BrushColor: TColor);
begin
  FCanvas.Pen.Color := PenColor;
  FCanvas.Brush.Color := BrushColor;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.RoundRect(X, Y, X + W, Y + H, RX, RY);
end;

procedure TShapeDrawer.DrawEllipse(X, Y, W, H: Integer; PenColor, BrushColor: TColor);
begin
  FCanvas.Pen.Color := PenColor;
  FCanvas.Pen.Width := 1;
  FCanvas.Brush.Color := BrushColor;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.Ellipse(X, Y, X + W, Y + H);
end;

procedure TShapeDrawer.DrawArc(X, Y, W, H, StartAngle, SweepAngle: Integer; Color: TColor);
begin
  // Arc ไม่มี fill
  FCanvas.Pen.Color := Color;
  FCanvas.Pen.Width := 2;
  FCanvas.Brush.Style := bsClear;
  FCanvas.Arc(X, Y, X + W, Y + H, StartAngle, SweepAngle);
end;

procedure TShapeDrawer.DrawPie(X, Y, W, H, StartAngle, SweepAngle: Integer; 
                               PenColor, BrushColor: TColor);
begin
  FCanvas.Pen.Color := PenColor;
  FCanvas.Brush.Color := BrushColor;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.Pie(X, Y, X + W, Y + H, 
              Round(Cos(StartAngle * Pi / 180) * W + X + W div 2),
              Round(-Sin(StartAngle * Pi / 180) * H + Y + H div 2),
              Round(Cos((StartAngle + SweepAngle) * Pi / 180) * W + X + W div 2),
              Round(-Sin((StartAngle + SweepAngle) * Pi / 180) * H + Y + H div 2));
end;

procedure TShapeDrawer.DrawPolygon(Points: array of TPoint; PenColor, BrushColor: TColor);
begin
  FCanvas.Pen.Color := PenColor;
  FCanvas.Brush.Color := BrushColor;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.Polygon(Points);
end;

procedure TShapeDrawer.DrawRegularPolygon(CX, CY, Radius, Sides: Integer;
                                          Angle: Double; PenColor, BrushColor: TColor);
var
  Points: array of TPoint;
  i: Integer;
  A: Double;
begin
  SetLength(Points, Sides);
  
  for i := 0 to Sides - 1 do
  begin
    A := Angle + (i * 2 * Pi / Sides);
    Points[i].X := CX + Round(Radius * Cos(A));
    Points[i].Y := CY + Round(Radius * Sin(A));
  end;
  
  DrawPolygon(Points, PenColor, BrushColor);
end;

procedure TShapeDrawer.DrawArrow(X1, Y1, X2, Y2: Integer; Color: TColor; ArrowSize: Integer = 10);
var
  Angle, SinA, CosA: Double;
  P1, P2: TPoint;
begin
  FCanvas.Pen.Color := Color;
  FCanvas.Pen.Width := 2;
  
  // วาดเส้นหลัก
  FCanvas.MoveTo(X1, Y1);
  FCanvas.LineTo(X2, Y2);
  
  // คำนวณ angle ของหัวลูกศร
  Angle := ArcTan2(Y2 - Y1, X2 - X1);
  
  SinA := Sin(Angle);
  CosA := Cos(Angle);
  
  // วาดหัวลูกศร 2 ด้าน
  P1.X := X2 - Round(ArrowSize * CosA + (ArrowSize/2) * SinA);
  P1.Y := Y2 - Round(ArrowSize * SinA - (ArrowSize/2) * CosA);
  
  P2.X := X2 - Round(ArrowSize * CosA - (ArrowSize/2) * SinA);
  P2.Y := Y2 - Round(ArrowSize * SinA + (ArrowSize/2) * CosA);
  
  FCanvas.Brush.Color := Color;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.Polygon([Point(X2, Y2), P1, P2]);
end;

procedure TShapeDrawer.DrawStar(CX, CY, OuterRadius, InnerRadius, Points: Integer;
                               Angle: Double; PenColor, BrushColor: TColor);
var
  StarPoints: array of TPoint;
  i: Integer;
  A, R: Double;
begin
  SetLength(StarPoints, Points * 2);
  
  for i := 0 to Points * 2 - 1 do
  begin
    A := Angle + (i * Pi / Points);
    R := IfThen(i mod 2 = 0, OuterRadius, InnerRadius);
    StarPoints[i].X := CX + Round(R * Cos(A - Pi/2));
    StarPoints[i].Y := CY + Round(R * Sin(A - Pi/2));
  end;
  
  FCanvas.Pen.Color := PenColor;
  FCanvas.Brush.Color := BrushColor;
  FCanvas.Brush.Style := bsSolid;
  FCanvas.Polygon(StarPoints);
end;

procedure TShapeDrawer.DrawGrid(X, Y, W, H, CellW, CellH: Integer;
                               PenColor: TColor; ShowLabels: Boolean = False);
var
  gx, gy: Integer;
begin
  FCanvas.Pen.Color := PenColor;
  FCanvas.Pen.Width := 1;
  FCanvas.Font.Size := 7;
  FCanvas.Font.Color := clGray;
  
  // แนวตั้ง
  gx := X;
  var ColNum := 0;
  while gx <= X + W do
  begin
    FCanvas.MoveTo(gx, Y);
    FCanvas.LineTo(gx, Y + H);
    if ShowLabels then
      FCanvas.TextOut(gx + 2, Y + 2, IntToStr(ColNum));
    gx := gx + CellW;
    Inc(ColNum);
  end;
  
  // แนวนอน
  gy := Y;
  var RowNum := 0;
  while gy <= Y + H do
  begin
    FCanvas.MoveTo(X, gy);
    FCanvas.LineTo(X + W, gy);
    if ShowLabels then
      FCanvas.TextOut(X + 2, gy + 2, IntToStr(RowNum));
    gy := gy + CellH;
    Inc(RowNum);
  end;
end;

end.
```

---

## การวาดข้อความ

```pascal
unit TextDrawing;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics;

procedure DrawTextExamples(Canvas: TCanvas; W, H: Integer);

implementation

procedure DrawTextExamples(Canvas: TCanvas; W, H: Integer);
var
  Y: Integer;
begin
  Y := 10;
  
  // ─── Font Properties ───
  // ขนาดและ Style
  Canvas.Font.Size := 24;
  Canvas.Font.Bold := True;
  Canvas.Font.Color := clNavy;
  Canvas.TextOut(10, Y, 'ขนาด 24 Bold');
  Y := Y + 30;
  
  Canvas.Font.Size := 16;
  Canvas.Font.Bold := False;
  Canvas.Font.Italic := True;
  Canvas.Font.Color := clDkGray;
  Canvas.TextOut(10, Y, 'ขนาด 16 Italic');
  Y := Y + 22;
  
  Canvas.Font.Italic := False;
  Canvas.Font.Underline := True;
  Canvas.Font.Size := 14;
  Canvas.Font.Color := clBlue;
  Canvas.TextOut(10, Y, 'Underline Text');
  Y := Y + 20;
  
  Canvas.Font.Underline := False;
  Canvas.Font.StrikeThrough := True;
  Canvas.Font.Color := clRed;
  Canvas.TextOut(10, Y, 'Strike Through');
  Canvas.Font.StrikeThrough := False;
  Y := Y + 20;
  
  // ─── Font Name ───
  Y := Y + 10;
  Canvas.Font.Size := 12;
  Canvas.Font.Color := clBlack;
  
  var Fonts: array[0..3] of string = ('Tahoma', 'Arial', 'Times New Roman', 'Courier New');
  for var Font in Fonts do
  begin
    Canvas.Font.Name := Font;
    Canvas.TextOut(10, Y, Font + ': กขคงจ ABCDE 12345');
    Y := Y + 18;
  end;
  
  // ─── TextRect (clip within rect) ───
  Y := Y + 10;
  Canvas.Font.Name := 'Tahoma';
  Canvas.Font.Size := 10;
  Canvas.Font.Color := clBlack;
  
  var R := Rect(10, Y, 200, Y + 50);
  Canvas.Pen.Color := clGray;
  Canvas.Brush.Style := bsClear;
  Canvas.Rectangle(R);
  
  var LongText := 'ข้อความยาวที่จะถูกตัดออกเมื่อเกินขอบเขตของ Rectangle';
  Canvas.TextRect(R, 12, Y + 2, LongText);
  Y := Y + 60;
  
  // ─── TextOut Alignment ───
  Y := Y + 10;
  var BoxW := 300;
  
  // Left align (default)
  Canvas.Pen.Color := clLtGray;
  Canvas.Brush.Style := bsClear;
  Canvas.Rectangle(10, Y, 10 + BoxW, Y + 20);
  Canvas.TextOut(12, Y + 2, 'ชิดซ้าย (default)');
  Y := Y + 25;
  
  // Center align
  Canvas.Rectangle(10, Y, 10 + BoxW, Y + 20);
  var Text := 'กึ่งกลาง';
  var TextW := Canvas.TextWidth(Text);
  Canvas.TextOut(10 + (BoxW - TextW) div 2, Y + 2, Text);
  Y := Y + 25;
  
  // Right align
  Canvas.Rectangle(10, Y, 10 + BoxW, Y + 20);
  Text := 'ชิดขวา';
  TextW := Canvas.TextWidth(Text);
  Canvas.TextOut(10 + BoxW - TextW - 2, Y + 2, Text);
  Y := Y + 30;
  
  // ─── Rotated Text (ต้องใช้ Font.Orientation) ───
  Y := Y + 20;
  Canvas.TextOut(10, Y, 'Rotated Text:');
  
  Canvas.Font.Size := 12;
  Canvas.Font.Color := clGreen;
  Canvas.Font.Orientation := 900;  // 90 degrees (หน่วยเป็น 1/10 degree)
  Canvas.TextOut(50, Y + 100, 'หมุน 90°');
  
  Canvas.Font.Orientation := 450;  // 45 degrees
  Canvas.TextOut(100, Y + 100, 'หมุน 45°');
  
  Canvas.Font.Orientation := 0;    // Reset
  
  // ─── Text Size Calculation ───
  Y := Y + 120;
  Canvas.Font.Size := 12;
  Canvas.Font.Color := clBlack;
  Canvas.Font.Orientation := 0;
  
  var SampleText := 'วัดขนาดข้อความ';
  var TW := Canvas.TextWidth(SampleText);
  var TH := Canvas.TextHeight(SampleText);
  
  Canvas.TextOut(10, Y, Format('"%s" width=%d height=%d', [SampleText, TW, TH]));
  
  // วาดกรอบรอบข้อความ
  Canvas.Pen.Color := clRed;
  Canvas.Brush.Style := bsClear;
  Canvas.Rectangle(10, Y, 10 + TW, Y + TH);
end;

end.
```

---

## Bitmaps: TBitmap

```pascal
unit BitmapOperations;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, GraphType, IntfGraphics;

type
  TBitmapHelper = class
  public
    // โหลดและบันทึก
    class function LoadBitmap(const FileName: string): TBitmap;
    class procedure SaveBitmap(Bitmap: TBitmap; const FileName: string);
    
    // การแก้ไขพื้นฐาน
    class function ResizeBitmap(Source: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
    class function CropBitmap(Source: TBitmap; CropRect: TRect): TBitmap;
    class function RotateBitmap(Source: TBitmap; Angle: Double): TBitmap;
    class function FlipHorizontal(Source: TBitmap): TBitmap;
    class function FlipVertical(Source: TBitmap): TBitmap;
    
    // Color operations
    class procedure GrayscaleBitmap(Bitmap: TBitmap);
    class procedure BrightnessAdjust(Bitmap: TBitmap; Delta: Integer);
    class procedure ContrastAdjust(Bitmap: TBitmap; Factor: Double);
    class procedure ApplySepia(Bitmap: TBitmap);
    class procedure InvertColors(Bitmap: TBitmap);
    
    // Effects
    class function BlurBitmap(Source: TBitmap; Radius: Integer): TBitmap;
    class procedure DrawWatermark(Bitmap: TBitmap; const Text: string);
    class function CreateThumbnail(Source: TBitmap; MaxSize: Integer): TBitmap;
  end;

implementation

class function TBitmapHelper.LoadBitmap(const FileName: string): TBitmap;
var
  Ext: string;
begin
  Result := TBitmap.Create;
  Ext := LowerCase(ExtractFileExt(FileName));
  
  if Ext = '.bmp' then
    Result.LoadFromFile(FileName)
  else if (Ext = '.jpg') or (Ext = '.jpeg') then
  begin
    var JPEG := TJPEGImage.Create;
    try
      JPEG.LoadFromFile(FileName);
      Result.Assign(JPEG);
    finally
      JPEG.Free;
    end;
  end
  else if Ext = '.png' then
  begin
    var PNG := TPortableNetworkGraphic.Create;
    try
      PNG.LoadFromFile(FileName);
      Result.Assign(PNG);
    finally
      PNG.Free;
    end;
  end
  else
  begin
    Result.Free;
    raise Exception.Create('ไม่รองรับรูปแบบไฟล์: ' + Ext);
  end;
end;

class procedure TBitmapHelper.SaveBitmap(Bitmap: TBitmap; const FileName: string);
var
  Ext: string;
begin
  Ext := LowerCase(ExtractFileExt(FileName));
  
  if Ext = '.bmp' then
    Bitmap.SaveToFile(FileName)
  else if (Ext = '.jpg') or (Ext = '.jpeg') then
  begin
    var JPEG := TJPEGImage.Create;
    try
      JPEG.Assign(Bitmap);
      JPEG.CompressionQuality := 90;
      JPEG.SaveToFile(FileName);
    finally
      JPEG.Free;
    end;
  end
  else if Ext = '.png' then
  begin
    var PNG := TPortableNetworkGraphic.Create;
    try
      PNG.Assign(Bitmap);
      PNG.SaveToFile(FileName);
    finally
      PNG.Free;
    end;
  end
  else
    raise Exception.Create('ไม่รองรับรูปแบบไฟล์: ' + Ext);
end;

class function TBitmapHelper.ResizeBitmap(Source: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
begin
  Result := TBitmap.Create;
  Result.Width := NewWidth;
  Result.Height := NewHeight;
  Result.PixelFormat := Source.PixelFormat;
  
  // Stretch copy
  Result.Canvas.StretchDraw(Rect(0, 0, NewWidth, NewHeight), Source);
end;

class function TBitmapHelper.CropBitmap(Source: TBitmap; CropRect: TRect): TBitmap;
begin
  Result := TBitmap.Create;
  Result.Width := CropRect.Width;
  Result.Height := CropRect.Height;
  
  Result.Canvas.CopyRect(Rect(0, 0, CropRect.Width, CropRect.Height),
                         Source.Canvas, CropRect);
end;

class procedure TBitmapHelper.GrayscaleBitmap(Bitmap: TBitmap);
var
  x, y: Integer;
  Color: TColor;
  R, G, B, Gray: Byte;
begin
  // ต้องกำหนด PixelFormat ก่อน
  Bitmap.PixelFormat := pf32bit;
  
  for y := 0 to Bitmap.Height - 1 do
    for x := 0 to Bitmap.Width - 1 do
    begin
      Color := Bitmap.Canvas.Pixels[x, y];
      R := GetRValue(Color);
      G := GetGValue(Color);
      B := GetBValue(Color);
      
      // Luminance formula
      Gray := Round(0.299 * R + 0.587 * G + 0.114 * B);
      
      Bitmap.Canvas.Pixels[x, y] := RGB(Gray, Gray, Gray);
    end;
end;

class procedure TBitmapHelper.BrightnessAdjust(Bitmap: TBitmap; Delta: Integer);
var
  x, y: Integer;
  Color: TColor;
  R, G, B: Integer;
begin
  Bitmap.PixelFormat := pf32bit;
  
  for y := 0 to Bitmap.Height - 1 do
    for x := 0 to Bitmap.Width - 1 do
    begin
      Color := Bitmap.Canvas.Pixels[x, y];
      
      R := Clamp(GetRValue(Color) + Delta, 0, 255);
      G := Clamp(GetGValue(Color) + Delta, 0, 255);
      B := Clamp(GetBValue(Color) + Delta, 0, 255);
      
      Bitmap.Canvas.Pixels[x, y] := RGB(R, G, B);
    end;
end;

class procedure TBitmapHelper.ApplySepia(Bitmap: TBitmap);
var
  x, y: Integer;
  Color: TColor;
  R, G, B: Byte;
  NR, NG, NB: Integer;
begin
  Bitmap.PixelFormat := pf32bit;
  
  for y := 0 to Bitmap.Height - 1 do
    for x := 0 to Bitmap.Width - 1 do
    begin
      Color := Bitmap.Canvas.Pixels[x, y];
      R := GetRValue(Color);
      G := GetGValue(Color);
      B := GetBValue(Color);
      
      NR := Clamp(Round(R * 0.393 + G * 0.769 + B * 0.189), 0, 255);
      NG := Clamp(Round(R * 0.349 + G * 0.686 + B * 0.168), 0, 255);
      NB := Clamp(Round(R * 0.272 + G * 0.534 + B * 0.131), 0, 255);
      
      Bitmap.Canvas.Pixels[x, y] := RGB(NR, NG, NB);
    end;
end;

class procedure TBitmapHelper.InvertColors(Bitmap: TBitmap);
var
  x, y: Integer;
  Color: TColor;
begin
  for y := 0 to Bitmap.Height - 1 do
    for x := 0 to Bitmap.Width - 1 do
    begin
      Color := Bitmap.Canvas.Pixels[x, y];
      Bitmap.Canvas.Pixels[x, y] := RGB(
        255 - GetRValue(Color),
        255 - GetGValue(Color),
        255 - GetBValue(Color));
    end;
end;

class function TBitmapHelper.CreateThumbnail(Source: TBitmap; MaxSize: Integer): TBitmap;
var
  ScaleX, ScaleY, Scale: Double;
  NewW, NewH: Integer;
begin
  ScaleX := MaxSize / Source.Width;
  ScaleY := MaxSize / Source.Height;
  Scale := Min(ScaleX, ScaleY);
  
  NewW := Round(Source.Width * Scale);
  NewH := Round(Source.Height * Scale);
  
  Result := ResizeBitmap(Source, NewW, NewH);
end;

class procedure TBitmapHelper.DrawWatermark(Bitmap: TBitmap; const Text: string);
begin
  Bitmap.Canvas.Font.Size := Max(Bitmap.Width div 20, 12);
  Bitmap.Canvas.Font.Color := RGB(200, 200, 200);  // สีเทาอ่อน
  Bitmap.Canvas.Font.Style := [fsBold];
  Bitmap.Canvas.Brush.Style := bsClear;
  
  var TW := Bitmap.Canvas.TextWidth(Text);
  var TH := Bitmap.Canvas.TextHeight(Text);
  var X := (Bitmap.Width - TW) div 2;
  var Y := (Bitmap.Height - TH) div 2;
  
  // Draw shadow
  Bitmap.Canvas.Font.Color := RGB(0, 0, 0);
  Bitmap.Canvas.TextOut(X + 2, Y + 2, Text);
  
  // Draw text
  Bitmap.Canvas.Font.Color := RGB(200, 200, 200);
  Bitmap.Canvas.TextOut(X, Y, Text);
end;

end.
```

---

## Double Buffering และ Off-screen Rendering

```pascal
unit DoubleBuffering;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls;

type
  // Form ที่ใช้ Double Buffering
  TDoubleBufferedForm = class(TForm)
  private
    FBuffer: TBitmap;
    FNeedRedraw: Boolean;
    
    procedure UpdateBuffer;
    procedure DrawScene(Canvas: TCanvas; W, H: Integer);
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure Paint; override;
    procedure Invalidate; override;
    procedure Resize; override;
  end;
  
  // Animation ด้วย Double Buffering
  TAnimatedPaintBox = class(TPaintBox)
  private
    FBuffer: TBitmap;
    FAnimTimer: TTimer;
    FFrame: Integer;
    
    procedure OnTimer(Sender: TObject);
    procedure DrawFrame;
    
  protected
    procedure Paint; override;
    procedure Resize; override;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure StartAnimation;
    procedure StopAnimation;
  end;

implementation

// ─── TDoubleBufferedForm ───

constructor TDoubleBufferedForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  FBuffer := TBitmap.Create;
  FNeedRedraw := True;
  
  // เปิด DoubleBuffered ของ Form
  DoubleBuffered := True;
end;

destructor TDoubleBufferedForm.Destroy;
begin
  FBuffer.Free;
  inherited Destroy;
end;

procedure TDoubleBufferedForm.Resize;
begin
  inherited Resize;
  FBuffer.Width := ClientWidth;
  FBuffer.Height := ClientHeight;
  FNeedRedraw := True;
  Invalidate;
end;

procedure TDoubleBufferedForm.UpdateBuffer;
begin
  if FNeedRedraw then
  begin
    FBuffer.Width := ClientWidth;
    FBuffer.Height := ClientHeight;
    
    // วาดลงใน buffer
    FBuffer.Canvas.Brush.Color := Color;
    FBuffer.Canvas.FillRect(Rect(0, 0, ClientWidth, ClientHeight));
    
    DrawScene(FBuffer.Canvas, ClientWidth, ClientHeight);
    
    FNeedRedraw := False;
  end;
end;

procedure TDoubleBufferedForm.Paint;
begin
  UpdateBuffer;
  // คัดลอก buffer มาแสดง
  Canvas.Draw(0, 0, FBuffer);
end;

procedure TDoubleBufferedForm.Invalidate;
begin
  FNeedRedraw := True;
  inherited Invalidate;
end;

procedure TDoubleBufferedForm.DrawScene(Canvas: TCanvas; W, H: Integer);
begin
  // Override ในคลาสลูก
  Canvas.TextOut(10, 10, 'Double Buffered Drawing');
end;

// ─── TAnimatedPaintBox ───

constructor TAnimatedPaintBox.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  FBuffer := TBitmap.Create;
  FFrame := 0;
  
  FAnimTimer := TTimer.Create(Self);
  FAnimTimer.Interval := 16;  // ~60 FPS
  FAnimTimer.Enabled := False;
  FAnimTimer.OnTimer := @OnTimer;
end;

destructor TAnimatedPaintBox.Destroy;
begin
  FAnimTimer.Free;
  FBuffer.Free;
  inherited Destroy;
end;

procedure TAnimatedPaintBox.Resize;
begin
  inherited Resize;
  FBuffer.Width := Width;
  FBuffer.Height := Height;
end;

procedure TAnimatedPaintBox.StartAnimation;
begin
  FAnimTimer.Enabled := True;
end;

procedure TAnimatedPaintBox.StopAnimation;
begin
  FAnimTimer.Enabled := False;
end;

procedure TAnimatedPaintBox.OnTimer(Sender: TObject);
begin
  Inc(FFrame);
  DrawFrame;
  Repaint;
end;

procedure TAnimatedPaintBox.DrawFrame;
var
  W, H: Integer;
  CX, CY: Integer;
  Angle: Double;
  X, Y: Integer;
begin
  W := FBuffer.Width;
  H := FBuffer.Height;
  CX := W div 2;
  CY := H div 2;
  
  // ล้าง buffer
  FBuffer.Canvas.Brush.Color := $1A1A2E;  // Dark background
  FBuffer.Canvas.FillRect(Rect(0, 0, W, H));
  
  Angle := FFrame * 0.02;
  
  // วาดดาวหมุน
  var NumStars := 50;
  for var i := 1 to NumStars do
  begin
    var StarAngle := Angle + (i * 2 * Pi / NumStars);
    var Radius := 50 + i * 3;
    
    X := CX + Round(Radius * Cos(StarAngle));
    Y := CY + Round(Radius * Sin(StarAngle));
    
    var StarSize := 2 + i div 10;
    var Brightness := Round(128 + 127 * Sin(Angle * 3 + i * 0.5));
    
    FBuffer.Canvas.Brush.Color := RGB(Brightness, Round(Brightness * 0.7), 0);
    FBuffer.Canvas.Pen.Style := psClear;
    FBuffer.Canvas.Ellipse(X - StarSize, Y - StarSize, X + StarSize, Y + StarSize);
  end;
  
  // วาดวงกลมหมุน
  FBuffer.Canvas.Pen.Style := psSolid;
  for var ring := 1 to 5 do
  begin
    var RingAngle := Angle * ring * 0.5;
    FBuffer.Canvas.Pen.Color := RGB(
      Round(128 + 127 * Sin(RingAngle)),
      Round(128 + 127 * Cos(RingAngle)),
      255);
    FBuffer.Canvas.Pen.Width := 2;
    FBuffer.Canvas.Brush.Style := bsClear;
    FBuffer.Canvas.Ellipse(
      CX - ring * 40, CY - ring * 30,
      CX + ring * 40, CY + ring * 30);
  end;
  
  // FPS counter
  FBuffer.Canvas.Font.Size := 10;
  FBuffer.Canvas.Font.Color := clWhite;
  FBuffer.Canvas.Brush.Style := bsClear;
  FBuffer.Canvas.TextOut(5, 5, Format('Frame: %d', [FFrame]));
end;

procedure TAnimatedPaintBox.Paint;
begin
  if FBuffer.Width <> Width then
    FBuffer.Width := Width;
  if FBuffer.Height <> Height then
    FBuffer.Height := Height;
  
  Canvas.Draw(0, 0, FBuffer);
end;

end.
```

---

## Alpha Blending และ Transparency

```pascal
unit AlphaBlending;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, IntfGraphics, FPImage;

type
  TAlphaBlend = class
  public
    class procedure DrawTransparentRect(Canvas: TCanvas; 
                                        R: TRect; Color: TColor; 
                                        Alpha: Byte);
    class procedure BlendBitmaps(Dest, Src: TBitmap; 
                                 Alpha: Byte; X, Y: Integer);
    class procedure DrawGlassEffect(Canvas: TCanvas; R: TRect; 
                                   Color: TColor);
    class procedure CreateDropShadow(Bitmap: TBitmap; 
                                    OffsetX, OffsetY, Blur: Integer;
                                    ShadowColor: TColor);
  end;

implementation

class procedure TAlphaBlend.DrawTransparentRect(Canvas: TCanvas;
                                                R: TRect; Color: TColor;
                                                Alpha: Byte);
var
  TempBitmap: TBitmap;
begin
  // สร้าง bitmap ชั่วคราว
  TempBitmap := TBitmap.Create;
  try
    TempBitmap.Width := R.Width;
    TempBitmap.Height := R.Height;
    TempBitmap.PixelFormat := pf32bit;
    
    // วาดสีลงใน bitmap
    TempBitmap.Canvas.Brush.Color := Color;
    TempBitmap.Canvas.FillRect(Rect(0, 0, R.Width, R.Height));
    
    // Blend onto destination
    Canvas.CopyMode := cmSrcAnd;
    Canvas.Draw(R.Left, R.Top, TempBitmap);
    
    // Method ที่ดีกว่าใช้ pixel-by-pixel
  finally
    TempBitmap.Free;
  end;
end;

class procedure TAlphaBlend.BlendBitmaps(Dest, Src: TBitmap;
                                         Alpha: Byte; X, Y: Integer);
var
  dx, dy: Integer;
  SrcColor, DstColor: TColor;
  SR, SG, SB, DR, DG, DB: Byte;
  A, InvA: Integer;
begin
  A := Alpha;
  InvA := 255 - A;
  
  for dy := 0 to Src.Height - 1 do
    for dx := 0 to Src.Width - 1 do
    begin
      if (X + dx < Dest.Width) and (Y + dy < Dest.Height) then
      begin
        SrcColor := Src.Canvas.Pixels[dx, dy];
        DstColor := Dest.Canvas.Pixels[X + dx, Y + dy];
        
        SR := GetRValue(SrcColor);
        SG := GetGValue(SrcColor);
        SB := GetBValue(SrcColor);
        
        DR := GetRValue(DstColor);
        DG := GetGValue(DstColor);
        DB := GetBValue(DstColor);
        
        // Alpha blending formula: Result = Src * Alpha + Dst * (1 - Alpha)
        Dest.Canvas.Pixels[X + dx, Y + dy] := RGB(
          (SR * A + DR * InvA) div 255,
          (SG * A + DG * InvA) div 255,
          (SB * A + DB * InvA) div 255);
      end;
    end;
end;

class procedure TAlphaBlend.DrawGlassEffect(Canvas: TCanvas; R: TRect; 
                                            Color: TColor);
var
  i: Integer;
begin
  // Glass effect ด้วย gradient
  var H := R.Height;
  var W := R.Width;
  
  Canvas.Pen.Style := psClear;
  
  for i := 0 to H div 2 - 1 do
  begin
    var Alpha := Round(80 - (i * 60 / (H div 2)));
    var LR := GetRValue(Color);
    var LG := GetGValue(Color);
    var LB := GetBValue(Color);
    
    Canvas.Brush.Color := RGB(
      LR + (255 - LR) * Alpha div 255,
      LG + (255 - LG) * Alpha div 255,
      LB + (255 - LB) * Alpha div 255);
    Canvas.FillRect(Rect(R.Left, R.Top + i, R.Right, R.Top + i + 1));
  end;
  
  // Bottom half
  for i := H div 2 to H - 1 do
  begin
    var Alpha := Round(20 + ((i - H div 2) * 30 / (H div 2)));
    Canvas.Brush.Color := RGB(
      GetRValue(Color) * (255 - Alpha) div 255,
      GetGValue(Color) * (255 - Alpha) div 255,
      GetBValue(Color) * (255 - Alpha) div 255);
    Canvas.FillRect(Rect(R.Left, R.Top + i, R.Right, R.Top + i + 1));
  end;
  
  // Border
  Canvas.Pen.Style := psSolid;
  Canvas.Pen.Color := RGB(
    Min(255, GetRValue(Color) + 50),
    Min(255, GetGValue(Color) + 50),
    Min(255, GetBValue(Color) + 50));
  Canvas.Pen.Width := 1;
  Canvas.Brush.Style := bsClear;
  Canvas.RoundRect(R.Left, R.Top, R.Right, R.Bottom, 8, 8);
end;

end.
```

---

## ตัวอย่าง Paint Application สมบูรณ์

```pascal
unit PaintApp;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls, Buttons, ColorBox, Menus;

type
  TToolType = (ttPen, ttLine, ttRectangle, ttEllipse, ttFill, 
               ttEraser, ttText, ttSelection);
  
  THistoryEntry = class
    Bitmap: TBitmap;
    constructor Create(Source: TBitmap);
    destructor Destroy; override;
  end;

  TfrmPaint = class(TForm)
    // Toolbar
    ToolBar1: TToolBar;
    btnPen: TSpeedButton;
    btnLine: TSpeedButton;
    btnRect: TSpeedButton;
    btnEllipse: TSpeedButton;
    btnFill: TSpeedButton;
    btnEraser: TSpeedButton;
    btnText: TSpeedButton;
    
    // Color
    clbForeColor: TColorButton;
    clbBackColor: TColorButton;
    
    // Size
    trkPenSize: TTrackBar;
    
    // Canvas
    PaintBox1: TPaintBox;
    sbHoriz: TScrollBar;
    sbVert: TScrollBar;
    
    // Status
    StatusBar1: TStatusBar;
    
    // Menu
    MainMenu1: TMainMenu;
    mnuFile: TMenuItem;
    mnuNew: TMenuItem;
    mnuOpen: TMenuItem;
    mnuSave: TMenuItem;
    mnuSaveAs: TMenuItem;
    mnuEdit: TMenuItem;
    mnuUndo: TMenuItem;
    mnuRedo: TMenuItem;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure PaintBoxPaint(Sender: TObject);
    procedure PaintBoxMouseDown(Sender: TObject; Button: TMouseButton; 
                               Shift: TShiftState; X, Y: Integer);
    procedure PaintBoxMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
    procedure PaintBoxMouseUp(Sender: TObject; Button: TMouseButton; 
                             Shift: TShiftState; X, Y: Integer);
    procedure ToolButtonClick(Sender: TObject);
    procedure mnuNewClick(Sender: TObject);
    procedure mnuOpenClick(Sender: TObject);
    procedure mnuSaveClick(Sender: TObject);
    procedure mnuUndoClick(Sender: TObject);
    procedure mnuRedoClick(Sender: TObject);
    
  private
    FCanvas: TBitmap;
    FTempCanvas: TBitmap;
    FCurrentTool: TToolType;
    FDrawing: Boolean;
    FStartX, FStartY: Integer;
    FLastX, FLastY: Integer;
    
    FUndoHistory: TList;
    FRedoHistory: TList;
    
    FCurrentFile: string;
    
    procedure InitCanvas(W, H: Integer);
    procedure SaveHistory;
    procedure Undo;
    procedure Redo;
    
    procedure DrawWithPen(X, Y: Integer);
    procedure DrawLine(X1, Y1, X2, Y2: Integer; Preview: Boolean = False);
    procedure DrawRect(X1, Y1, X2, Y2: Integer; Preview: Boolean = False);
    procedure DrawEllipse(X1, Y1, X2, Y2: Integer; Preview: Boolean = False);
    procedure FloodFill(X, Y: Integer; Color: TColor);
    procedure EraseAt(X, Y: Integer);
    
    procedure UpdateStatus(X, Y: Integer);
  end;

var
  frmPaint: TfrmPaint;

implementation

{$R *.lfm}

constructor THistoryEntry.Create(Source: TBitmap);
begin
  Bitmap := TBitmap.Create;
  Bitmap.Assign(Source);
end;

destructor THistoryEntry.Destroy;
begin
  Bitmap.Free;
  inherited;
end;

procedure TfrmPaint.FormCreate(Sender: TObject);
begin
  FUndoHistory := TList.Create;
  FRedoHistory := TList.Create;
  FCurrentTool := ttPen;
  FDrawing := False;
  
  InitCanvas(800, 600);
  
  // Setup paint box
  PaintBox1.OnPaint := @PaintBoxPaint;
  PaintBox1.OnMouseDown := @PaintBoxMouseDown;
  PaintBox1.OnMouseMove := @PaintBoxMouseMove;
  PaintBox1.OnMouseUp := @PaintBoxMouseUp;
end;

procedure TfrmPaint.FormDestroy(Sender: TObject);
var
  i: Integer;
begin
  for i := 0 to FUndoHistory.Count - 1 do
    THistoryEntry(FUndoHistory[i]).Free;
  FUndoHistory.Free;
  
  for i := 0 to FRedoHistory.Count - 1 do
    THistoryEntry(FRedoHistory[i]).Free;
  FRedoHistory.Free;
  
  FCanvas.Free;
  FTempCanvas.Free;
end;

procedure TfrmPaint.InitCanvas(W, H: Integer);
begin
  if Assigned(FCanvas) then FCanvas.Free;
  if Assigned(FTempCanvas) then FTempCanvas.Free;
  
  FCanvas := TBitmap.Create;
  FCanvas.Width := W;
  FCanvas.Height := H;
  FCanvas.Canvas.Brush.Color := clWhite;
  FCanvas.Canvas.FillRect(Rect(0, 0, W, H));
  
  FTempCanvas := TBitmap.Create;
  FTempCanvas.Width := W;
  FTempCanvas.Height := H;
end;

procedure TfrmPaint.SaveHistory;
var
  Entry: THistoryEntry;
  i: Integer;
begin
  // ล้าง redo history
  for i := 0 to FRedoHistory.Count - 1 do
    THistoryEntry(FRedoHistory[i]).Free;
  FRedoHistory.Clear;
  
  // บันทึก state ปัจจุบัน
  Entry := THistoryEntry.Create(FCanvas);
  FUndoHistory.Add(Entry);
  
  // จำกัดประวัติ 50 steps
  while FUndoHistory.Count > 50 do
  begin
    THistoryEntry(FUndoHistory[0]).Free;
    FUndoHistory.Delete(0);
  end;
  
  mnuUndo.Enabled := FUndoHistory.Count > 0;
  mnuRedo.Enabled := False;
end;

procedure TfrmPaint.Undo;
var
  Entry: THistoryEntry;
begin
  if FUndoHistory.Count = 0 then Exit;
  
  // บันทึก current state ไปไว้ใน redo
  Entry := THistoryEntry.Create(FCanvas);
  FRedoHistory.Add(Entry);
  
  // เรียกคืน state ก่อนหน้า
  Entry := THistoryEntry(FUndoHistory.Last);
  FCanvas.Assign(Entry.Bitmap);
  FUndoHistory.Remove(Entry);
  Entry.Free;
  
  PaintBox1.Repaint;
  mnuUndo.Enabled := FUndoHistory.Count > 0;
  mnuRedo.Enabled := True;
end;

procedure TfrmPaint.Redo;
var
  Entry: THistoryEntry;
begin
  if FRedoHistory.Count = 0 then Exit;
  
  Entry := THistoryEntry.Create(FCanvas);
  FUndoHistory.Add(Entry);
  
  Entry := THistoryEntry(FRedoHistory.Last);
  FCanvas.Assign(Entry.Bitmap);
  FRedoHistory.Remove(Entry);
  Entry.Free;
  
  PaintBox1.Repaint;
  mnuUndo.Enabled := True;
  mnuRedo.Enabled := FRedoHistory.Count > 0;
end;

procedure TfrmPaint.PaintBoxPaint(Sender: TObject);
begin
  // วาด canvas หลัก
  PaintBox1.Canvas.Draw(0, 0, FCanvas);
  
  // วาด preview (ถ้ากำลัง draw)
  if FDrawing and (FCurrentTool in [ttLine, ttRectangle, ttEllipse]) then
    PaintBox1.Canvas.Draw(0, 0, FTempCanvas);
end;

procedure TfrmPaint.PaintBoxMouseDown(Sender: TObject; Button: TMouseButton;
                                      Shift: TShiftState; X, Y: Integer);
begin
  if Button <> mbLeft then Exit;
  
  FDrawing := True;
  FStartX := X;
  FStartY := Y;
  FLastX := X;
  FLastY := Y;
  
  SaveHistory;
  
  case FCurrentTool of
    ttPen, ttEraser:
      DrawWithPen(X, Y);
    ttFill:
    begin
      FloodFill(X, Y, clbForeColor.ButtonColor);
      PaintBox1.Repaint;
    end;
  end;
end;

procedure TfrmPaint.PaintBoxMouseMove(Sender: TObject; Shift: TShiftState; 
                                      X, Y: Integer);
begin
  UpdateStatus(X, Y);
  
  if not FDrawing then Exit;
  
  case FCurrentTool of
    ttPen:
    begin
      // Draw line segment
      FCanvas.Canvas.Pen.Color := clbForeColor.ButtonColor;
      FCanvas.Canvas.Pen.Width := trkPenSize.Position;
      FCanvas.Canvas.MoveTo(FLastX, FLastY);
      FCanvas.Canvas.LineTo(X, Y);
      FLastX := X;
      FLastY := Y;
      PaintBox1.Repaint;
    end;
    
    ttEraser:
    begin
      EraseAt(X, Y);
      FLastX := X;
      FLastY := Y;
      PaintBox1.Repaint;
    end;
    
    ttLine, ttRectangle, ttEllipse:
    begin
      // Draw preview on temp canvas
      FTempCanvas.Assign(FCanvas);
      
      FTempCanvas.Canvas.Pen.Color := clbForeColor.ButtonColor;
      FTempCanvas.Canvas.Pen.Width := trkPenSize.Position;
      FTempCanvas.Canvas.Brush.Style := bsClear;
      
      case FCurrentTool of
        ttLine:      DrawLine(FStartX, FStartY, X, Y, True);
        ttRectangle: DrawRect(FStartX, FStartY, X, Y, True);
        ttEllipse:   DrawEllipse(FStartX, FStartY, X, Y, True);
      end;
      
      PaintBox1.Repaint;
    end;
  end;
end;

procedure TfrmPaint.PaintBoxMouseUp(Sender: TObject; Button: TMouseButton;
                                    Shift: TShiftState; X, Y: Integer);
begin
  if not FDrawing then Exit;
  FDrawing := False;
  
  case FCurrentTool of
    ttLine:
    begin
      DrawLine(FStartX, FStartY, X, Y);
      PaintBox1.Repaint;
    end;
    
    ttRectangle:
    begin
      DrawRect(FStartX, FStartY, X, Y);
      PaintBox1.Repaint;
    end;
    
    ttEllipse:
    begin
      DrawEllipse(FStartX, FStartY, X, Y);
      PaintBox1.Repaint;
    end;
  end;
end;

procedure TfrmPaint.DrawWithPen(X, Y: Integer);
begin
  FCanvas.Canvas.Pen.Color := clbForeColor.ButtonColor;
  FCanvas.Canvas.Pen.Width := trkPenSize.Position;
  FCanvas.Canvas.Pixels[X, Y] := clbForeColor.ButtonColor;
end;

procedure TfrmPaint.DrawLine(X1, Y1, X2, Y2: Integer; Preview: Boolean = False);
var
  C: TCanvas;
begin
  if Preview then
    C := FTempCanvas.Canvas
  else
    C := FCanvas.Canvas;
    
  C.Pen.Color := clbForeColor.ButtonColor;
  C.Pen.Width := trkPenSize.Position;
  C.MoveTo(X1, Y1);
  C.LineTo(X2, Y2);
end;

procedure TfrmPaint.DrawRect(X1, Y1, X2, Y2: Integer; Preview: Boolean = False);
var
  C: TCanvas;
begin
  if Preview then C := FTempCanvas.Canvas else C := FCanvas.Canvas;
  C.Pen.Color := clbForeColor.ButtonColor;
  C.Pen.Width := trkPenSize.Position;
  C.Brush.Style := bsClear;
  C.Rectangle(Min(X1,X2), Min(Y1,Y2), Max(X1,X2), Max(Y1,Y2));
end;

procedure TfrmPaint.DrawEllipse(X1, Y1, X2, Y2: Integer; Preview: Boolean = False);
var
  C: TCanvas;
begin
  if Preview then C := FTempCanvas.Canvas else C := FCanvas.Canvas;
  C.Pen.Color := clbForeColor.ButtonColor;
  C.Pen.Width := trkPenSize.Position;
  C.Brush.Style := bsClear;
  C.Ellipse(Min(X1,X2), Min(Y1,Y2), Max(X1,X2), Max(Y1,Y2));
end;

procedure TfrmPaint.FloodFill(X, Y: Integer; Color: TColor);
begin
  FCanvas.Canvas.Brush.Color := Color;
  FCanvas.Canvas.FloodFill(X, Y, FCanvas.Canvas.Pixels[X, Y], fsSurface);
end;

procedure TfrmPaint.EraseAt(X, Y: Integer);
var
  S: Integer;
begin
  S := trkPenSize.Position * 4;
  FCanvas.Canvas.Brush.Color := clWhite;
  FCanvas.Canvas.Pen.Style := psClear;
  FCanvas.Canvas.FillRect(Rect(X - S, Y - S, X + S, Y + S));
  FCanvas.Canvas.Pen.Style := psSolid;
end;

procedure TfrmPaint.UpdateStatus(X, Y: Integer);
begin
  StatusBar1.Panels[0].Text := Format('X: %d, Y: %d', [X, Y]);
  StatusBar1.Panels[1].Text := Format('%dx%d', [FCanvas.Width, FCanvas.Height]);
end;

procedure TfrmPaint.mnuNewClick(Sender: TObject);
begin
  if MessageDlg('สร้างใหม่', 'ต้องการสร้างภาพใหม่? ข้อมูลปัจจุบันจะหายไป',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    InitCanvas(800, 600);
    FCurrentFile := '';
    Caption := 'Paint - ไม่มีชื่อ';
    PaintBox1.Repaint;
  end;
end;

procedure TfrmPaint.mnuOpenClick(Sender: TObject);
var
  OpenDlg: TOpenDialog;
begin
  OpenDlg := TOpenDialog.Create(Self);
  try
    OpenDlg.Filter := 'Image Files|*.bmp;*.jpg;*.png|All Files|*.*';
    
    if OpenDlg.Execute then
    begin
      var LoadedBmp := TBitmapHelper.LoadBitmap(OpenDlg.FileName);
      try
        FCanvas.Assign(LoadedBmp);
        FCurrentFile := OpenDlg.FileName;
        Caption := 'Paint - ' + ExtractFileName(OpenDlg.FileName);
        PaintBox1.Repaint;
      finally
        LoadedBmp.Free;
      end;
    end;
  finally
    OpenDlg.Free;
  end;
end;

procedure TfrmPaint.mnuSaveClick(Sender: TObject);
begin
  if FCurrentFile = '' then
    mnuSaveAs.Click
  else
    TBitmapHelper.SaveBitmap(FCanvas, FCurrentFile);
end;

procedure TfrmPaint.mnuUndoClick(Sender: TObject);
begin
  Undo;
end;

procedure TfrmPaint.mnuRedoClick(Sender: TObject);
begin
  Redo;
end;

procedure TfrmPaint.ToolButtonClick(Sender: TObject);
begin
  case TSpeedButton(Sender).Tag of
    0: FCurrentTool := ttPen;
    1: FCurrentTool := ttLine;
    2: FCurrentTool := ttRectangle;
    3: FCurrentTool := ttEllipse;
    4: FCurrentTool := ttFill;
    5: FCurrentTool := ttEraser;
    6: FCurrentTool := ttText;
  end;
end;

end.
```

---

## ตัวอย่าง Chart Drawing จาก Scratch

```pascal
unit CustomChart;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Math;

type
  TChartType = (ctBar, ctLine, ctPie, ctArea);
  
  TDataSeries = class
    Name: string;
    Values: TDoubleArray;
    Color: TColor;
    constructor Create(const AName: string; AColor: TColor);
    procedure AddValue(V: Double);
  end;

  TCustomChart = class
  private
    FCanvas: TCanvas;
    FBounds: TRect;
    FSeries: TList;
    FTitle: string;
    FXLabels: TStringList;
    FChartType: TChartType;
    
    FPaddingLeft: Integer;
    FPaddingRight: Integer;
    FPaddingTop: Integer;
    FPaddingBottom: Integer;
    
    function GetChartArea: TRect;
    function GetMaxValue: Double;
    function GetMinValue: Double;
    
    procedure DrawBackground;
    procedure DrawTitle;
    procedure DrawAxes;
    procedure DrawXLabels;
    procedure DrawYLabels;
    procedure DrawGrid;
    procedure DrawLegend;
    
    procedure DrawBarChart;
    procedure DrawLineChart;
    procedure DrawPieChart;
    procedure DrawAreaChart;
    
    function ValueToY(Value, MinV, MaxV: Double; ChartArea: TRect): Integer;
    function IndexToX(Idx, Count: Integer; ChartArea: TRect): Integer;
    
  public
    constructor Create(ACanvas: TCanvas; const ABounds: TRect);
    destructor Destroy; override;
    
    function AddSeries(const Name: string; Color: TColor): TDataSeries;
    procedure AddXLabel(const Label_: string);
    procedure Draw;
    
    property Title: string read FTitle write FTitle;
    property ChartType: TChartType read FChartType write FChartType;
  end;

implementation

constructor TDataSeries.Create(const AName: string; AColor: TColor);
begin
  Name := AName;
  Color := AColor;
  SetLength(Values, 0);
end;

procedure TDataSeries.AddValue(V: Double);
begin
  SetLength(Values, Length(Values) + 1);
  Values[High(Values)] := V;
end;

constructor TCustomChart.Create(ACanvas: TCanvas; const ABounds: TRect);
begin
  FCanvas := ACanvas;
  FBounds := ABounds;
  FSeries := TList.Create;
  FXLabels := TStringList.Create;
  FChartType := ctBar;
  
  FPaddingLeft := 60;
  FPaddingRight := 20;
  FPaddingTop := 50;
  FPaddingBottom := 50;
end;

destructor TCustomChart.Destroy;
var
  i: Integer;
begin
  for i := 0 to FSeries.Count - 1 do
    TDataSeries(FSeries[i]).Free;
  FSeries.Free;
  FXLabels.Free;
  inherited;
end;

function TCustomChart.AddSeries(const Name: string; Color: TColor): TDataSeries;
begin
  Result := TDataSeries.Create(Name, Color);
  FSeries.Add(Result);
end;

procedure TCustomChart.AddXLabel(const Label_: string);
begin
  FXLabels.Add(Label_);
end;

function TCustomChart.GetChartArea: TRect;
begin
  Result := Rect(
    FBounds.Left + FPaddingLeft,
    FBounds.Top + FPaddingTop,
    FBounds.Right - FPaddingRight,
    FBounds.Bottom - FPaddingBottom);
end;

function TCustomChart.GetMaxValue: Double;
var
  i, j: Integer;
begin
  Result := -MaxDouble;
  for i := 0 to FSeries.Count - 1 do
    for j := 0 to High(TDataSeries(FSeries[i]).Values) do
      Result := Max(Result, TDataSeries(FSeries[i]).Values[j]);
  if Result = -MaxDouble then Result := 100;
end;

function TCustomChart.GetMinValue: Double;
var
  i, j: Integer;
begin
  Result := MaxDouble;
  for i := 0 to FSeries.Count - 1 do
    for j := 0 to High(TDataSeries(FSeries[i]).Values) do
      Result := Min(Result, TDataSeries(FSeries[i]).Values[j]);
  if Result = MaxDouble then Result := 0;
  Result := Min(Result, 0);  // Always include 0
end;

function TCustomChart.ValueToY(Value, MinV, MaxV: Double; ChartArea: TRect): Integer;
begin
  if MaxV = MinV then
    Result := ChartArea.Bottom
  else
    Result := ChartArea.Bottom - Round((Value - MinV) / (MaxV - MinV) * ChartArea.Height);
end;

function TCustomChart.IndexToX(Idx, Count: Integer; ChartArea: TRect): Integer;
begin
  if Count <= 1 then
    Result := (ChartArea.Left + ChartArea.Right) div 2
  else
    Result := ChartArea.Left + Round(Idx * ChartArea.Width / (Count - 1));
end;

procedure TCustomChart.DrawBackground;
begin
  FCanvas.Brush.Color := clWhite;
  FCanvas.Pen.Color := clSilver;
  FCanvas.Pen.Width := 1;
  FCanvas.FillRect(FBounds);
  FCanvas.Rectangle(FBounds);
end;

procedure TCustomChart.DrawTitle;
begin
  if FTitle = '' then Exit;
  
  FCanvas.Font.Size := 14;
  FCanvas.Font.Bold := True;
  FCanvas.Font.Color := clBlack;
  FCanvas.Brush.Style := bsClear;
  
  var TW := FCanvas.TextWidth(FTitle);
  var X := FBounds.Left + (FBounds.Width - TW) div 2;
  var Y := FBounds.Top + 10;
  
  FCanvas.TextOut(X, Y, FTitle);
  FCanvas.Font.Bold := False;
end;

procedure TCustomChart.DrawAxes;
var
  ChartArea: TRect;
begin
  ChartArea := GetChartArea;
  
  FCanvas.Pen.Color := clBlack;
  FCanvas.Pen.Width := 2;
  
  // Y Axis
  FCanvas.MoveTo(ChartArea.Left, ChartArea.Top);
  FCanvas.LineTo(ChartArea.Left, ChartArea.Bottom);
  
  // X Axis
  FCanvas.MoveTo(ChartArea.Left, ChartArea.Bottom);
  FCanvas.LineTo(ChartArea.Right, ChartArea.Bottom);
  
  FCanvas.Pen.Width := 1;
end;

procedure TCustomChart.DrawGrid;
var
  ChartArea: TRect;
  MinV, MaxV: Double;
  GridLines: Integer;
  StepY: Double;
  Y: Integer;
  i: Integer;
begin
  ChartArea := GetChartArea;
  MinV := GetMinValue;
  MaxV := GetMaxValue;
  GridLines := 5;
  StepY := (MaxV - MinV) / GridLines;
  
  FCanvas.Pen.Color := clLtGray;
  FCanvas.Pen.Width := 1;
  FCanvas.Pen.Style := psDot;
  
  for i := 1 to GridLines do
  begin
    Y := ValueToY(MinV + StepY * i, MinV, MaxV, ChartArea);
    FCanvas.MoveTo(ChartArea.Left, Y);
    FCanvas.LineTo(ChartArea.Right, Y);
  end;
  
  FCanvas.Pen.Style := psSolid;
end;

procedure TCustomChart.DrawYLabels;
var
  ChartArea: TRect;
  MinV, MaxV: Double;
  StepV: Double;
  GridLines: Integer;
  Y: Integer;
  LabelStr: string;
begin
  ChartArea := GetChartArea;
  MinV := GetMinValue;
  MaxV := GetMaxValue;
  GridLines := 5;
  StepV := (MaxV - MinV) / GridLines;
  
  FCanvas.Font.Size := 8;
  FCanvas.Font.Color := clDkGray;
  FCanvas.Brush.Style := bsClear;
  
  for var i := 0 to GridLines do
  begin
    Y := ValueToY(MinV + StepV * i, MinV, MaxV, ChartArea);
    LabelStr := FormatFloat('#,##0.#', MinV + StepV * i);
    var LW := FCanvas.TextWidth(LabelStr);
    FCanvas.TextOut(ChartArea.Left - LW - 5, Y - 7, LabelStr);
  end;
end;

procedure TCustomChart.DrawXLabels;
var
  ChartArea: TRect;
  i, X: Integer;
  Count: Integer;
begin
  if FXLabels.Count = 0 then Exit;
  
  ChartArea := GetChartArea;
  Count := FXLabels.Count;
  
  FCanvas.Font.Size := 8;
  FCanvas.Font.Color := clDkGray;
  FCanvas.Brush.Style := bsClear;
  
  for i := 0 to Count - 1 do
  begin
    X := ChartArea.Left + Round(i * ChartArea.Width / (Count - 1));
    if FChartType = ctBar then
      X := ChartArea.Left + Round((i + 0.5) * ChartArea.Width / Count);
    
    var LW := FCanvas.TextWidth(FXLabels[i]);
    FCanvas.TextOut(X - LW div 2, ChartArea.Bottom + 5, FXLabels[i]);
  end;
end;

procedure TCustomChart.DrawBarChart;
var
  ChartArea: TRect;
  MinV, MaxV: Double;
  SeriesCount, DataCount: Integer;
  BarWidth, BarGroupWidth: Integer;
  i, j: Integer;
  X, Y, BarHeight: Integer;
  Series: TDataSeries;
begin
  ChartArea := GetChartArea;
  MinV := GetMinValue;
  MaxV := GetMaxValue;
  SeriesCount := FSeries.Count;
  
  if SeriesCount = 0 then Exit;
  
  DataCount := TDataSeries(FSeries[0]).Values.Count;
  BarGroupWidth := ChartArea.Width div DataCount;
  BarWidth := Max(1, (BarGroupWidth - 10) div SeriesCount);
  
  for i := 0 to DataCount - 1 do
  begin
    var GroupX := ChartArea.Left + i * BarGroupWidth + 5;
    
    for j := 0 to SeriesCount - 1 do
    begin
      Series := TDataSeries(FSeries[j]);
      if i >= Length(Series.Values) then Continue;
      
      var Value := Series.Values[i];
      
      X := GroupX + j * BarWidth;
      Y := ValueToY(Value, MinV, MaxV, ChartArea);
      BarHeight := ChartArea.Bottom - Y;
      
      // Draw bar
      FCanvas.Brush.Color := Series.Color;
      FCanvas.Pen.Color := DarkenColor(Series.Color, 30);
      FCanvas.Pen.Width := 1;
      FCanvas.Rectangle(X, Y, X + BarWidth - 2, ChartArea.Bottom);
      
      // Draw value label
      if BarHeight > 15 then
      begin
        FCanvas.Font.Size := 7;
        FCanvas.Font.Color := clWhite;
        FCanvas.Brush.Style := bsClear;
        var ValStr := FormatFloat('#,##0', Value);
        var VW := FCanvas.TextWidth(ValStr);
        FCanvas.TextOut(X + (BarWidth - 2 - VW) div 2, Y + 3, ValStr);
      end;
    end;
  end;
end;

procedure TCustomChart.DrawLineChart;
var
  ChartArea: TRect;
  MinV, MaxV: Double;
  DataCount, i, j: Integer;
  X, Y: Integer;
  PrevX, PrevY: Integer;
  Series: TDataSeries;
begin
  ChartArea := GetChartArea;
  MinV := GetMinValue;
  MaxV := GetMaxValue;
  
  for j := 0 to FSeries.Count - 1 do
  begin
    Series := TDataSeries(FSeries[j]);
    DataCount := Length(Series.Values);
    
    if DataCount = 0 then Continue;
    
    FCanvas.Pen.Color := Series.Color;
    FCanvas.Pen.Width := 2;
    FCanvas.Brush.Color := Series.Color;
    FCanvas.Brush.Style := bsSolid;
    
    PrevX := -1;
    PrevY := -1;
    
    for i := 0 to DataCount - 1 do
    begin
      X := IndexToX(i, DataCount, ChartArea);
      Y := ValueToY(Series.Values[i], MinV, MaxV, ChartArea);
      
      if PrevX >= 0 then
      begin
        FCanvas.Pen.Color := Series.Color;
        FCanvas.MoveTo(PrevX, PrevY);
        FCanvas.LineTo(X, Y);
      end;
      
      // Draw point
      FCanvas.Ellipse(X - 4, Y - 4, X + 4, Y + 4);
      
      PrevX := X;
      PrevY := Y;
    end;
  end;
end;

procedure TCustomChart.DrawPieChart;
var
  ChartArea: TRect;
  Total: Double;
  StartAngle, SweepAngle: Integer;
  CX, CY, Radius: Integer;
  i, j: Integer;
  Series: TDataSeries;
  LabelAngle: Double;
  LX, LY: Integer;
begin
  ChartArea := GetChartArea;
  CX := (ChartArea.Left + ChartArea.Right) div 2;
  CY := (ChartArea.Top + ChartArea.Bottom) div 2;
  Radius := Min(ChartArea.Width, ChartArea.Height) div 2 - 20;
  
  // คำนวณยอดรวม
  Total := 0;
  if FSeries.Count > 0 then
    for var V in TDataSeries(FSeries[0]).Values do
      Total := Total + V;
  
  if Total = 0 then Exit;
  
  StartAngle := -90;  // เริ่มที่ 12 นาฬิกา
  
  if FSeries.Count > 0 then
  begin
    Series := TDataSeries(FSeries[0]);
    
    for i := 0 to Length(Series.Values) - 1 do
    begin
      SweepAngle := Round(Series.Values[i] / Total * 360);
      
      // สีแต่ละชิ้น
      var PieColor: TColor;
      if i < FSeries.Count then
        PieColor := TDataSeries(FSeries[i]).Color
      else
        PieColor := RGB(Random(200) + 55, Random(200) + 55, Random(200) + 55);
      
      FCanvas.Brush.Color := PieColor;
      FCanvas.Pen.Color := clWhite;
      FCanvas.Pen.Width := 2;
      
      // วาด pie slice
      var R := ChartArea;
      R := Rect(CX - Radius, CY - Radius, CX + Radius, CY + Radius);
      
      // Convert angles to points for Pie function
      var A1 := StartAngle * Pi / 180;
      var A2 := (StartAngle + SweepAngle) * Pi / 180;
      
      FCanvas.Pie(R.Left, R.Top, R.Right, R.Bottom,
                  CX + Round(Cos(A1) * Radius),
                  CY + Round(Sin(A1) * Radius),
                  CX + Round(Cos(A2) * Radius),
                  CY + Round(Sin(A2) * Radius));
      
      // Label
      LabelAngle := (StartAngle + SweepAngle / 2) * Pi / 180;
      LX := CX + Round(Cos(LabelAngle) * (Radius * 0.7));
      LY := CY + Round(Sin(LabelAngle) * (Radius * 0.7));
      
      if SweepAngle > 15 then
      begin
        FCanvas.Font.Size := 9;
        FCanvas.Font.Bold := True;
        FCanvas.Font.Color := clWhite;
        FCanvas.Brush.Style := bsClear;
        var Pct := Format('%.1f%%', [Series.Values[i] / Total * 100]);
        FCanvas.TextOut(LX - FCanvas.TextWidth(Pct) div 2, LY - 7, Pct);
        FCanvas.Font.Bold := False;
      end;
      
      Inc(StartAngle, SweepAngle);
    end;
  end;
end;

procedure TCustomChart.DrawLegend;
var
  LX, LY: Integer;
  i: Integer;
  Series: TDataSeries;
begin
  LX := FBounds.Right - FPaddingRight - 150;
  LY := FBounds.Top + FPaddingTop;
  
  FCanvas.Pen.Color := clSilver;
  FCanvas.Brush.Color := clWhite;
  FCanvas.Rectangle(LX - 5, LY - 5, FBounds.Right - 5, 
                    LY + FSeries.Count * 20 + 5);
  
  for i := 0 to FSeries.Count - 1 do
  begin
    Series := TDataSeries(FSeries[i]);
    
    FCanvas.Brush.Color := Series.Color;
    FCanvas.Pen.Color := clBlack;
    FCanvas.Rectangle(LX, LY + i * 20, LX + 15, LY + i * 20 + 12);
    
    FCanvas.Font.Size := 9;
    FCanvas.Font.Color := clBlack;
    FCanvas.Brush.Style := bsClear;
    FCanvas.TextOut(LX + 20, LY + i * 20, Series.Name);
  end;
end;

procedure TCustomChart.Draw;
begin
  DrawBackground;
  DrawTitle;
  
  if FChartType <> ctPie then
  begin
    DrawGrid;
    DrawAxes;
    DrawYLabels;
    DrawXLabels;
  end;
  
  case FChartType of
    ctBar:  DrawBarChart;
    ctLine: DrawLineChart;
    ctPie:  DrawPieChart;
    ctArea: DrawAreaChart;
  end;
  
  if FSeries.Count > 1 then
    DrawLegend;
end;

procedure TCustomChart.DrawAreaChart;
var
  ChartArea: TRect;
  MinV, MaxV: Double;
  DataCount, i, j: Integer;
  Series: TDataSeries;
  Points: array of TPoint;
begin
  ChartArea := GetChartArea;
  MinV := GetMinValue;
  MaxV := GetMaxValue;
  
  for j := FSeries.Count - 1 downto 0 do
  begin
    Series := TDataSeries(FSeries[j]);
    DataCount := Length(Series.Values);
    if DataCount = 0 then Continue;
    
    SetLength(Points, DataCount + 2);
    
    // First point: bottom left
    Points[0] := Point(IndexToX(0, DataCount, ChartArea), ChartArea.Bottom);
    
    for i := 0 to DataCount - 1 do
      Points[i + 1] := Point(
        IndexToX(i, DataCount, ChartArea),
        ValueToY(Series.Values[i], MinV, MaxV, ChartArea));
    
    // Last point: bottom right
    Points[DataCount + 1] := Point(
      IndexToX(DataCount - 1, DataCount, ChartArea), ChartArea.Bottom);
    
    // Fill area
    FCanvas.Brush.Color := LightenColor(Series.Color, 80);
    FCanvas.Pen.Style := psClear;
    FCanvas.Polygon(Points);
    
    // Draw line on top
    FCanvas.Pen.Style := psSolid;
    FCanvas.Pen.Color := Series.Color;
    FCanvas.Pen.Width := 2;
    FCanvas.Brush.Style := bsClear;
    
    for i := 1 to DataCount - 1 do
    begin
      FCanvas.MoveTo(Points[i].X, Points[i].Y);
      FCanvas.LineTo(Points[i + 1].X, Points[i + 1].Y);
    end;
  end;
end;

// Helper functions
function DarkenColor(Color: TColor; Amount: Integer): TColor;
begin
  var R := Max(0, GetRValue(Color) - Amount);
  var G := Max(0, GetGValue(Color) - Amount);
  var B := Max(0, GetBValue(Color) - Amount);
  Result := RGB(R, G, B);
end;

function LightenColor(Color: TColor; Amount: Integer): TColor;
begin
  var R := Min(255, GetRValue(Color) + Amount);
  var G := Min(255, GetGValue(Color) + Amount);
  var B := Min(255, GetBValue(Color) + Amount);
  Result := RGB(R, G, B);
end;

end.
```

---

## แบบฝึกหัด 15 ข้อ

**ข้อ 1:** เขียนโปรแกรมวาดสัญลักษณ์ทางคณิตศาสตร์
```pascal
// วาด: บวก, ลบ, คูณ, หาร, เท่ากับ
// ใช้ polygon สำหรับสัญลักษณ์ต่างๆ
// ปรับขนาดได้
```

**ข้อ 2:** สร้าง Digital Clock ด้วย TCanvas
```pascal
// แสดงเวลาเป็นตัวเลข 7-segment
// Update ทุกวินาที
// รองรับ 12h/24h format
// สีพื้นหลังปรับได้
```

**ข้อ 3:** วาดแผนที่อย่างง่าย
```pascal
// รับข้อมูล coordinates
// วาดเส้นถนน
// แสดง location markers
// Zoom in/out
```

**ข้อ 4:** สร้าง QR Code generator (simplified)
```pascal
// สร้าง matrix ของ black/white squares
// แสดงผลเป็น bitmap
// Export เป็น PNG
```

**ข้อ 5:** เขียน Mandelbrot Set viewer
```pascal
// คำนวณ Mandelbrot Set
// แสดงผลด้วยสีสวยงาม
// Zoom ได้
// บันทึกเป็น PNG
```

**ข้อ 6:** สร้าง Histogram จาก image
```pascal
// อ่านรูปภาพ
// นับความถี่ของแต่ละ color value
// แสดง histogram ของ R, G, B
// แสดง histogram แบบ luminance
```

**ข้อ 7:** เขียน Sprite animation
```pascal
// โหลด sprite sheet
// ตัด sprites ออกมา
// เล่น animation ตาม frame rate
// รองรับ multiple animations
```

**ข้อ 8:** สร้าง Progress bar แบบ custom
```pascal
// Rounded corners
// Gradient fill
// Animated stripes
// แสดงเปอร์เซ็นต์
// หลายสไตล์
```

**ข้อ 9:** เขียนโปรแกรม Image Viewer
```pascal
// โหลดรูปหลายรูป
// แสดง thumbnail grid
// Zoom in/out
// Slide show
// Basic editing: rotate, crop, brightness
```

**ข้อ 10:** วาด Flowchart อัตโนมัติ
```pascal
// รับข้อมูลขั้นตอน
// วาด box, diamond, arrow อัตโนมัติ
// Auto layout
// Export เป็น PNG/SVG
```

**ข้อ 11:** สร้าง Color Picker
```pascal
// Color wheel
// HSV/RGB sliders
// Eyedropper tool
// Recent colors
// Copy hex code
```

**ข้อ 12:** เขียน Waveform visualizer
```pascal
// รับข้อมูล audio samples
// วาด waveform
// แสดง frequency spectrum
// Animation real-time
```

**ข้อ 13:** สร้าง Drawing tool แบบ Vector
```pascal
// วาดรูปทรงและบันทึกเป็น vector
// เลือก, ย้าย, resize shapes
// Layer system
// Export SVG
```

**ข้อ 14:** สร้าง Signature Pad
```pascal
// วาดลายเซ็น
// Smooth curve (Bezier)
// ล้างและวาดใหม่
// Save เป็น PNG transparent
```

**ข้อ 15:** โปรเจกต์สุดท้าย - Paint Application สมบูรณ์
```pascal
// Tools: pen, eraser, shapes, fill, text, selection
// Layers system
// Undo/Redo (50 steps)
// Zoom in/out
// Export BMP/PNG/JPEG
// Import image for editing
// Filters: blur, sharpen, grayscale, sepia
// Custom brushes
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- TCanvas และ coordinate system
- TPen: color, width, style, mode
- TBrush: style, color, gradient
- การวาดรูปทรงพื้นฐาน: lines, rectangles, ellipses, arcs, polygons
- การวาดข้อความ: font, alignment, rotation
- TBitmap: โหลด, บันทึก, แก้ไข
- Color operations: grayscale, brightness, sepia, invert
- Double buffering สำหรับ animation
- Alpha blending และ transparency
- Paint application สมบูรณ์
- Chart drawing จาก scratch

กราฟิกใน Lazarus มีความยืดหยุ่นสูงมาก สามารถสร้างแอปพลิเคชันภาพที่ซับซ้อนได้โดยใช้ TCanvas API ที่ทำงานข้ามแพลตฟอร์ม
