# Part 36 - Canvas และ Drawing (เชิงลึก)

## บทนำ

Canvas คือพื้นที่วาดภาพในโปรแกรม Lazarus/Pascal ที่ช่วยให้เราสามารถวาดรูปทรงต่างๆ ข้อความ และภาพลงบน component ได้ ในบทนี้เราจะเรียนรู้การใช้งาน Canvas อย่างลึกซึ้ง รวมถึงเทคนิคการวาดขั้นสูง

---

## 36.1 พื้นฐาน Canvas

### TCanvas คืออะไร

`TCanvas` เป็น class หลักที่ใช้ในการวาดภาพใน Lazarus ทุก component ที่มีการแสดงผลจะมี property ชื่อ `Canvas` ที่เป็น `TCanvas`

```pascal
// การเข้าถึง Canvas ของ Form
procedure TForm1.FormPaint(Sender: TObject);
begin
  // Canvas ของ Form
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 2;
  Canvas.Rectangle(10, 10, 200, 100);
  
  // วาดข้อความ
  Canvas.Font.Name := 'Arial';
  Canvas.Font.Size := 14;
  Canvas.Font.Color := clBlue;
  Canvas.TextOut(20, 50, 'Hello Canvas!');
end;
```

### Coordinate System

ระบบพิกัดของ Canvas ใช้ระบบ (x, y) โดย:
- จุด (0, 0) อยู่ที่มุมบนซ้าย
- x เพิ่มขึ้นไปทางขวา
- y เพิ่มขึ้นลงด้านล่าง

```pascal
procedure TForm1.FormPaint(Sender: TObject);
begin
  // วาดเส้นแกน
  Canvas.Pen.Color := clRed;
  Canvas.MoveTo(0, 0);
  Canvas.LineTo(Width, 0);    // แกน X
  
  Canvas.Pen.Color := clBlue;
  Canvas.MoveTo(0, 0);
  Canvas.LineTo(0, Height);   // แกน Y
  
  // วาดจุดทดสอบ
  Canvas.Pixels[100, 100] := clGreen;  // จุดเขียว
end;
```

---

## 36.2 การใช้งาน Pen และ Brush

### TPen - ควบคุมการวาดเส้น

```pascal
procedure TForm1.FormPaint(Sender: TObject);
begin
  // ตัวอย่างการใช้ Pen แบบต่างๆ
  
  // เส้นทึบ
  Canvas.Pen.Style := psSolid;
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 2;
  Canvas.MoveTo(10, 10);
  Canvas.LineTo(200, 10);
  
  // เส้นประ
  Canvas.Pen.Style := psDash;
  Canvas.Pen.Color := clRed;
  Canvas.MoveTo(10, 30);
  Canvas.LineTo(200, 30);
  
  // เส้นจุด
  Canvas.Pen.Style := psDot;
  Canvas.Pen.Color := clBlue;
  Canvas.MoveTo(10, 50);
  Canvas.LineTo(200, 50);
  
  // เส้นประจุด
  Canvas.Pen.Style := psDashDot;
  Canvas.Pen.Color := clGreen;
  Canvas.MoveTo(10, 70);
  Canvas.LineTo(200, 70);
  
  // ไม่มีเส้น
  Canvas.Pen.Style := psInsideFrame;
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 3;
  Canvas.Rectangle(10, 90, 200, 130);
end;
```

### TBrush - ควบคุมการเติมสี

```pascal
procedure TForm1.FormPaint(Sender: TObject);
var
  i: Integer;
begin
  // แสดง Brush Style ต่างๆ
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 1;
  
  // bsSolid - เติมสีทึบ
  Canvas.Brush.Style := bsSolid;
  Canvas.Brush.Color := clYellow;
  Canvas.Rectangle(10, 10, 80, 60);
  Canvas.TextOut(15, 65, 'Solid');
  
  // bsClear - ไม่เติมสี
  Canvas.Brush.Style := bsClear;
  Canvas.Rectangle(100, 10, 170, 60);
  Canvas.TextOut(105, 65, 'Clear');
  
  // bsHorizontal - เส้นนอน
  Canvas.Brush.Style := bsHorizontal;
  Canvas.Brush.Color := clBlue;
  Canvas.Rectangle(190, 10, 260, 60);
  Canvas.TextOut(195, 65, 'Horiz');
  
  // bsVertical - เส้นตั้ง
  Canvas.Brush.Style := bsVertical;
  Canvas.Brush.Color := clRed;
  Canvas.Rectangle(280, 10, 350, 60);
  Canvas.TextOut(285, 65, 'Vert');
  
  // bsCross - ตาราง
  Canvas.Brush.Style := bsCross;
  Canvas.Brush.Color := clGreen;
  Canvas.Rectangle(10, 90, 80, 140);
  Canvas.TextOut(15, 145, 'Cross');
  
  // bsDiagCross - ตารางเฉียง
  Canvas.Brush.Style := bsDiagCross;
  Canvas.Brush.Color := clPurple;
  Canvas.Rectangle(100, 90, 170, 140);
  Canvas.TextOut(105, 145, 'DiagX');
end;
```

---

## 36.3 การวาดรูปทรงพื้นฐาน

### วาดรูปทรงต่างๆ

```pascal
program DrawShapes;

uses
  Forms, Graphics, Controls;

{$R *.lfm}

procedure TForm1.FormPaint(Sender: TObject);
begin
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 2;
  Canvas.Brush.Color := clSkyBlue;
  Canvas.Brush.Style := bsSolid;
  
  // สี่เหลี่ยมผืนผ้า
  Canvas.Rectangle(20, 20, 120, 80);
  Canvas.TextOut(45, 85, 'Rectangle');
  
  // วงรี
  Canvas.Brush.Color := clLime;
  Canvas.Ellipse(150, 20, 250, 80);
  Canvas.TextOut(175, 85, 'Ellipse');
  
  // สี่เหลี่ยมมุมมน
  Canvas.Brush.Color := clOrange;
  Canvas.RoundRect(280, 20, 380, 80, 20, 20);
  Canvas.TextOut(300, 85, 'RoundRect');
  
  // เส้นโค้งส่วนหนึ่งของวงรี (Arc)
  Canvas.Pen.Color := clRed;
  Canvas.Pen.Width := 3;
  Canvas.Arc(20, 120, 120, 200, 170, 120, 20, 120);
  Canvas.TextOut(45, 205, 'Arc');
  
  // รูปส่วนของวงกลม (Pie)
  Canvas.Brush.Color := clYellow;
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 1;
  Canvas.Pie(150, 120, 250, 200, 150, 120, 250, 160);
  Canvas.TextOut(175, 205, 'Pie');
  
  // ส่วนโค้งที่เชื่อมจุด (Chord)
  Canvas.Brush.Color := clFuchsia;
  Canvas.Chord(280, 120, 380, 200, 280, 120, 380, 160);
  Canvas.TextOut(305, 205, 'Chord');
  
  // รูปหลายเหลี่ยม (Polygon)
  Canvas.Brush.Color := clTeal;
  Canvas.Polygon([Point(60, 240), Point(120, 220), Point(140, 280),
                   Point(80, 310), Point(20, 280)]);
  Canvas.TextOut(45, 320, 'Polygon');
  
  // เส้นที่เชื่อมจุดต่างๆ (Polyline)
  Canvas.Pen.Color := clNavy;
  Canvas.Pen.Width := 2;
  Canvas.Polyline([Point(170, 220), Point(220, 240), Point(200, 280),
                    Point(250, 300), Point(170, 310)]);
  Canvas.TextOut(185, 320, 'Polyline');
end;
```

---

## 36.4 Clipping Regions

Clipping Region คือพื้นที่ที่อนุญาตให้วาดได้ ส่วนที่อยู่นอก Region จะไม่ถูกวาด

```pascal
unit ClippingDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Windows;

type
  TForm1 = class(TForm)
    procedure FormPaint(Sender: TObject);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormPaint(Sender: TObject);
var
  ClipRegion: HRGN;
  OldClipRegion: HRGN;
begin
  // สร้าง Circular Clipping Region
  ClipRegion := CreateEllipticRgn(50, 50, 250, 250);
  
  // บันทึก Clip Region เดิม
  OldClipRegion := CreateRectRgn(0, 0, 0, 0);
  GetClipRgn(Canvas.Handle, OldClipRegion);
  
  // ตั้งค่า Clipping Region ใหม่
  SelectClipRgn(Canvas.Handle, ClipRegion);
  
  // วาดพื้นหลัง - จะถูกตัดให้อยู่ในวงกลม
  Canvas.Brush.Color := clSkyBlue;
  Canvas.FillRect(Rect(0, 0, Width, Height));
  
  // วาดเส้นตาราง
  Canvas.Pen.Color := clWhite;
  Canvas.Pen.Width := 1;
  var i: Integer;
  for i := 0 to Width div 20 do
  begin
    Canvas.MoveTo(i * 20, 0);
    Canvas.LineTo(i * 20, Height);
  end;
  for i := 0 to Height div 20 do
  begin
    Canvas.MoveTo(0, i * 20);
    Canvas.LineTo(Width, i * 20);
  end;
  
  // วาดข้อความ
  Canvas.Font.Size := 20;
  Canvas.Font.Color := clYellow;
  Canvas.Font.Style := [fsBold];
  Canvas.TextOut(60, 130, 'CLIPPED!');
  
  // คืนค่า Clip Region เดิม
  SelectClipRgn(Canvas.Handle, OldClipRegion);
  
  // วาดกรอบวงกลม
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 3;
  Canvas.Brush.Style := bsClear;
  Canvas.Ellipse(50, 50, 250, 250);
  
  // ลบ Region Objects
  DeleteObject(ClipRegion);
  DeleteObject(OldClipRegion);
end;

end.
```

### Clipping Region แบบซับซ้อน

```pascal
procedure TForm1.FormPaint(Sender: TObject);
var
  Rgn1, Rgn2, CombinedRgn: HRGN;
begin
  // สร้าง Region สองอัน
  Rgn1 := CreateRectRgn(50, 50, 200, 200);
  Rgn2 := CreateEllipticRgn(100, 100, 300, 300);
  CombinedRgn := CreateRectRgn(0, 0, 0, 0);
  
  // รวม Region แบบ Union (บวกกัน)
  CombineRgn(CombinedRgn, Rgn1, Rgn2, RGN_OR);
  SelectClipRgn(Canvas.Handle, CombinedRgn);
  
  Canvas.Brush.Color := clLime;
  Canvas.FillRect(Rect(0, 0, Width, Height));
  
  SelectClipRgn(Canvas.Handle, 0);
  
  // แสดง Region แบบ Intersection (ตัดกัน)
  CombineRgn(CombinedRgn, Rgn1, Rgn2, RGN_AND);
  SelectClipRgn(Canvas.Handle, CombinedRgn);
  
  Canvas.Brush.Color := clRed;
  Canvas.FillRect(Rect(0, 0, Width, Height));
  
  SelectClipRgn(Canvas.Handle, 0);
  
  DeleteObject(Rgn1);
  DeleteObject(Rgn2);
  DeleteObject(CombinedRgn);
end;
```

---

## 36.5 การแปลงพิกัด (Transformations)

### วิธีที่ 1: การใช้ Windows API (สำหรับ Windows)

```pascal
unit TransformDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Windows, Math;

type
  TForm1 = class(TForm)
    procedure FormPaint(Sender: TObject);
  private
    procedure DrawArrow(ACanvas: TCanvas; CenterX, CenterY: Integer);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.DrawArrow(ACanvas: TCanvas; CenterX, CenterY: Integer);
begin
  // วาดลูกศรที่จุดกึ่งกลาง
  ACanvas.Polygon([
    Point(CenterX, CenterY - 40),
    Point(CenterX + 20, CenterY),
    Point(CenterX + 10, CenterY),
    Point(CenterX + 10, CenterY + 40),
    Point(CenterX - 10, CenterY + 40),
    Point(CenterX - 10, CenterY),
    Point(CenterX - 20, CenterY)
  ]);
end;

procedure TForm1.FormPaint(Sender: TObject);
var
  XForm: XFORM;
  OldMode: Integer;
  Angle: Double;
  i: Integer;
begin
  // เปิดใช้งาน World Transforms
  OldMode := SetGraphicsMode(Canvas.Handle, GM_ADVANCED);
  
  // วาดรูปเดิม
  Canvas.Brush.Color := clGray;
  Canvas.Pen.Color := clBlack;
  DrawArrow(Canvas, 150, 150);
  
  // หมุน 45 องศา
  Angle := 45 * Pi / 180;
  XForm.eM11 := Cos(Angle);
  XForm.eM12 := Sin(Angle);
  XForm.eM21 := -Sin(Angle);
  XForm.eM22 := Cos(Angle);
  XForm.eDx := 300;
  XForm.eDy := 150;
  SetWorldTransform(Canvas.Handle, XForm);
  
  Canvas.Brush.Color := clRed;
  DrawArrow(Canvas, 0, 0);
  
  // คืนค่า Transform
  SetGraphicsMode(Canvas.Handle, OldMode);
  ModifyWorldTransform(Canvas.Handle, XForm, MWT_IDENTITY);
  
  // วาดหลายๆ ตัวโดยหมุน
  for i := 0 to 7 do
  begin
    Angle := i * 45 * Pi / 180;
    XForm.eM11 := Cos(Angle);
    XForm.eM12 := Sin(Angle);
    XForm.eM21 := -Sin(Angle);
    XForm.eM22 := Cos(Angle);
    XForm.eDx := 450;
    XForm.eDy := 200;
    
    SetGraphicsMode(Canvas.Handle, GM_ADVANCED);
    SetWorldTransform(Canvas.Handle, XForm);
    
    Canvas.Brush.Color := TColor(RGB(i * 30, 255 - i * 30, 128));
    Canvas.Pen.Color := clBlack;
    Canvas.Brush.Style := bsSolid;
    Canvas.Ellipse(-5, -50, 5, 50);
    
    SetGraphicsMode(Canvas.Handle, OldMode);
    ModifyWorldTransform(Canvas.Handle, XForm, MWT_IDENTITY);
  end;
end;

end.
```

### วิธีที่ 2: การใช้ Matrix Transform แบบ Manual

```pascal
unit MatrixTransform;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Math;

type
  TMatrix3x3 = array[0..2, 0..2] of Double;
  
  TTransform = class
  private
    FMatrix: TMatrix3x3;
  public
    constructor Create;
    procedure Identity;
    procedure Translate(dx, dy: Double);
    procedure Rotate(Angle: Double);  // Angle in radians
    procedure Scale(sx, sy: Double);
    procedure TransformPoint(var x, y: Double);
    procedure Multiply(const Other: TMatrix3x3);
  end;
  
  TForm1 = class(TForm)
    procedure FormPaint(Sender: TObject);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

constructor TTransform.Create;
begin
  inherited;
  Identity;
end;

procedure TTransform.Identity;
begin
  FMatrix[0][0] := 1; FMatrix[0][1] := 0; FMatrix[0][2] := 0;
  FMatrix[1][0] := 0; FMatrix[1][1] := 1; FMatrix[1][2] := 0;
  FMatrix[2][0] := 0; FMatrix[2][1] := 0; FMatrix[2][2] := 1;
end;

procedure TTransform.Translate(dx, dy: Double);
var
  M: TMatrix3x3;
begin
  M[0][0] := 1; M[0][1] := 0; M[0][2] := dx;
  M[1][0] := 0; M[1][1] := 1; M[1][2] := dy;
  M[2][0] := 0; M[2][1] := 0; M[2][2] := 1;
  Multiply(M);
end;

procedure TTransform.Rotate(Angle: Double);
var
  M: TMatrix3x3;
  C, S: Double;
begin
  C := Cos(Angle);
  S := Sin(Angle);
  M[0][0] := C;  M[0][1] := -S; M[0][2] := 0;
  M[1][0] := S;  M[1][1] := C;  M[1][2] := 0;
  M[2][0] := 0;  M[2][1] := 0;  M[2][2] := 1;
  Multiply(M);
end;

procedure TTransform.Scale(sx, sy: Double);
var
  M: TMatrix3x3;
begin
  M[0][0] := sx; M[0][1] := 0;  M[0][2] := 0;
  M[1][0] := 0;  M[1][1] := sy; M[1][2] := 0;
  M[2][0] := 0;  M[2][1] := 0;  M[2][2] := 1;
  Multiply(M);
end;

procedure TTransform.TransformPoint(var x, y: Double);
var
  nx, ny: Double;
begin
  nx := FMatrix[0][0] * x + FMatrix[0][1] * y + FMatrix[0][2];
  ny := FMatrix[1][0] * x + FMatrix[1][1] * y + FMatrix[1][2];
  x := nx;
  y := ny;
end;

procedure TTransform.Multiply(const Other: TMatrix3x3);
var
  Result: TMatrix3x3;
  i, j, k: Integer;
begin
  for i := 0 to 2 do
    for j := 0 to 2 do
    begin
      Result[i][j] := 0;
      for k := 0 to 2 do
        Result[i][j] := Result[i][j] + FMatrix[i][k] * Other[k][j];
    end;
  FMatrix := Result;
end;

procedure TForm1.FormPaint(Sender: TObject);
var
  T: TTransform;
  x, y: Double;
  Points: array of TPoint;
  i: Integer;
  
  procedure DrawSquare(cx, cy: Double; Size: Double);
  var
    P: array[0..3] of TPoint;
    px, py: Double;
  begin
    px := cx - Size/2; py := cy - Size/2; T.TransformPoint(px, py);
    P[0] := Point(Round(px), Round(py));
    px := cx + Size/2; py := cy - Size/2; T.TransformPoint(px, py);
    P[1] := Point(Round(px), Round(py));
    px := cx + Size/2; py := cy + Size/2; T.TransformPoint(px, py);
    P[2] := Point(Round(px), Round(py));
    px := cx - Size/2; py := cy + Size/2; T.TransformPoint(px, py);
    P[3] := Point(Round(px), Round(py));
    Canvas.Polygon(P);
  end;

begin
  T := TTransform.Create;
  try
    // สี่เหลี่ยมธรรมดา
    T.Identity;
    T.Translate(100, 100);
    Canvas.Brush.Color := clRed;
    Canvas.Pen.Color := clBlack;
    DrawSquare(0, 0, 60);
    Canvas.TextOut(70, 170, 'Normal');
    
    // หมุน 45 องศา
    T.Identity;
    T.Translate(250, 100);
    T.Rotate(45 * Pi / 180);
    Canvas.Brush.Color := clBlue;
    DrawSquare(0, 0, 60);
    Canvas.TextOut(220, 170, 'Rotated 45°');
    
    // ขยาย
    T.Identity;
    T.Translate(400, 100);
    T.Scale(1.5, 0.7);
    Canvas.Brush.Color := clGreen;
    DrawSquare(0, 0, 60);
    Canvas.TextOut(370, 170, 'Scaled');
    
    // หมุนพร้อมขยาย
    T.Identity;
    T.Translate(150, 300);
    T.Rotate(30 * Pi / 180);
    T.Scale(1.2, 1.2);
    Canvas.Brush.Color := clOrange;
    DrawSquare(0, 0, 60);
    Canvas.TextOut(110, 390, 'Rot+Scale');
    
  finally
    T.Free;
  end;
end;

end.
```

---

## 36.6 Anti-aliasing

Anti-aliasing คือเทคนิคการทำให้ขอบของรูปดูเรียบขึ้น โดยลดรอยขรุขระ

### การใช้ GDI+ สำหรับ Anti-aliasing

```pascal
unit AntiAliasingDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Windows;

// GDI+ declarations
type
  GpStatus = Integer;
  GpGraphics = Pointer;
  GpPen = Pointer;
  GpBrush = Pointer;
  GpSolidFill = Pointer;
  GpImage = Pointer;
  GpBitmap = Pointer;
  ARGB = Cardinal;
  
const
  SmoothingModeAntiAlias = 4;
  SmoothingModeDefault = 0;
  
var
  gdiplusToken: ULONG_PTR;

// ประกาศ GDI+ functions (ต้องใช้ GDIPlus.dll)
// นี่เป็นตัวอย่างแบบง่ายโดยไม่ใช้ header เต็ม

type
  TForm1 = class(TForm)
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  // เริ่มต้น GDI+
  // GdiplusStartup(&gdiplusToken, &gdiplusStartupInput, NULL);
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  // ปิด GDI+
  // GdiplusShutdown(gdiplusToken);
end;

// วิธีง่ายๆ ใน Lazarus: ใช้ LCL drawing ธรรมดา
// แต่ใช้เทคนิค Super-sampling เพื่อจำลอง Anti-aliasing

procedure DrawAntiAliasedLine(ACanvas: TCanvas; 
  x1, y1, x2, y2: Integer; AColor: TColor; Width: Integer = 1);
var
  // ใช้ Wu's line algorithm สำหรับ Anti-aliasing
  dx, dy: Double;
  steep: Boolean;
  tmp: Integer;
  gradient: Double;
  xend, yend: Double;
  xgap: Double;
  xpxl1, xpxl2, ypxl1, ypxl2: Integer;
  intery: Double;
  
  procedure Plot(x, y: Integer; c: Double);
  var
    R, G, B: Byte;
    BaseR, BaseG, BaseB: Byte;
    BgColor: TColor;
  begin
    if (x < 0) or (x >= ACanvas.ClipRect.Right) or
       (y < 0) or (y >= ACanvas.ClipRect.Bottom) then Exit;
       
    BgColor := ACanvas.Pixels[x, y];
    BaseR := GetRValue(ColorToRGB(AColor));
    BaseG := GetGValue(ColorToRGB(AColor));
    BaseB := GetBValue(ColorToRGB(AColor));
    R := Round(BaseR * c + GetRValue(ColorToRGB(BgColor)) * (1 - c));
    G := Round(BaseG * c + GetGValue(ColorToRGB(BgColor)) * (1 - c));
    B := Round(BaseB * c + GetBValue(ColorToRGB(BgColor)) * (1 - c));
    ACanvas.Pixels[x, y] := RGB(R, G, B);
  end;
  
  function Frac(v: Double): Double;
  begin
    Result := v - Int(v);
  end;
  
  function RFrac(v: Double): Double;
  begin
    Result := 1 - Frac(v);
  end;

begin
  dx := x2 - x1;
  dy := y2 - y1;
  steep := Abs(dy) > Abs(dx);
  
  if steep then
  begin
    tmp := x1; x1 := y1; y1 := tmp;
    tmp := x2; x2 := y2; y2 := tmp;
  end;
  
  if x1 > x2 then
  begin
    tmp := x1; x1 := x2; x2 := tmp;
    tmp := y1; y1 := y2; y2 := tmp;
  end;
  
  dx := x2 - x1;
  dy := y2 - y1;
  
  if dx = 0 then gradient := 1
  else gradient := dy / dx;
  
  // จุดเริ่มต้น
  xend := Round(x1);
  yend := y1 + gradient * (xend - x1);
  xgap := RFrac(x1 + 0.5);
  xpxl1 := Round(xend);
  ypxl1 := Trunc(yend);
  
  if steep then
  begin
    Plot(ypxl1, xpxl1, RFrac(yend) * xgap);
    Plot(ypxl1 + 1, xpxl1, Frac(yend) * xgap);
  end else
  begin
    Plot(xpxl1, ypxl1, RFrac(yend) * xgap);
    Plot(xpxl1, ypxl1 + 1, Frac(yend) * xgap);
  end;
  
  intery := yend + gradient;
  
  // จุดสิ้นสุด
  xend := Round(x2);
  yend := y2 + gradient * (xend - x2);
  xgap := Frac(x2 + 0.5);
  xpxl2 := Round(xend);
  ypxl2 := Trunc(yend);
  
  if steep then
  begin
    Plot(ypxl2, xpxl2, RFrac(yend) * xgap);
    Plot(ypxl2 + 1, xpxl2, Frac(yend) * xgap);
  end else
  begin
    Plot(xpxl2, ypxl2, RFrac(yend) * xgap);
    Plot(xpxl2, ypxl2 + 1, Frac(yend) * xgap);
  end;
  
  // วาดเส้นกลาง
  var x: Integer;
  for x := xpxl1 + 1 to xpxl2 - 1 do
  begin
    if steep then
    begin
      Plot(Trunc(intery), x, RFrac(intery));
      Plot(Trunc(intery) + 1, x, Frac(intery));
    end else
    begin
      Plot(x, Trunc(intery), RFrac(intery));
      Plot(x, Trunc(intery) + 1, Frac(intery));
    end;
    intery := intery + gradient;
  end;
end;

procedure TForm1.FormPaint(Sender: TObject);
var
  i: Integer;
begin
  Canvas.Brush.Color := clWhite;
  Canvas.FillRect(ClientRect);
  
  // เส้นแบบปกติ (มีรอยขรุขระ)
  Canvas.Font.Size := 12;
  Canvas.TextOut(10, 10, 'เส้นปกติ (Aliased):');
  Canvas.Pen.Color := clBlack;
  Canvas.Pen.Width := 1;
  for i := 0 to 4 do
  begin
    Canvas.MoveTo(10 + i * 5, 40);
    Canvas.LineTo(150 + i * 5, 100 + i * 10);
  end;
  
  // เส้นแบบ Anti-aliased
  Canvas.TextOut(10, 130, 'เส้น Anti-aliased (Wu Algorithm):');
  for i := 0 to 4 do
  begin
    DrawAntiAliasedLine(Canvas, 10 + i * 5, 160, 
                         150 + i * 5, 220 + i * 10, clBlack);
  end;
end;

end.
```

---

## 36.7 เส้นโค้ง Bezier

Bezier curves เป็นเส้นโค้งที่ใช้ในงานกราฟิก โดยกำหนดด้วย Control Points

```pascal
unit BezierDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Math;

type
  TBezierPoint = record
    X, Y: Double;
  end;
  
  TForm1 = class(TForm)
    procedure FormPaint(Sender: TObject);
    procedure FormMouseDown(Sender: TObject; Button: TMouseButton;
      Shift: TShiftState; X, Y: Integer);
    procedure FormMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
  private
    FControlPoints: array[0..3] of TBezierPoint;
    FDragging: Integer;
  public
    constructor Create(AOwner: TComponent); override;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

// คำนวณจุดบน Cubic Bezier curve
function BezierPoint(t: Double; P0, P1, P2, P3: TBezierPoint): TBezierPoint;
var
  mt: Double;
begin
  mt := 1 - t;
  Result.X := mt*mt*mt*P0.X + 3*mt*mt*t*P1.X + 3*mt*t*t*P2.X + t*t*t*P3.X;
  Result.Y := mt*mt*mt*P0.Y + 3*mt*mt*t*P1.Y + 3*mt*t*t*P2.Y + t*t*t*P3.Y;
end;

// วาด Cubic Bezier curve
procedure DrawBezierCurve(ACanvas: TCanvas; P0, P1, P2, P3: TBezierPoint; 
                           Steps: Integer = 100);
var
  i: Integer;
  t: Double;
  Pt: TBezierPoint;
  PtPrev: TBezierPoint;
begin
  PtPrev := P0;
  for i := 1 to Steps do
  begin
    t := i / Steps;
    Pt := BezierPoint(t, P0, P1, P2, P3);
    ACanvas.MoveTo(Round(PtPrev.X), Round(PtPrev.Y));
    ACanvas.LineTo(Round(Pt.X), Round(Pt.Y));
    PtPrev := Pt;
  end;
end;

constructor TForm1.Create(AOwner: TComponent);
begin
  inherited;
  // กำหนด Control Points เริ่มต้น
  FControlPoints[0].X := 50;  FControlPoints[0].Y := 200;   // P0 - Start
  FControlPoints[1].X := 150; FControlPoints[1].Y := 50;    // P1 - Ctrl1
  FControlPoints[2].X := 350; FControlPoints[2].Y := 350;   // P2 - Ctrl2
  FControlPoints[3].X := 450; FControlPoints[3].Y := 200;   // P3 - End
  FDragging := -1;
end;

procedure TForm1.FormPaint(Sender: TObject);
var
  i: Integer;
  Colors: array[0..3] of TColor = (clGreen, clRed, clRed, clGreen);
  Labels: array[0..3] of string = ('P0 (Start)', 'P1 (Ctrl1)', 'P2 (Ctrl2)', 'P3 (End)');
begin
  Canvas.Brush.Color := clWhite;
  Canvas.FillRect(ClientRect);
  
  // วาดเส้นเชื่อม Control Points
  Canvas.Pen.Style := psDash;
  Canvas.Pen.Color := clSilver;
  Canvas.Pen.Width := 1;
  Canvas.MoveTo(Round(FControlPoints[0].X), Round(FControlPoints[0].Y));
  Canvas.LineTo(Round(FControlPoints[1].X), Round(FControlPoints[1].Y));
  Canvas.MoveTo(Round(FControlPoints[2].X), Round(FControlPoints[2].Y));
  Canvas.LineTo(Round(FControlPoints[3].X), Round(FControlPoints[3].Y));
  
  // วาด Bezier curve
  Canvas.Pen.Style := psSolid;
  Canvas.Pen.Color := clBlue;
  Canvas.Pen.Width := 2;
  DrawBezierCurve(Canvas, FControlPoints[0], FControlPoints[1],
                   FControlPoints[2], FControlPoints[3]);
  
  // วาด Control Points
  Canvas.Pen.Width := 1;
  Canvas.Pen.Color := clBlack;
  for i := 0 to 3 do
  begin
    Canvas.Brush.Color := Colors[i];
    Canvas.Ellipse(
      Round(FControlPoints[i].X) - 8,
      Round(FControlPoints[i].Y) - 8,
      Round(FControlPoints[i].X) + 8,
      Round(FControlPoints[i].Y) + 8
    );
    Canvas.Font.Size := 9;
    Canvas.Font.Color := clBlack;
    Canvas.TextOut(Round(FControlPoints[i].X) + 10,
                   Round(FControlPoints[i].Y) - 5,
                   Labels[i]);
  end;
  
  // คำแนะนำ
  Canvas.Font.Size := 10;
  Canvas.Font.Color := clGray;
  Canvas.TextOut(10, 10, 'คลิกและลาก Control Points เพื่อแก้ไขเส้นโค้ง');
end;

procedure TForm1.FormMouseDown(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
var
  i: Integer;
  dist: Double;
begin
  FDragging := -1;
  for i := 0 to 3 do
  begin
    dist := Sqrt(Sqr(X - FControlPoints[i].X) + Sqr(Y - FControlPoints[i].Y));
    if dist < 12 then
    begin
      FDragging := i;
      Break;
    end;
  end;
end;

procedure TForm1.FormMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
begin
  if (ssLeft in Shift) and (FDragging >= 0) then
  begin
    FControlPoints[FDragging].X := X;
    FControlPoints[FDragging].Y := Y;
    Invalidate;
  end;
end;

end.
```

---

## 36.8 Pattern Fills และ Textures

### การสร้าง Pattern Fill

```pascal
unit PatternFillDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, LCLType;

type
  TForm1 = class(TForm)
    procedure FormPaint(Sender: TObject);
  private
    procedure DrawHatchPattern(ACanvas: TCanvas; ARect: TRect; 
                                PatternType: Integer; FGColor, BGColor: TColor);
    procedure DrawCustomPattern(ACanvas: TCanvas; ARect: TRect);
    procedure DrawTexturePattern(ACanvas: TCanvas; ARect: TRect);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.DrawHatchPattern(ACanvas: TCanvas; ARect: TRect;
  PatternType: Integer; FGColor, BGColor: TColor);
var
  PatternBmp: TBitmap;
  i, j: Integer;
begin
  PatternBmp := TBitmap.Create;
  try
    PatternBmp.Width := 8;
    PatternBmp.Height := 8;
    PatternBmp.Canvas.Brush.Color := BGColor;
    PatternBmp.Canvas.FillRect(Rect(0, 0, 8, 8));
    
    case PatternType of
      0: // เส้นนอน
      begin
        PatternBmp.Canvas.Pen.Color := FGColor;
        PatternBmp.Canvas.MoveTo(0, 4);
        PatternBmp.Canvas.LineTo(8, 4);
      end;
      1: // เส้นตั้ง
      begin
        PatternBmp.Canvas.Pen.Color := FGColor;
        PatternBmp.Canvas.MoveTo(4, 0);
        PatternBmp.Canvas.LineTo(4, 8);
      end;
      2: // ตาราง
      begin
        PatternBmp.Canvas.Pen.Color := FGColor;
        PatternBmp.Canvas.MoveTo(0, 4);
        PatternBmp.Canvas.LineTo(8, 4);
        PatternBmp.Canvas.MoveTo(4, 0);
        PatternBmp.Canvas.LineTo(4, 8);
      end;
      3: // จุด
      begin
        for i := 0 to 1 do
          for j := 0 to 1 do
            PatternBmp.Canvas.Pixels[i * 4, j * 4] := FGColor;
      end;
      4: // เส้นทแยง
      begin
        PatternBmp.Canvas.Pen.Color := FGColor;
        PatternBmp.Canvas.MoveTo(0, 0);
        PatternBmp.Canvas.LineTo(8, 8);
      end;
    end;
    
    // วาด Pattern ซ้ำ
    var x, y: Integer;
    for y := ARect.Top to ARect.Bottom - 1 do
      for x := ARect.Left to ARect.Right - 1 do
        ACanvas.Pixels[x, y] := PatternBmp.Canvas.Pixels[
          (x - ARect.Left) mod 8, (y - ARect.Top) mod 8];
          
  finally
    PatternBmp.Free;
  end;
end;

procedure TForm1.DrawCustomPattern(ACanvas: TCanvas; ARect: TRect);
// สร้าง Pattern แบบ Checkerboard
var
  x, y: Integer;
begin
  for y := ARect.Top to ARect.Bottom - 1 do
    for x := ARect.Left to ARect.Right - 1 do
    begin
      if ((x div 10) + (y div 10)) mod 2 = 0 then
        ACanvas.Pixels[x, y] := clBlue
      else
        ACanvas.Pixels[x, y] := clWhite;
    end;
end;

procedure TForm1.DrawTexturePattern(ACanvas: TCanvas; ARect: TRect);
// สร้าง Texture แบบหินอ่อน (Marble effect)
var
  x, y: Integer;
  v: Double;
begin
  for y := ARect.Top to ARect.Bottom - 1 do
    for x := ARect.Left to ARect.Right - 1 do
    begin
      v := Abs(Sin((x + y) * 0.05 + Sin(x * 0.1) * 5));
      var r := Round(100 + v * 155);
      var g := Round(50 + v * 100);
      var b := Round(150 + v * 105);
      ACanvas.Pixels[x, y] := RGB(r, g, b);
    end;
end;

procedure TForm1.FormPaint(Sender: TObject);
var
  Titles: array[0..4] of string = ('Horizontal', 'Vertical', 'Cross', 'Dots', 'Diagonal');
  i: Integer;
begin
  Canvas.Brush.Color := clWhite;
  Canvas.FillRect(ClientRect);
  
  Canvas.Font.Size := 10;
  
  // แสดง Pattern ต่างๆ
  for i := 0 to 4 do
  begin
    Canvas.Pen.Color := clBlack;
    Canvas.Pen.Width := 1;
    Canvas.Rectangle(10 + i * 100, 10, 90 + i * 100, 80);
    DrawHatchPattern(Canvas, Rect(11 + i * 100, 11, 89 + i * 100, 79),
                      i, clBlue, clWhite);
    Canvas.TextOut(10 + i * 100, 85, Titles[i]);
  end;
  
  // Pattern แบบ Checkerboard
  Canvas.Pen.Color := clBlack;
  Canvas.Rectangle(10, 110, 200, 200);
  DrawCustomPattern(Canvas, Rect(11, 111, 199, 199));
  Canvas.TextOut(10, 205, 'Checkerboard');
  
  // Texture แบบหินอ่อน
  Canvas.Pen.Color := clBlack;
  Canvas.Rectangle(220, 110, 450, 200);
  DrawTexturePattern(Canvas, Rect(221, 111, 449, 199));
  Canvas.TextOut(220, 205, 'Marble Texture');
end;

end.
```

---

## 36.9 Double Buffering และการ Optimize

Double Buffering ป้องกัน flickering ในการ animation โดยวาดลง Bitmap ก่อนแล้วค่อย copy ไป Screen

```pascal
unit DoubleBufferDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls;

type
  TForm1 = class(TForm)
    Timer1: TTimer;
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
    procedure FormResize(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
  private
    FBackBuffer: TBitmap;
    FBallX, FBallY: Double;
    FBallVX, FBallVY: Double;
    FBallRadius: Integer;
    procedure RenderToBuffer;
    procedure UpdateBallPosition;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  // ปิด default double buffering ของ Form
  DoubleBuffered := True;  // หรือจัดการเอง
  
  FBackBuffer := TBitmap.Create;
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  
  // กำหนดค่าเริ่มต้นของลูกบอล
  FBallX := 100;
  FBallY := 100;
  FBallVX := 3;
  FBallVY := 2;
  FBallRadius := 20;
  
  Timer1.Interval := 16;  // ~60 FPS
  Timer1.Enabled := True;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FBackBuffer.Free;
end;

procedure TForm1.FormResize(Sender: TObject);
begin
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
end;

procedure TForm1.UpdateBallPosition;
begin
  FBallX := FBallX + FBallVX;
  FBallY := FBallY + FBallVY;
  
  // Bounce off walls
  if (FBallX - FBallRadius < 0) or (FBallX + FBallRadius > ClientWidth) then
    FBallVX := -FBallVX;
  if (FBallY - FBallRadius < 0) or (FBallY + FBallRadius > ClientHeight) then
    FBallVY := -FBallVY;
    
  // Keep in bounds
  FBallX := Max(FBallRadius, Min(ClientWidth - FBallRadius, FBallX));
  FBallY := Max(FBallRadius, Min(ClientHeight - FBallRadius, FBallY));
end;

procedure TForm1.RenderToBuffer;
var
  BC: TCanvas;
  CenterX, CenterY: Integer;
begin
  BC := FBackBuffer.Canvas;
  
  // วาดพื้นหลัง
  BC.Brush.Color := clBlack;
  BC.FillRect(Rect(0, 0, FBackBuffer.Width, FBackBuffer.Height));
  
  // วาดตาราง
  BC.Pen.Color := TColor(RGB(30, 30, 30));
  BC.Pen.Width := 1;
  var x, y: Integer;
  for x := 0 to FBackBuffer.Width div 20 do
  begin
    BC.MoveTo(x * 20, 0);
    BC.LineTo(x * 20, FBackBuffer.Height);
  end;
  for y := 0 to FBackBuffer.Height div 20 do
  begin
    BC.MoveTo(0, y * 20);
    BC.LineTo(FBackBuffer.Width, y * 20);
  end;
  
  // วาดลูกบอลพร้อม Gradient effect
  CenterX := Round(FBallX);
  CenterY := Round(FBallY);
  
  // Shadow
  BC.Brush.Color := TColor(RGB(20, 20, 20));
  BC.Pen.Style := psNone;
  BC.Ellipse(CenterX - FBallRadius + 4, CenterY - FBallRadius + 4,
              CenterX + FBallRadius + 4, CenterY + FBallRadius + 4);
  
  // Ball body
  BC.Brush.Color := clRed;
  BC.Pen.Style := psSolid;
  BC.Pen.Color := clDarkRed;
  BC.Pen.Width := 2;
  BC.Ellipse(CenterX - FBallRadius, CenterY - FBallRadius,
              CenterX + FBallRadius, CenterY + FBallRadius);
  
  // Highlight
  BC.Brush.Color := TColor(RGB(255, 150, 150));
  BC.Pen.Style := psNone;
  BC.Ellipse(CenterX - FBallRadius div 3, CenterY - FBallRadius div 2,
              CenterX, CenterY - FBallRadius div 6);
  
  // FPS counter
  BC.Font.Color := clWhite;
  BC.Font.Size := 10;
  BC.Brush.Style := bsClear;
  BC.TextOut(5, 5, Format('Ball: (%.0f, %.0f)', [FBallX, FBallY]));
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  UpdateBallPosition;
  RenderToBuffer;
  Invalidate;
end;

procedure TForm1.FormPaint(Sender: TObject);
begin
  // Copy buffer ไปยัง Screen
  Canvas.Draw(0, 0, FBackBuffer);
end;

end.
```

---

## 36.10 Animation ด้วย Timer

```pascal
unit AnimationDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls, Math;

type
  TParticle = record
    X, Y: Double;
    VX, VY: Double;
    Life: Double;
    MaxLife: Double;
    Color: TColor;
    Size: Integer;
  end;
  
  TForm1 = class(TForm)
    Timer1: TTimer;
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
    procedure FormMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
    procedure Timer1Timer(Sender: TObject);
  private
    FBackBuffer: TBitmap;
    FParticles: array of TParticle;
    FMouseX, FMouseY: Integer;
    FTime: Double;
    procedure SpawnParticles;
    procedure UpdateParticles;
    procedure RenderFrame;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  DoubleBuffered := True;
  FBackBuffer := TBitmap.Create;
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  SetLength(FParticles, 0);
  FMouseX := ClientWidth div 2;
  FMouseY := ClientHeight div 2;
  FTime := 0;
  Timer1.Interval := 16;
  Timer1.Enabled := True;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FBackBuffer.Free;
end;

procedure TForm1.FormMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
begin
  FMouseX := X;
  FMouseY := Y;
end;

procedure TForm1.SpawnParticles;
var
  i, NewCount: Integer;
  p: TParticle;
  Angle, Speed: Double;
begin
  NewCount := 5;
  for i := 0 to NewCount - 1 do
  begin
    Angle := Random * 2 * Pi;
    Speed := 1 + Random * 3;
    
    p.X := FMouseX;
    p.Y := FMouseY;
    p.VX := Cos(Angle) * Speed;
    p.VY := Sin(Angle) * Speed - 2;  // Initial upward velocity
    p.MaxLife := 30 + Random * 60;
    p.Life := p.MaxLife;
    p.Size := 3 + Random(5);
    
    // สีแบบ rainbow
    var hue := (FTime * 0.01 + i * 0.1);
    var r, g, b: Byte;
    var h := Frac(hue) * 6;
    var sector := Trunc(h);
    var f := h - sector;
    case sector mod 6 of
      0: begin r := 255; g := Round(255 * f); b := 0; end;
      1: begin r := Round(255 * (1 - f)); g := 255; b := 0; end;
      2: begin r := 0; g := 255; b := Round(255 * f); end;
      3: begin r := 0; g := Round(255 * (1 - f)); b := 255; end;
      4: begin r := Round(255 * f); g := 0; b := 255; end;
      else begin r := 255; g := 0; b := Round(255 * (1 - f)); end;
    end;
    p.Color := RGB(r, g, b);
    
    SetLength(FParticles, Length(FParticles) + 1);
    FParticles[High(FParticles)] := p;
  end;
end;

procedure TForm1.UpdateParticles;
var
  i, j: Integer;
begin
  j := 0;
  for i := 0 to High(FParticles) do
  begin
    FParticles[i].VY := FParticles[i].VY + 0.1;  // Gravity
    FParticles[i].X := FParticles[i].X + FParticles[i].VX;
    FParticles[i].Y := FParticles[i].Y + FParticles[i].VY;
    FParticles[i].Life := FParticles[i].Life - 1;
    
    if FParticles[i].Life > 0 then
    begin
      FParticles[j] := FParticles[i];
      Inc(j);
    end;
  end;
  SetLength(FParticles, j);
end;

procedure TForm1.RenderFrame;
var
  BC: TCanvas;
  i: Integer;
  Alpha: Double;
  R, G, B: Byte;
begin
  BC := FBackBuffer.Canvas;
  
  // Fade effect - ไม่ลบพื้นหลังทั้งหมด แต่ใส่ alpha
  BC.Brush.Color := TColor(RGB(10, 10, 20));
  BC.FillRect(Rect(0, 0, FBackBuffer.Width, FBackBuffer.Height));
  
  // วาด Particles
  for i := 0 to High(FParticles) do
  begin
    Alpha := FParticles[i].Life / FParticles[i].MaxLife;
    
    R := GetRValue(FParticles[i].Color);
    G := GetGValue(FParticles[i].Color);
    B := GetBValue(FParticles[i].Color);
    
    BC.Brush.Color := RGB(Round(R * Alpha), Round(G * Alpha), Round(B * Alpha));
    BC.Pen.Style := psNone;
    
    var S := Round(FParticles[i].Size * Alpha);
    if S < 1 then S := 1;
    BC.Ellipse(
      Round(FParticles[i].X) - S,
      Round(FParticles[i].Y) - S,
      Round(FParticles[i].X) + S,
      Round(FParticles[i].Y) + S
    );
  end;
  
  // แสดงจำนวน Particles
  BC.Font.Color := clWhite;
  BC.Font.Size := 10;
  BC.Brush.Style := bsClear;
  BC.TextOut(5, 5, Format('Particles: %d', [Length(FParticles)]));
  BC.TextOut(5, 20, 'เลื่อนเมาส์เพื่อสร้าง Particles');
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  FTime := FTime + 1;
  SpawnParticles;
  UpdateParticles;
  RenderFrame;
  Invalidate;
end;

procedure TForm1.FormPaint(Sender: TObject);
begin
  Canvas.Draw(0, 0, FBackBuffer);
end;

end.
```

---

## 36.11 Sprite Animation

```pascal
unit SpriteAnimation;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls, LCLType;

type
  TSprite = class
  private
    FSpriteSheet: TBitmap;
    FFrameWidth, FFrameHeight: Integer;
    FCurrentFrame: Integer;
    FFrameCount: Integer;
    FX, FY: Double;
    FVX, FVY: Double;
    FAnimSpeed: Integer;
    FAnimCounter: Integer;
    FRow: Integer;
  public
    constructor Create(AWidth, AHeight: Integer; AFrameCount: Integer);
    destructor Destroy; override;
    procedure Update;
    procedure Draw(ACanvas: TCanvas);
    procedure CreateTestSprite;
    property X: Double read FX write FX;
    property Y: Double read FY write FY;
    property VX: Double read FVX write FVX;
    property VY: Double read FVY write FVY;
    property Row: Integer read FRow write FRow;
  end;
  
  TForm1 = class(TForm)
    Timer1: TTimer;
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure FormKeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
  private
    FSprite: TSprite;
    FBackBuffer: TBitmap;
    FKeys: array[0..255] of Boolean;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

constructor TSprite.Create(AWidth, AHeight: Integer; AFrameCount: Integer);
begin
  inherited Create;
  FFrameWidth := AWidth;
  FFrameHeight := AHeight;
  FFrameCount := AFrameCount;
  FCurrentFrame := 0;
  FAnimSpeed := 5;
  FAnimCounter := 0;
  FRow := 0;
  
  FSpriteSheet := TBitmap.Create;
  FSpriteSheet.Width := AWidth * AFrameCount;
  FSpriteSheet.Height := AHeight * 4;  // 4 rows: down, left, right, up
end;

destructor TSprite.Destroy;
begin
  FSpriteSheet.Free;
  inherited;
end;

procedure TSprite.CreateTestSprite;
// สร้าง Sprite แบบ placeholder
var
  Frame, Row: Integer;
  Colors: array[0..3] of TColor = (clBlue, clGreen, clRed, clYellow);
begin
  for Row := 0 to 3 do
    for Frame := 0 to FFrameCount - 1 do
    begin
      var FC := FSpriteSheet.Canvas;
      FC.Brush.Color := Colors[Row];
      FC.FillRect(Rect(Frame * FFrameWidth, Row * FFrameHeight,
                        (Frame + 1) * FFrameWidth - 1, (Row + 1) * FFrameHeight - 1));
      FC.Pen.Color := clBlack;
      FC.Pen.Width := 1;
      FC.Rectangle(Frame * FFrameWidth, Row * FFrameHeight,
                    (Frame + 1) * FFrameWidth - 1, (Row + 1) * FFrameHeight - 1);
      
      // วาดหน้าตา
      FC.Brush.Color := clWhite;
      FC.Ellipse(Frame * FFrameWidth + 8, Row * FFrameHeight + 5,
                  Frame * FFrameWidth + 24, Row * FFrameHeight + 21);
      FC.Brush.Color := clBlack;
      
      // ตา
      var EyeOffset := (Frame mod 2) * 2;
      FC.Ellipse(Frame * FFrameWidth + 9 + EyeOffset, Row * FFrameHeight + 8,
                  Frame * FFrameWidth + 14 + EyeOffset, Row * FFrameHeight + 13);
      
      // ขา
      FC.Pen.Color := clBlack;
      FC.Pen.Width := 2;
      var LegOffset := (Frame mod 2 = 0);
      if LegOffset then
      begin
        FC.MoveTo(Frame * FFrameWidth + 10, Row * FFrameHeight + 28);
        FC.LineTo(Frame * FFrameWidth + 7, Row * FFrameHeight + 36);
        FC.MoveTo(Frame * FFrameWidth + 22, Row * FFrameHeight + 28);
        FC.LineTo(Frame * FFrameWidth + 25, Row * FFrameHeight + 36);
      end else
      begin
        FC.MoveTo(Frame * FFrameWidth + 10, Row * FFrameHeight + 28);
        FC.LineTo(Frame * FFrameWidth + 13, Row * FFrameHeight + 36);
        FC.MoveTo(Frame * FFrameWidth + 22, Row * FFrameHeight + 28);
        FC.LineTo(Frame * FFrameWidth + 19, Row * FFrameHeight + 36);
      end;
    end;
end;

procedure TSprite.Update;
begin
  FX := FX + FVX;
  FY := FY + FVY;
  
  Inc(FAnimCounter);
  if FAnimCounter >= FAnimSpeed then
  begin
    FAnimCounter := 0;
    FCurrentFrame := (FCurrentFrame + 1) mod FFrameCount;
  end;
end;

procedure TSprite.Draw(ACanvas: TCanvas);
var
  SrcRect, DstRect: TRect;
begin
  SrcRect := Rect(FCurrentFrame * FFrameWidth, FRow * FFrameHeight,
                   (FCurrentFrame + 1) * FFrameWidth, (FRow + 1) * FFrameHeight);
  DstRect := Rect(Round(FX), Round(FY), 
                   Round(FX) + FFrameWidth, Round(FY) + FFrameHeight);
  
  ACanvas.CopyRect(DstRect, FSpriteSheet.Canvas, SrcRect);
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  DoubleBuffered := True;
  FBackBuffer := TBitmap.Create;
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  
  FSprite := TSprite.Create(32, 40, 4);
  FSprite.CreateTestSprite;
  FSprite.X := 200;
  FSprite.Y := 200;
  
  FillChar(FKeys, SizeOf(FKeys), 0);
  
  Timer1.Interval := 16;
  Timer1.Enabled := True;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FSprite.Free;
  FBackBuffer.Free;
end;

procedure TForm1.FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  if Key < 256 then FKeys[Key] := True;
end;

procedure TForm1.FormKeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  if Key < 256 then FKeys[Key] := False;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
const
  Speed = 3;
var
  Moving: Boolean;
begin
  Moving := False;
  FSprite.VX := 0;
  FSprite.VY := 0;
  
  if FKeys[VK_UP] or FKeys[Ord('W')] then
  begin
    FSprite.VY := -Speed;
    FSprite.Row := 3;  // Up row
    Moving := True;
  end;
  if FKeys[VK_DOWN] or FKeys[Ord('S')] then
  begin
    FSprite.VY := Speed;
    FSprite.Row := 0;  // Down row
    Moving := True;
  end;
  if FKeys[VK_LEFT] or FKeys[Ord('A')] then
  begin
    FSprite.VX := -Speed;
    FSprite.Row := 1;  // Left row
    Moving := True;
  end;
  if FKeys[VK_RIGHT] or FKeys[Ord('D')] then
  begin
    FSprite.VX := Speed;
    FSprite.Row := 2;  // Right row
    Moving := True;
  end;
  
  if not Moving then
    FSprite.VX := 0;
  
  FSprite.Update;
  
  // Boundary check
  FSprite.X := Max(0, Min(ClientWidth - 32, FSprite.X));
  FSprite.Y := Max(0, Min(ClientHeight - 40, FSprite.Y));
  
  // Render
  FBackBuffer.Canvas.Brush.Color := TColor(RGB(34, 139, 34));
  FBackBuffer.Canvas.FillRect(Rect(0, 0, ClientWidth, ClientHeight));
  FSprite.Draw(FBackBuffer.Canvas);
  
  FBackBuffer.Canvas.Font.Color := clWhite;
  FBackBuffer.Canvas.Font.Size := 10;
  FBackBuffer.Canvas.Brush.Style := bsClear;
  FBackBuffer.Canvas.TextOut(5, 5, 'ใช้ WASD หรือ Arrow Keys เพื่อเคลื่อนที่');
  
  Invalidate;
end;

procedure TForm1.FormPaint(Sender: TObject);
begin
  Canvas.Draw(0, 0, FBackBuffer);
end;

end.
```

---

## 36.12 ตัวอย่างสมบูรณ์: นาฬิกาอนาล็อก

```pascal
unit AnalogClock;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls, Math, DateUtils;

type
  TForm1 = class(TForm)
    Timer1: TTimer;
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
    procedure FormResize(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
  private
    FBackBuffer: TBitmap;
    FCenterX, FCenterY: Integer;
    FRadius: Integer;
    procedure DrawClockFace;
    procedure DrawHands;
    procedure DrawHand(ACanvas: TCanvas; AngleRad: Double; 
                        Length: Integer; Width: Integer; AColor: TColor);
    procedure DrawTick(ACanvas: TCanvas; AngleRad: Double; 
                        IsHour: Boolean);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  DoubleBuffered := True;
  FBackBuffer := TBitmap.Create;
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  
  Timer1.Interval := 1000;
  Timer1.Enabled := True;
  
  FormResize(nil);
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FBackBuffer.Free;
end;

procedure TForm1.FormResize(Sender: TObject);
begin
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  FCenterX := ClientWidth div 2;
  FCenterY := ClientHeight div 2;
  FRadius := Min(ClientWidth, ClientHeight) div 2 - 10;
  Invalidate;
end;

procedure TForm1.DrawClockFace;
var
  BC: TCanvas;
  i: Integer;
begin
  BC := FBackBuffer.Canvas;
  
  // พื้นหลัง
  BC.Brush.Color := TColor(RGB(20, 20, 40));
  BC.FillRect(Rect(0, 0, FBackBuffer.Width, FBackBuffer.Height));
  
  // วงนอก - Shadow
  BC.Pen.Style := psNone;
  BC.Brush.Color := TColor(RGB(0, 0, 0));
  BC.Ellipse(FCenterX - FRadius - 3 + 4,
              FCenterY - FRadius - 3 + 4,
              FCenterX + FRadius + 3 + 4,
              FCenterY + FRadius + 3 + 4);
  
  // วงนอก gradient effect
  BC.Pen.Color := TColor(RGB(200, 180, 100));
  BC.Pen.Width := 4;
  BC.Brush.Color := TColor(RGB(40, 40, 60));
  BC.Ellipse(FCenterX - FRadius - 3,
              FCenterY - FRadius - 3,
              FCenterX + FRadius + 3,
              FCenterY + FRadius + 3);
  
  // หน้าปัด
  BC.Pen.Style := psNone;
  BC.Brush.Color := TColor(RGB(240, 235, 220));
  BC.Ellipse(FCenterX - FRadius, FCenterY - FRadius,
              FCenterX + FRadius, FCenterY + FRadius);
  
  // Tick marks
  for i := 0 to 59 do
    DrawTick(BC, i * Pi / 30 - Pi / 2, i mod 5 = 0);
  
  // ตัวเลข
  BC.Font.Name := 'Arial';
  BC.Font.Style := [fsBold];
  BC.Font.Color := TColor(RGB(20, 20, 50));
  BC.Brush.Style := bsClear;
  
  var NumR := FRadius - 30;
  for i := 1 to 12 do
  begin
    var Angle := i * Pi / 6 - Pi / 2;
    var TX := FCenterX + Round(NumR * Cos(Angle));
    var TY := FCenterY + Round(NumR * Sin(Angle));
    BC.Font.Size := Max(8, FRadius div 10);
    var S := IntToStr(i);
    var TW := BC.TextWidth(S);
    var TH := BC.TextHeight(S);
    BC.TextOut(TX - TW div 2, TY - TH div 2, S);
  end;
  
  // Center dot
  BC.Brush.Color := TColor(RGB(180, 150, 50));
  BC.Pen.Style := psNone;
  BC.Ellipse(FCenterX - 6, FCenterY - 6, FCenterX + 6, FCenterY + 6);
end;

procedure TForm1.DrawTick(ACanvas: TCanvas; AngleRad: Double; IsHour: Boolean);
var
  OutR, InR: Integer;
  X1, Y1, X2, Y2: Integer;
begin
  if IsHour then
  begin
    OutR := FRadius - 2;
    InR := FRadius - 15;
    ACanvas.Pen.Width := 3;
    ACanvas.Pen.Color := TColor(RGB(50, 50, 80));
  end else
  begin
    OutR := FRadius - 2;
    InR := FRadius - 8;
    ACanvas.Pen.Width := 1;
    ACanvas.Pen.Color := TColor(RGB(150, 140, 100));
  end;
  
  X1 := FCenterX + Round(OutR * Cos(AngleRad));
  Y1 := FCenterY + Round(OutR * Sin(AngleRad));
  X2 := FCenterX + Round(InR * Cos(AngleRad));
  Y2 := FCenterY + Round(InR * Sin(AngleRad));
  
  ACanvas.MoveTo(X1, Y1);
  ACanvas.LineTo(X2, Y2);
end;

procedure TForm1.DrawHand(ACanvas: TCanvas; AngleRad: Double;
  Length: Integer; Width: Integer; AColor: TColor);
var
  EndX, EndY: Integer;
  BaseAngle: Double;
  Points: array[0..3] of TPoint;
begin
  EndX := FCenterX + Round(Length * Cos(AngleRad));
  EndY := FCenterY + Round(Length * Sin(AngleRad));
  
  // วาดเงา
  ACanvas.Pen.Color := TColor(RGB(0, 0, 0));
  ACanvas.Pen.Width := Width + 2;
  ACanvas.MoveTo(FCenterX + 2, FCenterY + 2);
  ACanvas.LineTo(EndX + 2, EndY + 2);
  
  // วาดเข็ม
  ACanvas.Pen.Color := AColor;
  ACanvas.Pen.Width := Width;
  ACanvas.MoveTo(FCenterX, FCenterY);
  ACanvas.LineTo(EndX, EndY);
  
  // วาดส่วนตรงข้าม (counterbalance)
  var BackX := FCenterX + Round(Length * 0.15 * Cos(AngleRad + Pi));
  var BackY := FCenterY + Round(Length * 0.15 * Sin(AngleRad + Pi));
  ACanvas.Pen.Width := Width + 2;
  ACanvas.MoveTo(FCenterX, FCenterY);
  ACanvas.LineTo(BackX, BackY);
end;

procedure TForm1.DrawHands;
var
  Now: TDateTime;
  H, M, S, MS: Word;
  HAngle, MAngle, SAngle: Double;
begin
  DecodeTime(Now, H, M, S, MS);
  Now := Time;
  DecodeTime(Now, H, M, S, MS);
  
  // คำนวณมุม (เริ่มจาก 12 นาฬิกา = -Pi/2)
  HAngle := (H mod 12) * Pi / 6 + M * Pi / 360 - Pi / 2;
  MAngle := M * Pi / 30 + S * Pi / 1800 - Pi / 2;
  SAngle := S * Pi / 30 - Pi / 2;
  
  // วาดเข็มชั่วโมง
  DrawHand(FBackBuffer.Canvas, HAngle, 
           Round(FRadius * 0.55), 5, TColor(RGB(20, 20, 50)));
  
  // วาดเข็มนาที
  DrawHand(FBackBuffer.Canvas, MAngle,
           Round(FRadius * 0.75), 3, TColor(RGB(20, 20, 80)));
  
  // วาดเข็มวินาที
  DrawHand(FBackBuffer.Canvas, SAngle,
           Round(FRadius * 0.85), 1, TColor(RGB(220, 30, 30)));
  
  // วาดจุดกลาง
  FBackBuffer.Canvas.Brush.Color := TColor(RGB(220, 30, 30));
  FBackBuffer.Canvas.Pen.Style := psNone;
  FBackBuffer.Canvas.Ellipse(FCenterX - 4, FCenterY - 4, 
                               FCenterX + 4, FCenterY + 4);
  
  // แสดงเวลาดิจิทัล
  FBackBuffer.Canvas.Brush.Style := bsClear;
  FBackBuffer.Canvas.Font.Color := TColor(RGB(80, 80, 120));
  FBackBuffer.Canvas.Font.Size := Max(8, FRadius div 12);
  FBackBuffer.Canvas.Font.Style := [];
  var TimeStr := FormatDateTime('HH:MM:SS', Now);
  var TW := FBackBuffer.Canvas.TextWidth(TimeStr);
  FBackBuffer.Canvas.TextOut(FCenterX - TW div 2, 
                               FCenterY + FRadius div 3, TimeStr);
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  DrawClockFace;
  DrawHands;
  Invalidate;
end;

procedure TForm1.FormPaint(Sender: TObject);
begin
  if not Assigned(FBackBuffer) then Exit;
  DrawClockFace;
  DrawHands;
  Canvas.Draw(0, 0, FBackBuffer);
end;

end.
```

---

## 36.13 ตัวอย่างสมบูรณ์: เกม 2D อย่างง่าย (Breakout)

```pascal
unit BreakoutGame;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls, LCLType, Math;

const
  BRICK_ROWS = 5;
  BRICK_COLS = 10;
  BRICK_WIDTH = 56;
  BRICK_HEIGHT = 20;
  BRICK_MARGIN = 4;
  BRICK_TOP = 60;
  
  PADDLE_WIDTH = 80;
  PADDLE_HEIGHT = 12;
  PADDLE_SPEED = 6;
  
  BALL_RADIUS = 8;
  BALL_SPEED = 5;

type
  TBrick = record
    Active: Boolean;
    Color: TColor;
    Hits: Integer;
    MaxHits: Integer;
  end;
  
  TGameState = (gsMenu, gsPlaying, gsPaused, gsGameOver, gsWin);
  
  TForm1 = class(TForm)
    Timer1: TTimer;
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure FormKeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure Timer1Timer(Sender: TObject);
    procedure FormMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
    procedure FormClick(Sender: TObject);
  private
    FBackBuffer: TBitmap;
    FBricks: array[0..BRICK_ROWS-1, 0..BRICK_COLS-1] of TBrick;
    FPaddleX: Double;
    FBallX, FBallY: Double;
    FBallVX, FBallVY: Double;
    FScore, FLives: Integer;
    FGameState: TGameState;
    FKeys: array[0..255] of Boolean;
    FBrickCount: Integer;
    
    procedure InitGame;
    procedure InitBricks;
    procedure Update;
    procedure RenderFrame;
    procedure DrawBrick(ACanvas: TCanvas; Col, Row: Integer);
    procedure DrawPaddle(ACanvas: TCanvas);
    procedure DrawBall(ACanvas: TCanvas);
    procedure DrawHUD(ACanvas: TCanvas);
    procedure DrawMenu(ACanvas: TCanvas);
    procedure DrawGameOver(ACanvas: TCanvas);
    function BrickRect(Col, Row: Integer): TRect;
    procedure CheckBrickCollision;
    procedure AddParticles(X, Y: Integer; AColor: TColor);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  DoubleBuffered := True;
  ClientWidth := BRICK_COLS * (BRICK_WIDTH + BRICK_MARGIN) + BRICK_MARGIN;
  ClientHeight := 500;
  
  FBackBuffer := TBitmap.Create;
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  
  FillChar(FKeys, SizeOf(FKeys), 0);
  
  InitGame;
  
  Timer1.Interval := 16;
  Timer1.Enabled := True;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FBackBuffer.Free;
end;

procedure TForm1.InitBricks;
const
  BrickColors: array[0..BRICK_ROWS-1] of TColor = (
    $0000FF,  // แดง
    $0080FF,  // ส้ม
    $00FFFF,  // เหลือง
    $00FF00,  // เขียว
    $FF0000   // น้ำเงิน
  );
  BrickHits: array[0..BRICK_ROWS-1] of Integer = (2, 2, 1, 1, 1);
var
  Row, Col: Integer;
begin
  FBrickCount := 0;
  for Row := 0 to BRICK_ROWS - 1 do
    for Col := 0 to BRICK_COLS - 1 do
    begin
      FBricks[Row][Col].Active := True;
      FBricks[Row][Col].Color := BrickColors[Row];
      FBricks[Row][Col].MaxHits := BrickHits[Row];
      FBricks[Row][Col].Hits := BrickHits[Row];
      Inc(FBrickCount);
    end;
end;

procedure TForm1.InitGame;
begin
  FPaddleX := ClientWidth / 2 - PADDLE_WIDTH / 2;
  FBallX := ClientWidth / 2;
  FBallY := ClientHeight - 80;
  FBallVX := BALL_SPEED * (Random(2) * 2 - 1);
  FBallVY := -BALL_SPEED;
  FScore := 0;
  FLives := 3;
  FGameState := gsPlaying;
  InitBricks;
end;

function TForm1.BrickRect(Col, Row: Integer): TRect;
begin
  Result.Left := BRICK_MARGIN + Col * (BRICK_WIDTH + BRICK_MARGIN);
  Result.Top := BRICK_TOP + Row * (BRICK_HEIGHT + BRICK_MARGIN);
  Result.Right := Result.Left + BRICK_WIDTH;
  Result.Bottom := Result.Top + BRICK_HEIGHT;
end;

procedure TForm1.CheckBrickCollision;
var
  Row, Col: Integer;
  BR: TRect;
begin
  for Row := 0 to BRICK_ROWS - 1 do
    for Col := 0 to BRICK_COLS - 1 do
    begin
      if not FBricks[Row][Col].Active then Continue;
      
      BR := BrickRect(Col, Row);
      
      if (FBallX + BALL_RADIUS >= BR.Left) and
         (FBallX - BALL_RADIUS <= BR.Right) and
         (FBallY + BALL_RADIUS >= BR.Top) and
         (FBallY - BALL_RADIUS <= BR.Bottom) then
      begin
        // Hit!
        Dec(FBricks[Row][Col].Hits);
        if FBricks[Row][Col].Hits <= 0 then
        begin
          FBricks[Row][Col].Active := False;
          Dec(FBrickCount);
          FScore := FScore + (Row + 1) * 10;
          AddParticles(Round((BR.Left + BR.Right) / 2),
                       Round((BR.Top + BR.Bottom) / 2),
                       FBricks[Row][Col].Color);
        end;
        
        // Determine bounce direction
        var OverlapLeft := (FBallX + BALL_RADIUS) - BR.Left;
        var OverlapRight := BR.Right - (FBallX - BALL_RADIUS);
        var OverlapTop := (FBallY + BALL_RADIUS) - BR.Top;
        var OverlapBottom := BR.Bottom - (FBallY - BALL_RADIUS);
        
        var MinOverlap := Min(Min(OverlapLeft, OverlapRight),
                               Min(OverlapTop, OverlapBottom));
        
        if MinOverlap = OverlapTop then FBallVY := Abs(FBallVY)
        else if MinOverlap = OverlapBottom then FBallVY := -Abs(FBallVY)
        else if MinOverlap = OverlapLeft then FBallVX := Abs(FBallVX)
        else FBallVX := -Abs(FBallVX);
        
        Exit;  // Hit only one brick per frame
      end;
    end;
end;

procedure TForm1.AddParticles(X, Y: Integer; AColor: TColor);
begin
  // Simplified - just flash effect
  // In full game, would use particle system
end;

procedure TForm1.Update;
var
  PaddleRect: TRect;
begin
  if FGameState <> gsPlaying then Exit;
  
  // ควบคุม Paddle
  if FKeys[VK_LEFT] or FKeys[Ord('A')] then
    FPaddleX := Max(0, FPaddleX - PADDLE_SPEED);
  if FKeys[VK_RIGHT] or FKeys[Ord('D')] then
    FPaddleX := Min(ClientWidth - PADDLE_WIDTH, FPaddleX + PADDLE_SPEED);
  
  // อัปเดตตำแหน่งลูก
  FBallX := FBallX + FBallVX;
  FBallY := FBallY + FBallVY;
  
  // ชนกำแพง
  if FBallX - BALL_RADIUS < 0 then
  begin
    FBallX := BALL_RADIUS;
    FBallVX := Abs(FBallVX);
  end;
  if FBallX + BALL_RADIUS > ClientWidth then
  begin
    FBallX := ClientWidth - BALL_RADIUS;
    FBallVX := -Abs(FBallVX);
  end;
  if FBallY - BALL_RADIUS < 0 then
  begin
    FBallY := BALL_RADIUS;
    FBallVY := Abs(FBallVY);
  end;
  
  // ชน Paddle
  PaddleRect := Rect(Round(FPaddleX), ClientHeight - 50,
                      Round(FPaddleX) + PADDLE_WIDTH, ClientHeight - 50 + PADDLE_HEIGHT);
  if (FBallY + BALL_RADIUS >= PaddleRect.Top) and
     (FBallY - BALL_RADIUS <= PaddleRect.Bottom) and
     (FBallX >= PaddleRect.Left) and
     (FBallX <= PaddleRect.Right) then
  begin
    FBallVY := -Abs(FBallVY);
    // ปรับทิศทางตำแหน่งที่ชน Paddle
    var HitPos := (FBallX - FPaddleX) / PADDLE_WIDTH - 0.5;  // -0.5 to 0.5
    FBallVX := HitPos * BALL_SPEED * 2;
    // Normalize speed
    var Speed := Sqrt(FBallVX * FBallVX + FBallVY * FBallVY);
    FBallVX := FBallVX / Speed * BALL_SPEED;
    FBallVY := FBallVY / Speed * BALL_SPEED;
    FBallVY := -Abs(FBallVY);
  end;
  
  // ตกลงด้านล่าง
  if FBallY > ClientHeight + BALL_RADIUS then
  begin
    Dec(FLives);
    if FLives <= 0 then
      FGameState := gsGameOver
    else
    begin
      FBallX := ClientWidth / 2;
      FBallY := ClientHeight - 80;
      FBallVX := BALL_SPEED * (Random(2) * 2 - 1);
      FBallVY := -BALL_SPEED;
    end;
  end;
  
  // ชนอิฐ
  CheckBrickCollision;
  
  // ชนะ
  if FBrickCount <= 0 then
    FGameState := gsWin;
end;

procedure TForm1.DrawBrick(ACanvas: TCanvas; Col, Row: Integer);
var
  BR: TRect;
  BrickColor, DarkColor, LightColor: TColor;
  Alpha: Double;
begin
  if not FBricks[Row][Col].Active then Exit;
  
  BR := BrickRect(Col, Row);
  
  // สีตามจำนวน Hits ที่เหลือ
  Alpha := FBricks[Row][Col].Hits / FBricks[Row][Col].MaxHits;
  var BaseR := GetRValue(FBricks[Row][Col].Color);
  var BaseG := GetGValue(FBricks[Row][Col].Color);
  var BaseB := GetBValue(FBricks[Row][Col].Color);
  BrickColor := RGB(Round(BaseR * Alpha + 80 * (1 - Alpha)),
                    Round(BaseG * Alpha + 80 * (1 - Alpha)),
                    Round(BaseB * Alpha + 80 * (1 - Alpha)));
  
  // วาด Brick
  ACanvas.Brush.Color := BrickColor;
  ACanvas.Pen.Style := psNone;
  ACanvas.FillRect(BR);
  
  // Highlight บน
  ACanvas.Pen.Style := psSolid;
  ACanvas.Pen.Color := clWhite;
  ACanvas.Pen.Width := 1;
  ACanvas.MoveTo(BR.Left, BR.Bottom - 1);
  ACanvas.LineTo(BR.Left, BR.Top);
  ACanvas.LineTo(BR.Right - 1, BR.Top);
  
  // Shadow ล่าง
  ACanvas.Pen.Color := TColor(RGB(0, 0, 0));
  ACanvas.MoveTo(BR.Right - 1, BR.Top);
  ACanvas.LineTo(BR.Right - 1, BR.Bottom - 1);
  ACanvas.LineTo(BR.Left, BR.Bottom - 1);
end;

procedure TForm1.DrawPaddle(ACanvas: TCanvas);
var
  PR: TRect;
begin
  PR := Rect(Round(FPaddleX), ClientHeight - 50,
              Round(FPaddleX) + PADDLE_WIDTH, ClientHeight - 50 + PADDLE_HEIGHT);
  
  // Paddle body
  ACanvas.Brush.Color := TColor(RGB(80, 150, 250));
  ACanvas.Pen.Style := psNone;
  ACanvas.FillRect(PR);
  
  // Highlight
  ACanvas.Pen.Style := psSolid;
  ACanvas.Pen.Color := clWhite;
  ACanvas.MoveTo(PR.Left, PR.Bottom - 1);
  ACanvas.LineTo(PR.Left, PR.Top);
  ACanvas.LineTo(PR.Right - 1, PR.Top);
end;

procedure TForm1.DrawBall(ACanvas: TCanvas);
var
  BX, BY: Integer;
begin
  BX := Round(FBallX);
  BY := Round(FBallY);
  
  // Shadow
  ACanvas.Brush.Color := TColor(RGB(0, 0, 0));
  ACanvas.Pen.Style := psNone;
  ACanvas.Ellipse(BX - BALL_RADIUS + 3, BY - BALL_RADIUS + 3,
                   BX + BALL_RADIUS + 3, BY + BALL_RADIUS + 3);
  
  // Ball
  ACanvas.Brush.Color := clWhite;
  ACanvas.Pen.Style := psSolid;
  ACanvas.Pen.Color := TColor(RGB(200, 200, 200));
  ACanvas.Pen.Width := 1;
  ACanvas.Ellipse(BX - BALL_RADIUS, BY - BALL_RADIUS,
                   BX + BALL_RADIUS, BY + BALL_RADIUS);
  
  // Highlight
  ACanvas.Brush.Color := clWhite;
  ACanvas.Pen.Style := psNone;
  ACanvas.Ellipse(BX - BALL_RADIUS + 2, BY - BALL_RADIUS + 2,
                   BX - BALL_RADIUS + 6, BY - BALL_RADIUS + 6);
end;

procedure TForm1.DrawHUD(ACanvas: TCanvas);
var
  i: Integer;
begin
  ACanvas.Font.Color := clWhite;
  ACanvas.Font.Size := 11;
  ACanvas.Font.Style := [fsBold];
  ACanvas.Brush.Style := bsClear;
  
  ACanvas.TextOut(10, 10, Format('คะแนน: %d', [FScore]));
  ACanvas.TextOut(10, 30, 'ชีวิต: ');
  for i := 0 to FLives - 1 do
  begin
    ACanvas.Brush.Color := clRed;
    ACanvas.Pen.Style := psNone;
    ACanvas.Ellipse(70 + i * 20, 33, 85 + i * 20, 48);
  end;
  ACanvas.TextOut(ClientWidth - 120, 10, Format('อิฐ: %d', [FBrickCount]));
end;

procedure TForm1.DrawMenu(ACanvas: TCanvas);
begin
  ACanvas.Font.Size := 24;
  ACanvas.Font.Color := clYellow;
  ACanvas.Font.Style := [fsBold];
  ACanvas.Brush.Style := bsClear;
  var S := 'BREAKOUT';
  ACanvas.TextOut((ClientWidth - ACanvas.TextWidth(S)) div 2, 150, S);
  
  ACanvas.Font.Size := 14;
  ACanvas.Font.Color := clWhite;
  ACanvas.Font.Style := [];
  S := 'คลิกหรือกด Space เพื่อเริ่ม';
  ACanvas.TextOut((ClientWidth - ACanvas.TextWidth(S)) div 2, 220, S);
end;

procedure TForm1.DrawGameOver(ACanvas: TCanvas);
var
  S: string;
begin
  ACanvas.Font.Size := 24;
  ACanvas.Font.Style := [fsBold];
  
  if FGameState = gsGameOver then
  begin
    ACanvas.Font.Color := clRed;
    S := 'GAME OVER';
  end else
  begin
    ACanvas.Font.Color := clYellow;
    S := 'ยอดเยี่ยม! คุณชนะ!';
  end;
  
  ACanvas.Brush.Style := bsClear;
  ACanvas.TextOut((ClientWidth - ACanvas.TextWidth(S)) div 2, 180, S);
  
  ACanvas.Font.Size := 14;
  ACanvas.Font.Color := clWhite;
  ACanvas.Font.Style := [];
  S := Format('คะแนนสุดท้าย: %d', [FScore]);
  ACanvas.TextOut((ClientWidth - ACanvas.TextWidth(S)) div 2, 230, S);
  
  S := 'คลิกเพื่อเล่นใหม่';
  ACanvas.TextOut((ClientWidth - ACanvas.TextWidth(S)) div 2, 270, S);
end;

procedure TForm1.RenderFrame;
var
  BC: TCanvas;
  Row, Col: Integer;
begin
  BC := FBackBuffer.Canvas;
  
  // พื้นหลัง gradient
  BC.Brush.Color := TColor(RGB(10, 10, 30));
  BC.FillRect(Rect(0, 0, ClientWidth, ClientHeight));
  
  case FGameState of
    gsMenu:
    begin
      DrawMenu(BC);
    end;
    
    gsPlaying, gsPaused:
    begin
      // วาดอิฐ
      for Row := 0 to BRICK_ROWS - 1 do
        for Col := 0 to BRICK_COLS - 1 do
          DrawBrick(BC, Col, Row);
      
      DrawPaddle(BC);
      DrawBall(BC);
      DrawHUD(BC);
      
      if FGameState = gsPaused then
      begin
        BC.Font.Size := 20;
        BC.Font.Color := clYellow;
        BC.Brush.Style := bsClear;
        var S := 'หยุดชั่วคราว (P เพื่อเล่นต่อ)';
        BC.TextOut((ClientWidth - BC.TextWidth(S)) div 2, 
                    ClientHeight div 2, S);
      end;
    end;
    
    gsGameOver, gsWin:
    begin
      DrawGameOver(BC);
    end;
  end;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  Update;
  RenderFrame;
  Invalidate;
end;

procedure TForm1.FormPaint(Sender: TObject);
begin
  Canvas.Draw(0, 0, FBackBuffer);
end;

procedure TForm1.FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  if Key < 256 then FKeys[Key] := True;
  
  if Key = VK_SPACE then
  begin
    if FGameState = gsMenu then InitGame
    else if FGameState = gsPlaying then FGameState := gsPaused
    else if FGameState = gsPaused then FGameState := gsPlaying;
  end;
  
  if Key = Ord('P') then
  begin
    if FGameState = gsPlaying then FGameState := gsPaused
    else if FGameState = gsPaused then FGameState := gsPlaying;
  end;
end;

procedure TForm1.FormKeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  if Key < 256 then FKeys[Key] := False;
end;

procedure TForm1.FormMouseMove(Sender: TObject; Shift: TShiftState; X, Y: Integer);
begin
  if FGameState = gsPlaying then
    FPaddleX := Max(0, Min(ClientWidth - PADDLE_WIDTH, X - PADDLE_WIDTH div 2));
end;

procedure TForm1.FormClick(Sender: TObject);
begin
  if FGameState in [gsGameOver, gsWin] then
    InitGame
  else if FGameState = gsMenu then
    InitGame;
end;

end.
```

---

## 36.14 Cairo Graphics บน Linux

```pascal
// หมายเหตุ: ต้องติดตั้ง libcairo และ unit cairo ก่อน
// ใน Lazarus ใช้ package CairoCanvas

unit CairoDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Cairo, CairoCanvas;

type
  TForm1 = class(TForm)
    procedure FormPaint(Sender: TObject);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormPaint(Sender: TObject);
var
  CC: TCairoCanvas;
begin
  CC := TCairoCanvas.Create(Canvas);
  try
    // ตั้งค่า Anti-aliasing
    CC.SetAntialias(CAIRO_ANTIALIAS_BEST);
    
    // วาดสี่เหลี่ยมมุมมน
    CC.SetSourceRGBA(0.2, 0.4, 0.8, 1.0);
    CC.RoundedRectangle(20, 20, 200, 100, 15);
    CC.FillAndStroke;
    
    // วาดเส้นโค้ง Bezier
    CC.SetSourceRGBA(0.8, 0.2, 0.2, 1.0);
    CC.SetLineWidth(3);
    CC.MoveTo(50, 200);
    CC.CurveTo(100, 150, 200, 250, 300, 200);
    CC.Stroke;
    
    // Gradient
    var Grad := CC.CreateLinearGradient(50, 250, 300, 350);
    CC.GradientAddColorStop(Grad, 0, 1, 0, 0, 1);    // แดง
    CC.GradientAddColorStop(Grad, 0.5, 0, 1, 0, 1);  // เขียว
    CC.GradientAddColorStop(Grad, 1, 0, 0, 1, 1);    // น้ำเงิน
    CC.SetSource(Grad);
    CC.Rectangle(50, 250, 250, 100);
    CC.Fill;
    
  finally
    CC.Free;
  end;
end;

end.
```

---

## 36.15 Drawing Optimizations

```pascal
unit DrawingOptimization;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, ExtCtrls;

type
  TForm1 = class(TForm)
    Timer1: TTimer;
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormPaint(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
  private
    FBackBuffer: TBitmap;
    FNeedRedraw: Boolean;
    FDirtyRect: TRect;
    FLastTime: Cardinal;
    FFPSCounter: Integer;
    FFPS: Integer;
    procedure MarkDirty(const ARect: TRect);
    procedure OptimizedRender;
  end;

var
  Form1: TForm1;

implementation

uses
  DateUtils, Windows;

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  DoubleBuffered := True;
  FBackBuffer := TBitmap.Create;
  FBackBuffer.Width := ClientWidth;
  FBackBuffer.Height := ClientHeight;
  FNeedRedraw := True;
  FDirtyRect := Rect(0, 0, ClientWidth, ClientHeight);
  FLastTime := GetTickCount;
  FFPSCounter := 0;
  FFPS := 0;
  Timer1.Interval := 16;
  Timer1.Enabled := True;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FBackBuffer.Free;
end;

procedure TForm1.MarkDirty(const ARect: TRect);
begin
  if not FNeedRedraw then
  begin
    FDirtyRect := ARect;
    FNeedRedraw := True;
  end else
  begin
    // Expand dirty rect
    FDirtyRect.Left := Min(FDirtyRect.Left, ARect.Left);
    FDirtyRect.Top := Min(FDirtyRect.Top, ARect.Top);
    FDirtyRect.Right := Max(FDirtyRect.Right, ARect.Right);
    FDirtyRect.Bottom := Max(FDirtyRect.Bottom, ARect.Bottom);
  end;
end;

procedure TForm1.OptimizedRender;
var
  BC: TCanvas;
  Now: Cardinal;
begin
  if not FNeedRedraw then Exit;
  
  BC := FBackBuffer.Canvas;
  
  // วาดเฉพาะส่วนที่ dirty
  BC.ClipRect; // Lazarus จัดการ ClipRect เอง
  
  // ตัวอย่าง: วาดพื้นหลังเฉพาะพื้นที่ที่เปลี่ยน
  BC.Brush.Color := clWhite;
  BC.FillRect(FDirtyRect);
  
  // วาด Content
  BC.Pen.Color := clBlue;
  BC.Pen.Width := 2;
  BC.Ellipse(100, 100, 300, 300);
  
  // FPS calculation
  Now := GetTickCount;
  Inc(FFPSCounter);
  if Now - FLastTime >= 1000 then
  begin
    FFPS := FFPSCounter;
    FFPSCounter := 0;
    FLastTime := Now;
  end;
  
  BC.Font.Color := clBlack;
  BC.Font.Size := 10;
  BC.Brush.Style := bsClear;
  BC.TextOut(5, 5, Format('FPS: %d', [FFPS]));
  BC.TextOut(5, 20, Format('Dirty: (%d,%d)-(%d,%d)',
    [FDirtyRect.Left, FDirtyRect.Top, FDirtyRect.Right, FDirtyRect.Bottom]));
  
  FNeedRedraw := False;
  FDirtyRect := Rect(0, 0, 0, 0);
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  // จำลองการเปลี่ยนแปลง
  MarkDirty(Rect(50, 50, 350, 350));
  OptimizedRender;
  Invalidate;
end;

procedure TForm1.FormPaint(Sender: TObject);
begin
  Canvas.Draw(0, 0, FBackBuffer);
end;

end.
```

---

## แบบฝึกหัดบทที่ 36

### ข้อ 1: วาดรูปดาว
สร้างโปรแกรมวาดดาว 5 แฉกที่สามารถกำหนดขนาดและสีได้

```pascal
// แนวทาง: ใช้ Polygon โดยคำนวณจุดของดาวจากสมการตรีโกณมิติ
procedure DrawStar(ACanvas: TCanvas; CX, CY, OuterR, InnerR: Integer; 
                   Points: Integer; AColor: TColor);
var
  i: Integer;
  Angle, InnerAngle: Double;
  Pts: array of TPoint;
begin
  SetLength(Pts, Points * 2);
  for i := 0 to Points - 1 do
  begin
    Angle := i * 2 * Pi / Points - Pi / 2;
    InnerAngle := Angle + Pi / Points;
    Pts[i * 2].X := CX + Round(OuterR * Cos(Angle));
    Pts[i * 2].Y := CY + Round(OuterR * Sin(Angle));
    Pts[i * 2 + 1].X := CX + Round(InnerR * Cos(InnerAngle));
    Pts[i * 2 + 1].Y := CY + Round(InnerR * Sin(InnerAngle));
  end;
  ACanvas.Brush.Color := AColor;
  ACanvas.Polygon(Pts);
end;
```

### ข้อ 2: วาดกราฟ Bar Chart
สร้างโปรแกรมแสดง Bar Chart จากข้อมูลที่กำหนด

### ข้อ 3: นาฬิกาดิจิทัล
สร้างนาฬิกาดิจิทัลที่แสดงเวลาปัจจุบัน พร้อม animation ที่สวยงาม

### ข้อ 4: Paint Application
สร้าง Paint โปรแกรมขั้นพื้นฐานที่สามารถวาดเส้น วงกลม สี่เหลี่ยมได้

```pascal
// แนวทาง: ใช้ Mouse Events ในการวาด
procedure TForm1.FormMouseDown(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  FDrawing := True;
  FStartX := X;
  FStartY := Y;
end;

procedure TForm1.FormMouseUp(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  if FDrawing then
  begin
    FDrawing := False;
    // วาดรูปทรงที่เลือก
    case FCurrentTool of
      toolLine: Canvas.MoveTo(FStartX, FStartY);
                Canvas.LineTo(X, Y);
      toolRect: Canvas.Rectangle(FStartX, FStartY, X, Y);
      toolEllipse: Canvas.Ellipse(FStartX, FStartY, X, Y);
    end;
  end;
end;
```

### ข้อ 5: Mandelbrot Set
สร้างโปรแกรมแสดง Mandelbrot Set fractal

```pascal
// แนวทาง:
procedure DrawMandelbrot(ACanvas: TCanvas; Width, Height: Integer);
var
  x, y, iter, MaxIter: Integer;
  cr, ci, zr, zi, tmp: Double;
  Scale: Double;
begin
  MaxIter := 100;
  Scale := 3.5;
  for y := 0 to Height - 1 do
    for x := 0 to Width - 1 do
    begin
      cr := (x / Width) * Scale - Scale / 2;
      ci := (y / Height) * Scale - Scale / 2;
      zr := 0; zi := 0;
      iter := 0;
      while (zr*zr + zi*zi < 4) and (iter < MaxIter) do
      begin
        tmp := zr*zr - zi*zi + cr;
        zi := 2*zr*zi + ci;
        zr := tmp;
        Inc(iter);
      end;
      if iter = MaxIter then
        ACanvas.Pixels[x, y] := clBlack
      else
      begin
        var v := Round((iter / MaxIter) * 255);
        ACanvas.Pixels[x, y] := RGB(v, v div 2, 255 - v);
      end;
    end;
end;
```

### ข้อ 6: Rubber Band Selection
สร้างกรอบเลือกพื้นที่แบบ Rubber Band (ขณะลากเมาส์)

### ข้อ 7: Gradient Background
สร้าง Gradient Background แนวตั้ง แนวนอน และแนวรัศมี

### ข้อ 8: Text Effects
สร้าง Text Effects: Shadow, Outline, Gradient text

### ข้อ 9: Screen Capture
สร้างโปรแกรม Capture หน้าจอบางส่วน บันทึกเป็น PNG

### ข้อ 10: Rotating Cube (2D Projection)
วาด Cube ในระบบ 3D แบบ isometric projection

### ข้อ 11: Pie Chart
สร้าง Pie Chart ที่มี animation เมื่อโหลดข้อมูล

### ข้อ 12: Kaleidoscope
สร้าง Kaleidoscope ที่เปลี่ยนลวดลายตามเวลา

### ข้อ 13: Waveform Visualizer
สร้าง Waveform visualizer สำหรับข้อมูล audio

### ข้อ 14: Maze Generator
สร้าง Random Maze และวาดแสดงผล

### ข้อ 15: Physics Simulation
สร้าง Simple Physics simulation: ลูกบอลที่ตกลงพื้นและกระดอน

```pascal
// แนวทาง Physics:
const
  GRAVITY = 0.5;
  BOUNCE = 0.8;
  
procedure UpdatePhysics(var Ball: TBall; FloorY: Integer);
begin
  // Gravity
  Ball.VY := Ball.VY + GRAVITY;
  
  // Update position
  Ball.X := Ball.X + Ball.VX;
  Ball.Y := Ball.Y + Ball.VY;
  
  // Bounce off floor
  if Ball.Y + Ball.Radius >= FloorY then
  begin
    Ball.Y := FloorY - Ball.Radius;
    Ball.VY := -Ball.VY * BOUNCE;
    Ball.VX := Ball.VX * 0.99;  // Friction
  end;
end;
```

---

## สรุปบทที่ 36

ในบทนี้เราได้เรียนรู้:
1. **พื้นฐาน Canvas** - TCanvas, Pen, Brush, Coordinate System
2. **Clipping Regions** - การกำหนดพื้นที่วาด
3. **Transformations** - การหมุน ขยาย เลื่อน
4. **Anti-aliasing** - การทำให้ขอบเส้นเรียบ
5. **Bezier Curves** - การวาดเส้นโค้ง
6. **Patterns และ Textures** - การเติม Pattern
7. **Double Buffering** - ป้องกัน Flickering
8. **Animation** - การสร้าง Animation
9. **Sprite Animation** - Animation แบบ Sprite
10. **Complete Examples** - นาฬิกาอนาล็อก และเกม Breakout

การใช้ Canvas อย่างมีประสิทธิภาพจะช่วยให้โปรแกรมมี UI ที่สวยงามและ Animation ที่ลื่นไหล
