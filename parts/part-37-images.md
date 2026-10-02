# Part 37 - Image Processing

## บทนำ

Image Processing คือการประมวลผลภาพดิจิทัลเพื่อแก้ไข ปรับปรุง หรือวิเคราะห์ภาพ Lazarus มี component และ library หลายตัวสำหรับการทำงานกับรูปภาพ ในบทนี้เราจะเรียนรู้การโหลด บันทึก และประมวลผลภาพในรูปแบบต่างๆ

---

## 37.1 การโหลดและบันทึกรูปภาพ

### รูปแบบภาพที่รองรับ

Lazarus รองรับรูปแบบภาพหลักๆ ผ่าน FCL-Image library:
- **BMP** - Windows Bitmap (built-in)
- **PNG** - Portable Network Graphics
- **JPEG** - Joint Photographic Experts Group
- **GIF** - Graphics Interchange Format
- **TIFF** - Tagged Image File Format
- **ICO** - Windows Icon

### การโหลดภาพ

```pascal
unit ImageLoadSave;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, 
  Dialogs, ExtCtrls, Buttons, ComCtrls,
  // เพิ่ม Format handlers
  FPImage, FPReadPNG, FPWritePNG,
  FPReadJPEG, FPWriteJPEG,
  FPReadBMP, FPWriteBMP,
  FPReadGIF,
  FPReadTIFF, FPWriteTIFF,
  IntfGraphics, GraphType;

type
  TForm1 = class(TForm)
    Image1: TImage;
    OpenDialog1: TOpenDialog;
    SaveDialog1: TSaveDialog;
    btnLoad: TButton;
    btnSave: TButton;
    StatusBar1: TStatusBar;
    procedure btnLoadClick(Sender: TObject);
    procedure btnSaveClick(Sender: TObject);
  private
    FCurrentImage: TBitmap;
    procedure UpdateStatus(const Msg: string);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.UpdateStatus(const Msg: string);
begin
  StatusBar1.SimpleText := Msg;
end;

procedure TForm1.btnLoadClick(Sender: TObject);
var
  Ext: string;
  Pic: TPicture;
begin
  OpenDialog1.Filter := 
    'All Images|*.png;*.jpg;*.jpeg;*.bmp;*.gif;*.tif;*.tiff|' +
    'PNG (*.png)|*.png|' +
    'JPEG (*.jpg;*.jpeg)|*.jpg;*.jpeg|' +
    'BMP (*.bmp)|*.bmp|' +
    'GIF (*.gif)|*.gif|' +
    'TIFF (*.tif;*.tiff)|*.tif;*.tiff';
  
  if not OpenDialog1.Execute then Exit;
  
  Pic := TPicture.Create;
  try
    Pic.LoadFromFile(OpenDialog1.FileName);
    
    // แปลงเป็น Bitmap เพื่อการประมวลผล
    if not Assigned(FCurrentImage) then
      FCurrentImage := TBitmap.Create;
    
    FCurrentImage.Width := Pic.Width;
    FCurrentImage.Height := Pic.Height;
    FCurrentImage.Canvas.Draw(0, 0, Pic.Graphic);
    
    Image1.Picture.Assign(FCurrentImage);
    Image1.Stretch := True;
    
    UpdateStatus(Format('โหลด: %s (%dx%d pixels)', [
      ExtractFileName(OpenDialog1.FileName),
      Pic.Width, Pic.Height
    ]));
  finally
    Pic.Free;
  end;
end;

procedure TForm1.btnSaveClick(Sender: TObject);
var
  Ext: string;
begin
  if not Assigned(FCurrentImage) then
  begin
    ShowMessage('ไม่มีรูปภาพ');
    Exit;
  end;
  
  SaveDialog1.Filter :=
    'PNG (*.png)|*.png|' +
    'JPEG (*.jpg)|*.jpg|' +
    'BMP (*.bmp)|*.bmp|' +
    'TIFF (*.tif)|*.tif';
  
  if not SaveDialog1.Execute then Exit;
  
  Ext := LowerCase(ExtractFileExt(SaveDialog1.FileName));
  FCurrentImage.SaveToFile(SaveDialog1.FileName);
  
  UpdateStatus('บันทึก: ' + ExtractFileName(SaveDialog1.FileName));
end;

end.
```

### การใช้ FCL-Image สำหรับรูปแบบขั้นสูง

```pascal
uses
  FPImage, FPReadPNG, FPWritePNG, FPReadJPEG, FPWriteJPEG,
  FPCanvas, IntfGraphics;

// โหลด PNG
function LoadPNG(const FileName: string): TFPCustomImage;
var
  Reader: TFPReaderPNG;
begin
  Result := TFPMemoryImage.Create(0, 0);
  Reader := TFPReaderPNG.Create;
  try
    Reader.LoadFromFile(FileName, Result);
  finally
    Reader.Free;
  end;
end;

// บันทึก JPEG พร้อมตั้งค่า Quality
procedure SaveJPEG(AImage: TFPCustomImage; const FileName: string; Quality: Integer);
var
  Writer: TFPWriterJPEG;
begin
  Writer := TFPWriterJPEG.Create;
  try
    Writer.CompressionQuality := Quality;  // 1-100
    AImage.SaveToFile(FileName, Writer);
  finally
    Writer.Free;
  end;
end;

// แปลง TBitmap เป็น TFPMemoryImage
function BitmapToFPImage(ABitmap: TBitmap): TFPMemoryImage;
var
  LazImage: TLazIntfImage;
  x, y: Integer;
  Pixel: TFPColor;
  R, G, B: Byte;
begin
  LazImage := ABitmap.CreateIntfImage;
  Result := TFPMemoryImage.Create(ABitmap.Width, ABitmap.Height);
  try
    for y := 0 to ABitmap.Height - 1 do
      for x := 0 to ABitmap.Width - 1 do
      begin
        Pixel := LazImage.Colors[x, y];
        // FPColor ใช้ 16-bit ต่อ channel, แปลงเป็น 8-bit
        R := Pixel.Red shr 8;
        G := Pixel.Green shr 8;
        B := Pixel.Blue shr 8;
        Result.Colors[x, y] := FPColor(R shl 8 or R, G shl 8 or G, B shl 8 or B, $FFFF);
      end;
  finally
    LazImage.Free;
  end;
end;
```

---

## 37.2 การปรับขนาดภาพ (Image Resizing)

```pascal
unit ImageResize;

interface

uses
  Classes, SysUtils, Graphics, Math;

type
  TResizeMethod = (rmNearest, rmBilinear, rmBicubic);

// Nearest Neighbor - เร็วที่สุด แต่คุณภาพต่ำ
function ResizeNearest(ASource: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
// Bilinear Interpolation - คุณภาพดีกว่า
function ResizeBilinear(ASource: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
// Bicubic - คุณภาพดีที่สุด แต่ช้าที่สุด
function ResizeBicubic(ASource: TBitmap; NewWidth, NewHeight: Integer): TBitmap;

implementation

function ResizeNearest(ASource: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
var
  x, y: Integer;
  SrcX, SrcY: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := NewWidth;
  Result.Height := NewHeight;
  
  for y := 0 to NewHeight - 1 do
  begin
    SrcY := (y * ASource.Height) div NewHeight;
    for x := 0 to NewWidth - 1 do
    begin
      SrcX := (x * ASource.Width) div NewWidth;
      Result.Canvas.Pixels[x, y] := ASource.Canvas.Pixels[SrcX, SrcY];
    end;
  end;
end;

function ResizeBilinear(ASource: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
var
  x, y: Integer;
  SrcX, SrcY: Double;
  X0, Y0, X1, Y1: Integer;
  Fx, Fy: Double;
  C00, C10, C01, C11: TColor;
  R, G, B: Double;
  
  function Lerp(A, B, T: Double): Double;
  begin
    Result := A * (1 - T) + B * T;
  end;
  
  function BiLerp(C00, C10, C01, C11: TColor; Fx, Fy: Double): TColor;
  var
    R, G, B: Double;
  begin
    R := Lerp(Lerp(GetRValue(C00), GetRValue(C10), Fx),
               Lerp(GetRValue(C01), GetRValue(C11), Fx), Fy);
    G := Lerp(Lerp(GetGValue(C00), GetGValue(C10), Fx),
               Lerp(GetGValue(C01), GetGValue(C11), Fx), Fy);
    B := Lerp(Lerp(GetBValue(C00), GetBValue(C10), Fx),
               Lerp(GetBValue(C01), GetBValue(C11), Fx), Fy);
    Result := RGB(Round(R), Round(G), Round(B));
  end;

begin
  Result := TBitmap.Create;
  Result.Width := NewWidth;
  Result.Height := NewHeight;
  
  for y := 0 to NewHeight - 1 do
  begin
    SrcY := y * (ASource.Height - 1) / NewHeight;
    Y0 := Trunc(SrcY);
    Y1 := Min(Y0 + 1, ASource.Height - 1);
    Fy := Frac(SrcY);
    
    for x := 0 to NewWidth - 1 do
    begin
      SrcX := x * (ASource.Width - 1) / NewWidth;
      X0 := Trunc(SrcX);
      X1 := Min(X0 + 1, ASource.Width - 1);
      Fx := Frac(SrcX);
      
      C00 := ASource.Canvas.Pixels[X0, Y0];
      C10 := ASource.Canvas.Pixels[X1, Y0];
      C01 := ASource.Canvas.Pixels[X0, Y1];
      C11 := ASource.Canvas.Pixels[X1, Y1];
      
      Result.Canvas.Pixels[x, y] := BiLerp(C00, C10, C01, C11, Fx, Fy);
    end;
  end;
end;

function CubicWeight(t: Double): Double;
const
  A = -0.5;
var
  AbsT: Double;
begin
  AbsT := Abs(t);
  if AbsT <= 1 then
    Result := (A + 2) * AbsT * AbsT * AbsT - (A + 3) * AbsT * AbsT + 1
  else if AbsT < 2 then
    Result := A * AbsT * AbsT * AbsT - 5 * A * AbsT * AbsT + 8 * A * AbsT - 4 * A
  else
    Result := 0;
end;

function ResizeBicubic(ASource: TBitmap; NewWidth, NewHeight: Integer): TBitmap;
var
  x, y: Integer;
  SrcX, SrcY: Double;
  X0, Y0: Integer;
  i, j: Integer;
  Weight, TotalWeight: Double;
  SX, SY: Integer;
  R, G, B: Double;
  C: TColor;
begin
  Result := TBitmap.Create;
  Result.Width := NewWidth;
  Result.Height := NewHeight;
  
  for y := 0 to NewHeight - 1 do
  begin
    SrcY := y * ASource.Height / NewHeight;
    Y0 := Trunc(SrcY);
    
    for x := 0 to NewWidth - 1 do
    begin
      SrcX := x * ASource.Width / NewWidth;
      X0 := Trunc(SrcX);
      
      R := 0; G := 0; B := 0;
      TotalWeight := 0;
      
      for j := -1 to 2 do
      begin
        for i := -1 to 2 do
        begin
          SX := Max(0, Min(ASource.Width - 1, X0 + i));
          SY := Max(0, Min(ASource.Height - 1, Y0 + j));
          
          Weight := CubicWeight(SrcX - (X0 + i)) * CubicWeight(SrcY - (Y0 + j));
          C := ASource.Canvas.Pixels[SX, SY];
          
          R := R + GetRValue(C) * Weight;
          G := G + GetGValue(C) * Weight;
          B := B + GetBValue(C) * Weight;
          TotalWeight := TotalWeight + Weight;
        end;
      end;
      
      if TotalWeight > 0 then
        Result.Canvas.Pixels[x, y] := RGB(
          Max(0, Min(255, Round(R / TotalWeight))),
          Max(0, Min(255, Round(G / TotalWeight))),
          Max(0, Min(255, Round(B / TotalWeight)))
        );
    end;
  end;
end;

end.
```

---

## 37.3 การตัดภาพ (Image Cropping)

```pascal
// ตัดภาพตามพื้นที่ที่กำหนด
function CropImage(ASource: TBitmap; CropRect: TRect): TBitmap;
var
  NewWidth, NewHeight: Integer;
begin
  // ตรวจสอบ bounds
  CropRect.Left := Max(0, Min(ASource.Width - 1, CropRect.Left));
  CropRect.Top := Max(0, Min(ASource.Height - 1, CropRect.Top));
  CropRect.Right := Max(CropRect.Left + 1, Min(ASource.Width, CropRect.Right));
  CropRect.Bottom := Max(CropRect.Top + 1, Min(ASource.Height, CropRect.Bottom));
  
  NewWidth := CropRect.Right - CropRect.Left;
  NewHeight := CropRect.Bottom - CropRect.Top;
  
  Result := TBitmap.Create;
  Result.Width := NewWidth;
  Result.Height := NewHeight;
  
  // Copy pixels
  Result.Canvas.CopyRect(Rect(0, 0, NewWidth, NewHeight),
                          ASource.Canvas, CropRect);
end;

// การ Crop แบบ Center Crop (เหมาะกับ Thumbnail)
function CenterCrop(ASource: TBitmap; TargetWidth, TargetHeight: Integer): TBitmap;
var
  SrcAspect, DstAspect: Double;
  SrcRect: TRect;
  CW, CH: Integer;
begin
  SrcAspect := ASource.Width / ASource.Height;
  DstAspect := TargetWidth / TargetHeight;
  
  if SrcAspect > DstAspect then
  begin
    // Source กว้างกว่า - Crop ด้านข้าง
    CH := ASource.Height;
    CW := Round(CH * DstAspect);
  end else
  begin
    // Source สูงกว่า - Crop ด้านบนล่าง
    CW := ASource.Width;
    CH := Round(CW / DstAspect);
  end;
  
  SrcRect.Left := (ASource.Width - CW) div 2;
  SrcRect.Top := (ASource.Height - CH) div 2;
  SrcRect.Right := SrcRect.Left + CW;
  SrcRect.Bottom := SrcRect.Top + CH;
  
  var Cropped := CropImage(ASource, SrcRect);
  try
    Result := ResizeBilinear(Cropped, TargetWidth, TargetHeight);
  finally
    Cropped.Free;
  end;
end;
```

---

## 37.4 การปรับ Brightness, Contrast, Saturation

```pascal
unit ImageAdjustments;

interface

uses
  Classes, SysUtils, Graphics, Math;

// ปรับความสว่าง: Value -255 ถึง +255
function AdjustBrightness(ASource: TBitmap; Value: Integer): TBitmap;

// ปรับ Contrast: Value -100 ถึง +100
function AdjustContrast(ASource: TBitmap; Value: Integer): TBitmap;

// ปรับ Saturation: Value -100 ถึง +100
function AdjustSaturation(ASource: TBitmap; Value: Integer): TBitmap;

// ปรับ Hue: Value -180 ถึง +180 degrees
function AdjustHue(ASource: TBitmap; Value: Integer): TBitmap;

// ปรับ Gamma: Value 0.1 ถึง 5.0 (1.0 = ไม่เปลี่ยน)
function AdjustGamma(ASource: TBitmap; Value: Double): TBitmap;

// ปรับหลายค่าพร้อมกัน
function AdjustImage(ASource: TBitmap; 
                      Brightness, Contrast, Saturation: Integer): TBitmap;

implementation

function ClampByte(Value: Integer): Byte;
begin
  if Value < 0 then Result := 0
  else if Value > 255 then Result := 255
  else Result := Value;
end;

function AdjustBrightness(ASource: TBitmap; Value: Integer): TBitmap;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      R := ClampByte(GetRValue(C) + Value);
      G := ClampByte(GetGValue(C) + Value);
      B := ClampByte(GetBValue(C) + Value);
      Result.Canvas.Pixels[x, y] := RGB(R, G, B);
    end;
end;

function AdjustContrast(ASource: TBitmap; Value: Integer): TBitmap;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Integer;
  Factor: Double;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  // คำนวณ Factor
  Factor := (259 * (Value + 255)) / (255 * (259 - Value));
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      R := ClampByte(Round(Factor * (GetRValue(C) - 128) + 128));
      G := ClampByte(Round(Factor * (GetGValue(C) - 128) + 128));
      B := ClampByte(Round(Factor * (GetBValue(C) - 128) + 128));
      Result.Canvas.Pixels[x, y] := RGB(R, G, B);
    end;
end;

// แปลง RGB เป็น HSL
procedure RGBtoHSL(R, G, B: Byte; out H, S, L: Double);
var
  MaxC, MinC, Delta: Double;
  RF, GF, BF: Double;
begin
  RF := R / 255;
  GF := G / 255;
  BF := B / 255;
  
  MaxC := Max(Max(RF, GF), BF);
  MinC := Min(Min(RF, GF), BF);
  Delta := MaxC - MinC;
  
  L := (MaxC + MinC) / 2;
  
  if Delta = 0 then
  begin
    H := 0;
    S := 0;
  end else
  begin
    if L < 0.5 then S := Delta / (MaxC + MinC)
    else S := Delta / (2 - MaxC - MinC);
    
    if MaxC = RF then
      H := ((GF - BF) / Delta) mod 6
    else if MaxC = GF then
      H := (BF - RF) / Delta + 2
    else
      H := (RF - GF) / Delta + 4;
    
    H := H * 60;
    if H < 0 then H := H + 360;
  end;
end;

// แปลง HSL เป็น RGB
procedure HSLtoRGB(H, S, L: Double; out R, G, B: Byte);

  function HueToRGB(P, Q, T: Double): Double;
  begin
    if T < 0 then T := T + 1;
    if T > 1 then T := T - 1;
    if T < 1/6 then Result := P + (Q - P) * 6 * T
    else if T < 1/2 then Result := Q
    else if T < 2/3 then Result := P + (Q - P) * (2/3 - T) * 6
    else Result := P;
  end;

var
  Q, P: Double;
begin
  if S = 0 then
  begin
    R := Round(L * 255);
    G := R;
    B := R;
  end else
  begin
    if L < 0.5 then Q := L * (1 + S)
    else Q := L + S - L * S;
    P := 2 * L - Q;
    H := H / 360;
    R := Round(HueToRGB(P, Q, H + 1/3) * 255);
    G := Round(HueToRGB(P, Q, H) * 255);
    B := Round(HueToRGB(P, Q, H - 1/3) * 255);
  end;
end;

function AdjustSaturation(ASource: TBitmap; Value: Integer): TBitmap;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Byte;
  H, S, L: Double;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      RGBtoHSL(GetRValue(C), GetGValue(C), GetBValue(C), H, S, L);
      
      S := S + Value / 100;
      if S < 0 then S := 0;
      if S > 1 then S := 1;
      
      HSLtoRGB(H, S, L, R, G, B);
      Result.Canvas.Pixels[x, y] := RGB(R, G, B);
    end;
end;

function AdjustHue(ASource: TBitmap; Value: Integer): TBitmap;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Byte;
  H, S, L: Double;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      RGBtoHSL(GetRValue(C), GetGValue(C), GetBValue(C), H, S, L);
      
      H := H + Value;
      while H < 0 do H := H + 360;
      while H >= 360 do H := H - 360;
      
      HSLtoRGB(H, S, L, R, G, B);
      Result.Canvas.Pixels[x, y] := RGB(R, G, B);
    end;
end;

function AdjustGamma(ASource: TBitmap; Value: Double): TBitmap;
var
  x, y: Integer;
  C: TColor;
  GammaTable: array[0..255] of Byte;
  i: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  // สร้าง Gamma Lookup Table
  for i := 0 to 255 do
    GammaTable[i] := Round(255 * Power(i / 255, 1 / Value));
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      Result.Canvas.Pixels[x, y] := RGB(
        GammaTable[GetRValue(C)],
        GammaTable[GetGValue(C)],
        GammaTable[GetBValue(C)]
      );
    end;
end;

function AdjustImage(ASource: TBitmap;
  Brightness, Contrast, Saturation: Integer): TBitmap;
var
  Temp1, Temp2: TBitmap;
begin
  Temp1 := AdjustBrightness(ASource, Brightness);
  try
    Temp2 := AdjustContrast(Temp1, Contrast);
    try
      Result := AdjustSaturation(Temp2, Saturation);
    finally
      Temp2.Free;
    end;
  finally
    Temp1.Free;
  end;
end;

end.
```

---

## 37.5 แปลงเป็น Grayscale

```pascal
// หลายวิธีในการแปลงเป็นภาพขาวดำ

// วิธีที่ 1: Average (ง่ายที่สุด)
function ToGrayscaleAverage(ASource: TBitmap): TBitmap;
var
  x, y: Integer;
  C: TColor;
  Gray: Byte;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      Gray := (GetRValue(C) + GetGValue(C) + GetBValue(C)) div 3;
      Result.Canvas.Pixels[x, y] := RGB(Gray, Gray, Gray);
    end;
end;

// วิธีที่ 2: Luminance (ตรงกับการรับรู้ของมนุษย์)
function ToGrayscaleLuminance(ASource: TBitmap): TBitmap;
var
  x, y: Integer;
  C: TColor;
  Gray: Byte;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      // ITU-R BT.709 standard
      Gray := ClampByte(Round(
        0.2126 * GetRValue(C) +
        0.7152 * GetGValue(C) +
        0.0722 * GetBValue(C)
      ));
      Result.Canvas.Pixels[x, y] := RGB(Gray, Gray, Gray);
    end;
end;

// วิธีที่ 3: Desaturation (ค่า Max+Min / 2)
function ToGrayscaleDesaturate(ASource: TBitmap): TBitmap;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Byte;
  Gray: Byte;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      R := GetRValue(C);
      G := GetGValue(C);
      B := GetBValue(C);
      Gray := (Max(Max(R, G), B) + Min(Min(R, G), B)) div 2;
      Result.Canvas.Pixels[x, y] := RGB(Gray, Gray, Gray);
    end;
end;

// Sepia effect
function ToSepia(ASource: TBitmap): TBitmap;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Byte;
  Gray: Double;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      Gray := 0.2126 * GetRValue(C) + 0.7152 * GetGValue(C) + 0.0722 * GetBValue(C);
      
      R := ClampByte(Round(Gray * 1.07));
      G := ClampByte(Round(Gray * 0.74));
      B := ClampByte(Round(Gray * 0.43));
      
      Result.Canvas.Pixels[x, y] := RGB(R, G, B);
    end;
end;
```

---

## 37.6 Histogram

```pascal
unit ImageHistogram;

interface

uses
  Classes, SysUtils, Graphics, Math;

type
  THistogram = array[0..255] of Cardinal;
  
  TImageHistogram = class
  private
    FRed, FGreen, FBlue, FLuminance: THistogram;
    FTotalPixels: Cardinal;
    FBitmap: TBitmap;
    procedure Calculate;
  public
    constructor Create(ABitmap: TBitmap);
    procedure DrawHistogram(ACanvas: TCanvas; ARect: TRect; Channel: Integer);
    // Channel: 0=R, 1=G, 2=B, 3=Luminance, 4=All
    property Red: THistogram read FRed;
    property Green: THistogram read FGreen;
    property Blue: THistogram read FBlue;
    property Luminance: THistogram read FLuminance;
    property TotalPixels: Cardinal read FTotalPixels;
  end;

implementation

constructor TImageHistogram.Create(ABitmap: TBitmap);
begin
  inherited Create;
  FBitmap := ABitmap;
  FTotalPixels := ABitmap.Width * ABitmap.Height;
  Calculate;
end;

procedure TImageHistogram.Calculate;
var
  x, y: Integer;
  C: TColor;
  R, G, B: Byte;
  Lum: Byte;
begin
  FillChar(FRed, SizeOf(FRed), 0);
  FillChar(FGreen, SizeOf(FGreen), 0);
  FillChar(FBlue, SizeOf(FBlue), 0);
  FillChar(FLuminance, SizeOf(FLuminance), 0);
  
  for y := 0 to FBitmap.Height - 1 do
    for x := 0 to FBitmap.Width - 1 do
    begin
      C := FBitmap.Canvas.Pixels[x, y];
      R := GetRValue(C);
      G := GetGValue(C);
      B := GetBValue(C);
      Lum := Round(0.2126 * R + 0.7152 * G + 0.0722 * B);
      
      Inc(FRed[R]);
      Inc(FGreen[G]);
      Inc(FBlue[B]);
      Inc(FLuminance[Lum]);
    end;
end;

procedure TImageHistogram.DrawHistogram(ACanvas: TCanvas; ARect: TRect; Channel: Integer);
var
  i: Integer;
  MaxVal: Cardinal;
  HistData: THistogram;
  BarH, BarX: Integer;
  BarColor: TColor;
  DrawWidth: Integer;
begin
  // เลือก Channel
  case Channel of
    0: begin HistData := FRed;   BarColor := clRed;   end;
    1: begin HistData := FGreen; BarColor := clGreen; end;
    2: begin HistData := FBlue;  BarColor := clBlue;  end;
    3: begin HistData := FLuminance; BarColor := clGray; end;
    else
    begin
      // วาดทุก Channel ซ้อนกัน
      DrawHistogram(ACanvas, ARect, 0);
      DrawHistogram(ACanvas, ARect, 1);
      DrawHistogram(ACanvas, ARect, 2);
      Exit;
    end;
  end;
  
  // หา Max value
  MaxVal := 1;
  for i := 0 to 255 do
    if HistData[i] > MaxVal then MaxVal := HistData[i];
  
  DrawWidth := ARect.Right - ARect.Left;
  
  // วาด Histogram
  ACanvas.Pen.Style := psNone;
  
  for i := 0 to 255 do
  begin
    BarH := Round((ARect.Bottom - ARect.Top) * HistData[i] / MaxVal);
    BarX := ARect.Left + Round(i * DrawWidth / 256);
    
    // วาดแท่ง
    ACanvas.Brush.Color := BarColor;
    ACanvas.FillRect(Rect(BarX, ARect.Bottom - BarH,
                           BarX + Max(1, DrawWidth div 256), ARect.Bottom));
  end;
  
  // วาดกรอบ
  ACanvas.Pen.Style := psSolid;
  ACanvas.Pen.Color := clBlack;
  ACanvas.Pen.Width := 1;
  ACanvas.Brush.Style := bsClear;
  ACanvas.Rectangle(ARect);
end;

end.
```

---

## 37.7 Edge Detection (Sobel Filter)

```pascal
unit EdgeDetection;

interface

uses
  Classes, SysUtils, Graphics, Math;

// Sobel Edge Detection
function SobelEdgeDetection(ASource: TBitmap; Threshold: Integer = 128): TBitmap;

// Canny Edge Detection (ง่ายขึ้น)
function CannyEdgeDetection(ASource: TBitmap; LowThreshold, HighThreshold: Integer): TBitmap;

// Roberts Cross
function RobertsCross(ASource: TBitmap; Threshold: Integer = 128): TBitmap;

// Laplacian
function LaplacianEdge(ASource: TBitmap; Threshold: Integer = 128): TBitmap;

implementation

// แปลงเป็น Grayscale ก่อน
function GetGray(C: TColor): Integer;
begin
  Result := Round(0.2126 * GetRValue(C) + 
                   0.7152 * GetGValue(C) + 
                   0.0722 * GetBValue(C));
end;

function SobelEdgeDetection(ASource: TBitmap; Threshold: Integer): TBitmap;
var
  x, y: Integer;
  Gx, Gy, G: Integer;
  Sobel_x: array[-1..1, -1..1] of Integer = ((-1, -2, -1), (0, 0, 0), (1, 2, 1));
  Sobel_y: array[-1..1, -1..1] of Integer = ((-1, 0, 1), (-2, 0, 2), (-1, 0, 1));
  i, j: Integer;
  Gray: Integer;
  GrayBitmap: TBitmap;
begin
  // สร้าง Grayscale bitmap ก่อน
  GrayBitmap := TBitmap.Create;
  try
    GrayBitmap.Width := ASource.Width;
    GrayBitmap.Height := ASource.Height;
    
    for y := 0 to ASource.Height - 1 do
      for x := 0 to ASource.Width - 1 do
      begin
        Gray := GetGray(ASource.Canvas.Pixels[x, y]);
        GrayBitmap.Canvas.Pixels[x, y] := RGB(Gray, Gray, Gray);
      end;
    
    Result := TBitmap.Create;
    Result.Width := ASource.Width;
    Result.Height := ASource.Height;
    
    // ใช้ Sobel filter (ข้าม border)
    for y := 1 to ASource.Height - 2 do
      for x := 1 to ASource.Width - 2 do
      begin
        Gx := 0;
        Gy := 0;
        
        for j := -1 to 1 do
          for i := -1 to 1 do
          begin
            Gray := GetGray(GrayBitmap.Canvas.Pixels[x + i, y + j]);
            Gx := Gx + Sobel_x[j, i] * Gray;
            Gy := Gy + Sobel_y[j, i] * Gray;
          end;
        
        G := Min(255, Round(Sqrt(Gx * Gx + Gy * Gy)));
        
        if G >= Threshold then
          Result.Canvas.Pixels[x, y] := clWhite
        else
          Result.Canvas.Pixels[x, y] := clBlack;
      end;
  finally
    GrayBitmap.Free;
  end;
end;

function RobertsCross(ASource: TBitmap; Threshold: Integer): TBitmap;
var
  x, y: Integer;
  G1, G2, G: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 0 to ASource.Height - 2 do
    for x := 0 to ASource.Width - 2 do
    begin
      G1 := GetGray(ASource.Canvas.Pixels[x, y]) -
             GetGray(ASource.Canvas.Pixels[x + 1, y + 1]);
      G2 := GetGray(ASource.Canvas.Pixels[x + 1, y]) -
             GetGray(ASource.Canvas.Pixels[x, y + 1]);
      
      G := Min(255, Round(Sqrt(G1 * G1 + G2 * G2)));
      
      if G >= Threshold then
        Result.Canvas.Pixels[x, y] := clWhite
      else
        Result.Canvas.Pixels[x, y] := clBlack;
    end;
end;

function LaplacianEdge(ASource: TBitmap; Threshold: Integer): TBitmap;
var
  x, y: Integer;
  G: Integer;
  Kernel: array[-1..1, -1..1] of Integer = ((0, -1, 0), (-1, 4, -1), (0, -1, 0));
  i, j: Integer;
  Gray: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 1 to ASource.Height - 2 do
    for x := 1 to ASource.Width - 2 do
    begin
      G := 0;
      for j := -1 to 1 do
        for i := -1 to 1 do
        begin
          Gray := GetGray(ASource.Canvas.Pixels[x + i, y + j]);
          G := G + Kernel[j, i] * Gray;
        end;
      
      G := Abs(G);
      if G > 255 then G := 255;
      
      if G >= Threshold then
        Result.Canvas.Pixels[x, y] := clWhite
      else
        Result.Canvas.Pixels[x, y] := clBlack;
    end;
end;

// Canny ขั้นพื้นฐาน
function CannyEdgeDetection(ASource: TBitmap; LowThreshold, HighThreshold: Integer): TBitmap;
var
  BlurredBitmap: TBitmap;
  x, y, i, j: Integer;
  // Gaussian 5x5 kernel
  GaussKernel: array[0..4, 0..4] of Double = (
    (2,  4,  5,  4,  2),
    (4,  9, 12,  9,  4),
    (5, 12, 15, 12,  5),
    (4,  9, 12,  9,  4),
    (2,  4,  5,  4,  2)
  );
  GaussSum: Double;
  R, G, B: Double;
  C: TColor;
begin
  // 1. Gaussian Blur
  BlurredBitmap := TBitmap.Create;
  BlurredBitmap.Width := ASource.Width;
  BlurredBitmap.Height := ASource.Height;
  
  GaussSum := 0;
  for i := 0 to 4 do
    for j := 0 to 4 do
      GaussSum := GaussSum + GaussKernel[i, j];
  
  for y := 2 to ASource.Height - 3 do
    for x := 2 to ASource.Width - 3 do
    begin
      R := 0; G := 0; B := 0;
      for j := 0 to 4 do
        for i := 0 to 4 do
        begin
          C := ASource.Canvas.Pixels[x + i - 2, y + j - 2];
          R := R + GetRValue(C) * GaussKernel[j, i];
          G := G + GetGValue(C) * GaussKernel[j, i];
          B := B + GetBValue(C) * GaussKernel[j, i];
        end;
      BlurredBitmap.Canvas.Pixels[x, y] := RGB(
        Round(R / GaussSum),
        Round(G / GaussSum),
        Round(B / GaussSum)
      );
    end;
  
  // 2. ใช้ Sobel บน Blurred image
  Result := SobelEdgeDetection(BlurredBitmap, LowThreshold);
  BlurredBitmap.Free;
end;

end.
```

---

## 37.8 Blur และ Sharpen

```pascal
unit ImageFilters;

interface

uses
  Classes, SysUtils, Graphics, Math;

// Gaussian Blur
function GaussianBlur(ASource: TBitmap; Radius: Integer): TBitmap;

// Box Blur (เร็วกว่า Gaussian)
function BoxBlur(ASource: TBitmap; Radius: Integer): TBitmap;

// Unsharp Mask (Sharpen)
function UnsharpMask(ASource: TBitmap; Radius: Integer; Amount: Double; 
                      Threshold: Integer = 0): TBitmap;

// Motion Blur
function MotionBlur(ASource: TBitmap; Distance: Integer; 
                    AngleDeg: Double): TBitmap;

// Emboss
function EmbossFilter(ASource: TBitmap): TBitmap;

implementation

function BoxBlur(ASource: TBitmap; Radius: Integer): TBitmap;
var
  x, y, dx, dy: Integer;
  R, G, B: Integer;
  Count: Integer;
  C: TColor;
  KernelSize: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  KernelSize := 2 * Radius + 1;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      R := 0; G := 0; B := 0;
      Count := 0;
      
      for dy := -Radius to Radius do
        for dx := -Radius to Radius do
        begin
          var NX := Max(0, Min(ASource.Width - 1, x + dx));
          var NY := Max(0, Min(ASource.Height - 1, y + dy));
          C := ASource.Canvas.Pixels[NX, NY];
          R := R + GetRValue(C);
          G := G + GetGValue(C);
          B := B + GetBValue(C);
          Inc(Count);
        end;
      
      Result.Canvas.Pixels[x, y] := RGB(R div Count, G div Count, B div Count);
    end;
end;

function GaussianBlur(ASource: TBitmap; Radius: Integer): TBitmap;
var
  x, y, dx, dy: Integer;
  R, G, B: Double;
  TotalWeight, Weight: Double;
  Sigma: Double;
  C: TColor;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  Sigma := Radius / 3;
  if Sigma = 0 then Sigma := 1;
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      R := 0; G := 0; B := 0;
      TotalWeight := 0;
      
      for dy := -Radius to Radius do
        for dx := -Radius to Radius do
        begin
          Weight := Exp(-(dx * dx + dy * dy) / (2 * Sigma * Sigma));
          
          var NX := Max(0, Min(ASource.Width - 1, x + dx));
          var NY := Max(0, Min(ASource.Height - 1, y + dy));
          C := ASource.Canvas.Pixels[NX, NY];
          
          R := R + GetRValue(C) * Weight;
          G := G + GetGValue(C) * Weight;
          B := B + GetBValue(C) * Weight;
          TotalWeight := TotalWeight + Weight;
        end;
      
      Result.Canvas.Pixels[x, y] := RGB(
        Round(R / TotalWeight),
        Round(G / TotalWeight),
        Round(B / TotalWeight)
      );
    end;
end;

function UnsharpMask(ASource: TBitmap; Radius: Integer; Amount: Double;
  Threshold: Integer): TBitmap;
var
  Blurred: TBitmap;
  x, y: Integer;
  OrigC, BlurC: TColor;
  R, G, B: Integer;
  DiffR, DiffG, DiffB: Integer;
begin
  Blurred := GaussianBlur(ASource, Radius);
  try
    Result := TBitmap.Create;
    Result.Width := ASource.Width;
    Result.Height := ASource.Height;
    
    for y := 0 to ASource.Height - 1 do
      for x := 0 to ASource.Width - 1 do
      begin
        OrigC := ASource.Canvas.Pixels[x, y];
        BlurC := Blurred.Canvas.Pixels[x, y];
        
        DiffR := GetRValue(OrigC) - GetRValue(BlurC);
        DiffG := GetGValue(OrigC) - GetGValue(BlurC);
        DiffB := GetBValue(OrigC) - GetBValue(BlurC);
        
        // ใช้ Threshold
        if Abs(DiffR) < Threshold then DiffR := 0;
        if Abs(DiffG) < Threshold then DiffG := 0;
        if Abs(DiffB) < Threshold then DiffB := 0;
        
        R := Max(0, Min(255, Round(GetRValue(OrigC) + Amount * DiffR)));
        G := Max(0, Min(255, Round(GetGValue(OrigC) + Amount * DiffG)));
        B := Max(0, Min(255, Round(GetBValue(OrigC) + Amount * DiffB)));
        
        Result.Canvas.Pixels[x, y] := RGB(R, G, B);
      end;
  finally
    Blurred.Free;
  end;
end;

function MotionBlur(ASource: TBitmap; Distance: Integer; AngleDeg: Double): TBitmap;
var
  x, y, i: Integer;
  R, G, B: Integer;
  Count: Integer;
  C: TColor;
  AngleRad: Double;
  DX, DY: Double;
  SX, SY: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  AngleRad := AngleDeg * Pi / 180;
  DX := Cos(AngleRad);
  DY := Sin(AngleRad);
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      R := 0; G := 0; B := 0;
      Count := 0;
      
      for i := -Distance to Distance do
      begin
        SX := Max(0, Min(ASource.Width - 1, x + Round(i * DX)));
        SY := Max(0, Min(ASource.Height - 1, y + Round(i * DY)));
        C := ASource.Canvas.Pixels[SX, SY];
        R := R + GetRValue(C);
        G := G + GetGValue(C);
        B := B + GetBValue(C);
        Inc(Count);
      end;
      
      Result.Canvas.Pixels[x, y] := RGB(R div Count, G div Count, B div Count);
    end;
end;

function EmbossFilter(ASource: TBitmap): TBitmap;
var
  x, y, i, j: Integer;
  G: Integer;
  Kernel: array[-1..1, -1..1] of Integer = ((-1, -1, 0), (-1, 0, 1), (0, 1, 1));
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  
  for y := 1 to ASource.Height - 2 do
    for x := 1 to ASource.Width - 2 do
    begin
      G := 128;  // Bias
      for j := -1 to 1 do
        for i := -1 to 1 do
        begin
          var C := ASource.Canvas.Pixels[x + i, y + j];
          var Gray := Round(0.2126 * GetRValue(C) + 0.7152 * GetGValue(C) + 0.0722 * GetBValue(C));
          G := G + Kernel[j, i] * Gray;
        end;
      
      G := Max(0, Min(255, G));
      Result.Canvas.Pixels[x, y] := RGB(G, G, G);
    end;
end;

end.
```

---

## 37.9 Color Replacement

```pascal
// แทนที่สีเฉพาะในภาพ
function ReplaceColor(ASource: TBitmap; OldColor, NewColor: TColor; 
                       Tolerance: Integer): TBitmap;
var
  x, y: Integer;
  C: TColor;
  DR, DG, DB: Integer;
  Distance: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  Result.Canvas.Draw(0, 0, ASource);
  
  for y := 0 to ASource.Height - 1 do
    for x := 0 to ASource.Width - 1 do
    begin
      C := ASource.Canvas.Pixels[x, y];
      
      DR := GetRValue(C) - GetRValue(OldColor);
      DG := GetGValue(C) - GetGValue(OldColor);
      DB := GetBValue(C) - GetBValue(OldColor);
      
      Distance := Round(Sqrt(DR * DR + DG * DG + DB * DB));
      
      if Distance <= Tolerance then
        Result.Canvas.Pixels[x, y] := NewColor;
    end;
end;

// Green Screen / Chroma Key
function ChromaKey(AForeground: TBitmap; ABackground: TBitmap; 
                    KeyColor: TColor; Tolerance: Integer): TBitmap;
var
  x, y: Integer;
  C: TColor;
  DR, DG, DB: Integer;
  Distance: Integer;
  Alpha: Double;
begin
  Result := TBitmap.Create;
  Result.Width := AForeground.Width;
  Result.Height := AForeground.Height;
  
  // วาดพื้นหลังก่อน
  Result.Canvas.StretchDraw(Rect(0, 0, Result.Width, Result.Height), ABackground);
  
  for y := 0 to AForeground.Height - 1 do
    for x := 0 to AForeground.Width - 1 do
    begin
      C := AForeground.Canvas.Pixels[x, y];
      
      DR := GetRValue(C) - GetRValue(KeyColor);
      DG := GetGValue(C) - GetGValue(KeyColor);
      DB := GetBValue(C) - GetBValue(KeyColor);
      
      Distance := Round(Sqrt(DR * DR + DG * DG + DB * DB));
      
      if Distance <= Tolerance then
        Continue  // ใช้พื้นหลัง
      else if Distance <= Tolerance * 2 then
      begin
        // Blend บริเวณขอบ
        Alpha := (Distance - Tolerance) / Tolerance;
        var BGC := Result.Canvas.Pixels[x, y];
        Result.Canvas.Pixels[x, y] := RGB(
          Round(GetRValue(C) * Alpha + GetRValue(BGC) * (1 - Alpha)),
          Round(GetGValue(C) * Alpha + GetGValue(BGC) * (1 - Alpha)),
          Round(GetBValue(C) * Alpha + GetBValue(BGC) * (1 - Alpha))
        );
      end else
        Result.Canvas.Pixels[x, y] := C;
    end;
end;
```

---

## 37.10 Watermark

```pascal
unit Watermark;

interface

uses
  Classes, SysUtils, Graphics;

// ใส่ Text Watermark
function AddTextWatermark(ASource: TBitmap; const WatermarkText: string;
                            FontSize: Integer; Opacity: Double;
                            Position: Integer): TBitmap;
// Position: 0=TopLeft, 1=TopRight, 2=Center, 3=BottomLeft, 4=BottomRight

// ใส่ Image Watermark
function AddImageWatermark(ASource: TBitmap; AWatermark: TBitmap;
                             Opacity: Double; Position: Integer): TBitmap;

// ใส่ Diagonal Text Watermark (ทั่วภาพ)
function AddDiagonalWatermark(ASource: TBitmap; const WatermarkText: string;
                                FontSize: Integer; Opacity: Double): TBitmap;

implementation

function BlendColor(C1, C2: TColor; Alpha: Double): TColor;
begin
  Result := RGB(
    Round(GetRValue(C1) * Alpha + GetRValue(C2) * (1 - Alpha)),
    Round(GetGValue(C1) * Alpha + GetGValue(C2) * (1 - Alpha)),
    Round(GetBValue(C1) * Alpha + GetBValue(C2) * (1 - Alpha))
  );
end;

function AddTextWatermark(ASource: TBitmap; const WatermarkText: string;
  FontSize: Integer; Opacity: Double; Position: Integer): TBitmap;
var
  WMBitmap: TBitmap;
  TW, TH: Integer;
  WX, WY: Integer;
  x, y: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  Result.Canvas.Draw(0, 0, ASource);
  
  // สร้าง Watermark bitmap
  WMBitmap := TBitmap.Create;
  try
    WMBitmap.Width := ASource.Width;
    WMBitmap.Height := ASource.Height;
    WMBitmap.Canvas.Brush.Color := clBlack;
    WMBitmap.Canvas.FillRect(Rect(0, 0, WMBitmap.Width, WMBitmap.Height));
    
    WMBitmap.Canvas.Font.Name := 'Arial';
    WMBitmap.Canvas.Font.Size := FontSize;
    WMBitmap.Canvas.Font.Style := [fsBold];
    WMBitmap.Canvas.Font.Color := clWhite;
    WMBitmap.Canvas.Brush.Style := bsClear;
    
    TW := WMBitmap.Canvas.TextWidth(WatermarkText);
    TH := WMBitmap.Canvas.TextHeight(WatermarkText);
    
    // คำนวณตำแหน่ง
    case Position of
      0: begin WX := 10; WY := 10; end;  // TopLeft
      1: begin WX := ASource.Width - TW - 10; WY := 10; end;  // TopRight
      2: begin WX := (ASource.Width - TW) div 2; WY := (ASource.Height - TH) div 2; end;
      3: begin WX := 10; WY := ASource.Height - TH - 10; end;  // BottomLeft
      else begin WX := ASource.Width - TW - 10; WY := ASource.Height - TH - 10; end;
    end;
    
    WMBitmap.Canvas.TextOut(WX, WY, WatermarkText);
    
    // Blend Watermark
    for y := WY to Min(WY + TH + 4, ASource.Height - 1) do
      for x := WX to Min(WX + TW + 4, ASource.Width - 1) do
      begin
        if WMBitmap.Canvas.Pixels[x, y] <> clBlack then
          Result.Canvas.Pixels[x, y] := BlendColor(
            WMBitmap.Canvas.Pixels[x, y],
            ASource.Canvas.Pixels[x, y],
            Opacity
          );
      end;
  finally
    WMBitmap.Free;
  end;
end;

function AddDiagonalWatermark(ASource: TBitmap; const WatermarkText: string;
  FontSize: Integer; Opacity: Double): TBitmap;
var
  WMBitmap: TBitmap;
  x, y: Integer;
  TW, TH: Integer;
  Spacing: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  Result.Canvas.Draw(0, 0, ASource);
  
  WMBitmap := TBitmap.Create;
  try
    WMBitmap.Width := ASource.Width;
    WMBitmap.Height := ASource.Height;
    WMBitmap.Canvas.Brush.Color := clBlack;
    WMBitmap.Canvas.FillRect(Rect(0, 0, WMBitmap.Width, WMBitmap.Height));
    
    WMBitmap.Canvas.Font.Name := 'Arial';
    WMBitmap.Canvas.Font.Size := FontSize;
    WMBitmap.Canvas.Font.Color := clWhite;
    WMBitmap.Canvas.Brush.Style := bsClear;
    
    TW := WMBitmap.Canvas.TextWidth(WatermarkText);
    TH := WMBitmap.Canvas.TextHeight(WatermarkText);
    Spacing := TW + 50;
    
    // วาดซ้ำแบบเฉียง
    y := -TH;
    while y < ASource.Height + Spacing do
    begin
      x := -TW;
      while x < ASource.Width + Spacing do
      begin
        WMBitmap.Canvas.TextOut(x, y, WatermarkText);
        x := x + Spacing;
      end;
      y := y + Spacing;
    end;
    
    // Blend
    for y := 0 to ASource.Height - 1 do
      for x := 0 to ASource.Width - 1 do
      begin
        if WMBitmap.Canvas.Pixels[x, y] <> clBlack then
          Result.Canvas.Pixels[x, y] := BlendColor(
            WMBitmap.Canvas.Pixels[x, y],
            ASource.Canvas.Pixels[x, y],
            Opacity
          );
      end;
  finally
    WMBitmap.Free;
  end;
end;

function AddImageWatermark(ASource: TBitmap; AWatermark: TBitmap;
  Opacity: Double; Position: Integer): TBitmap;
var
  WX, WY: Integer;
  x, y: Integer;
  WMC, SrcC: TColor;
  MaskC: TColor;
begin
  Result := TBitmap.Create;
  Result.Width := ASource.Width;
  Result.Height := ASource.Height;
  Result.Canvas.Draw(0, 0, ASource);
  
  case Position of
    0: begin WX := 10; WY := 10; end;
    1: begin WX := ASource.Width - AWatermark.Width - 10; WY := 10; end;
    2: begin WX := (ASource.Width - AWatermark.Width) div 2;
            WY := (ASource.Height - AWatermark.Height) div 2; end;
    3: begin WX := 10; WY := ASource.Height - AWatermark.Height - 10; end;
    else begin
      WX := ASource.Width - AWatermark.Width - 10;
      WY := ASource.Height - AWatermark.Height - 10;
    end;
  end;
  
  for y := 0 to AWatermark.Height - 1 do
    for x := 0 to AWatermark.Width - 1 do
    begin
      var DX := WX + x;
      var DY := WY + y;
      if (DX < 0) or (DX >= ASource.Width) or
         (DY < 0) or (DY >= ASource.Height) then Continue;
      
      WMC := AWatermark.Canvas.Pixels[x, y];
      SrcC := ASource.Canvas.Pixels[DX, DY];
      
      Result.Canvas.Pixels[DX, DY] := BlendColor(WMC, SrcC, Opacity);
    end;
end;

end.
```

---

## 37.11 การสร้าง Thumbnail

```pascal
unit ThumbnailGenerator;

interface

uses
  Classes, SysUtils, Graphics, Math;

type
  TThumbnailOptions = record
    Width, Height: Integer;
    Crop: Boolean;       // True = Center Crop, False = Fit
    Quality: Integer;    // สำหรับ JPEG
    AddBorder: Boolean;
    BorderColor: TColor;
    BorderWidth: Integer;
  end;

function GenerateThumbnail(const SourceFile, OutputFile: string;
                             Options: TThumbnailOptions): Boolean;

function GenerateThumbnailBitmap(ASource: TBitmap; 
                                  Width, Height: Integer): TBitmap;

implementation

uses
  FPImage, FPReadJPEG, FPWriteJPEG, FPReadPNG, FPWritePNG;

function GenerateThumbnailBitmap(ASource: TBitmap; Width, Height: Integer): TBitmap;
var
  SrcRatio, DstRatio: Double;
  ScaledW, ScaledH: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := Width;
  Result.Height := Height;
  
  // Fit with white background
  Result.Canvas.Brush.Color := clWhite;
  Result.Canvas.FillRect(Rect(0, 0, Width, Height));
  
  SrcRatio := ASource.Width / ASource.Height;
  DstRatio := Width / Height;
  
  if SrcRatio > DstRatio then
  begin
    ScaledW := Width;
    ScaledH := Round(Width / SrcRatio);
  end else
  begin
    ScaledH := Height;
    ScaledW := Round(Height * SrcRatio);
  end;
  
  var DstX := (Width - ScaledW) div 2;
  var DstY := (Height - ScaledH) div 2;
  
  Result.Canvas.StretchDraw(Rect(DstX, DstY, DstX + ScaledW, DstY + ScaledH), ASource);
end;

function GenerateThumbnail(const SourceFile, OutputFile: string;
  Options: TThumbnailOptions): Boolean;
var
  SourcePic: TPicture;
  SourceBmp: TBitmap;
  Thumbnail: TBitmap;
  OutExt: string;
begin
  Result := False;
  
  SourcePic := TPicture.Create;
  try
    try
      SourcePic.LoadFromFile(SourceFile);
    except
      on E: Exception do
      begin
        WriteLn('Error loading: ', E.Message);
        Exit;
      end;
    end;
    
    SourceBmp := TBitmap.Create;
    try
      SourceBmp.Width := SourcePic.Width;
      SourceBmp.Height := SourcePic.Height;
      SourceBmp.Canvas.Draw(0, 0, SourcePic.Graphic);
      
      Thumbnail := GenerateThumbnailBitmap(SourceBmp, Options.Width, Options.Height);
      try
        if Options.AddBorder then
        begin
          Thumbnail.Canvas.Pen.Color := Options.BorderColor;
          Thumbnail.Canvas.Pen.Width := Options.BorderWidth;
          Thumbnail.Canvas.Brush.Style := bsClear;
          Thumbnail.Canvas.Rectangle(0, 0, Options.Width, Options.Height);
        end;
        
        OutExt := LowerCase(ExtractFileExt(OutputFile));
        Thumbnail.SaveToFile(OutputFile);
        Result := True;
      finally
        Thumbnail.Free;
      end;
    finally
      SourceBmp.Free;
    end;
  finally
    SourcePic.Free;
  end;
end;

end.
```

---

## 37.12 Batch Processing

```pascal
unit BatchImageProcess;

interface

uses
  Classes, SysUtils, Graphics, FileUtil, LazFileUtils;

type
  TBatchOperation = (bopResize, bopGrayscale, bopBrightness, bopWatermark);
  
  TBatchOptions = record
    SourceDir: string;
    OutputDir: string;
    Operation: TBatchOperation;
    // สำหรับ Resize
    NewWidth, NewHeight: Integer;
    // สำหรับ Brightness
    BrightnessValue: Integer;
    // สำหรับ Watermark
    WatermarkText: string;
    WatermarkOpacity: Double;
    // รูปแบบที่ประมวลผล
    Extensions: TStringList;
    // Callback
    OnProgress: procedure(const FileName: string; Current, Total: Integer) of object;
  end;

function ProcessBatch(const Options: TBatchOptions): Integer;

implementation

uses
  ImageAdjustments, ImageResize, Watermark;

function ProcessBatch(const Options: TBatchOptions): Integer;
var
  Files: TStringList;
  i: Integer;
  SourceFile, OutputFile: string;
  SourceBmp, ResultBmp: TBitmap;
  Ext: string;
begin
  Result := 0;
  Files := TStringList.Create;
  try
    // หาไฟล์ทั้งหมด
    for i := 0 to Options.Extensions.Count - 1 do
    begin
      var SearchFiles := FindAllFiles(Options.SourceDir, Options.Extensions[i], False);
      Files.AddStrings(SearchFiles);
    end;
    
    // สร้าง Output directory
    if not DirectoryExists(Options.OutputDir) then
      ForceDirectories(Options.OutputDir);
    
    // ประมวลผลแต่ละไฟล์
    for i := 0 to Files.Count - 1 do
    begin
      SourceFile := Files[i];
      OutputFile := Options.OutputDir + PathDelim + 
                     ExtractFileName(SourceFile);
      
      if Assigned(Options.OnProgress) then
        Options.OnProgress(ExtractFileName(SourceFile), i + 1, Files.Count);
      
      SourceBmp := TBitmap.Create;
      ResultBmp := nil;
      
      try
        try
          var Pic := TPicture.Create;
          try
            Pic.LoadFromFile(SourceFile);
            SourceBmp.Width := Pic.Width;
            SourceBmp.Height := Pic.Height;
            SourceBmp.Canvas.Draw(0, 0, Pic.Graphic);
          finally
            Pic.Free;
          end;
          
          case Options.Operation of
            bopResize:
              ResultBmp := ResizeBilinear(SourceBmp, Options.NewWidth, Options.NewHeight);
            
            bopGrayscale:
              ResultBmp := ToGrayscaleLuminance(SourceBmp);
            
            bopBrightness:
              ResultBmp := AdjustBrightness(SourceBmp, Options.BrightnessValue);
            
            bopWatermark:
              ResultBmp := AddDiagonalWatermark(SourceBmp, Options.WatermarkText,
                                                  16, Options.WatermarkOpacity);
          end;
          
          if Assigned(ResultBmp) then
          begin
            ResultBmp.SaveToFile(OutputFile);
            Inc(Result);
          end;
          
        except
          on E: Exception do
            WriteLn('Error processing ', SourceFile, ': ', E.Message);
        end;
        
      finally
        SourceBmp.Free;
        if Assigned(ResultBmp) then ResultBmp.Free;
      end;
    end;
  finally
    Files.Free;
  end;
end;

end.
```

---

## 37.13 ตัวอย่างสมบูรณ์: Photo Editor Application

```pascal
unit PhotoEditor;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, 
  Dialogs, ComCtrls, StdCtrls, ExtCtrls, Buttons,
  Spin, Menus;

type
  TEditHistory = class
  private
    FHistory: array of TBitmap;
    FCurrentIndex: Integer;
    FMaxHistory: Integer;
  public
    constructor Create(MaxHistory: Integer = 20);
    destructor Destroy; override;
    procedure Push(ABitmap: TBitmap);
    function Undo: TBitmap;
    function Redo: TBitmap;
    function CanUndo: Boolean;
    function CanRedo: Boolean;
    procedure Clear;
  end;
  
  TForm1 = class(TForm)
    Panel1: TPanel;
    ScrollBox1: TScrollBox;
    Image1: TImage;
    
    // Tools
    btnOpen: TButton;
    btnSave: TButton;
    btnUndo: TButton;
    btnRedo: TButton;
    
    // Adjustments
    TrackBrightness: TTrackBar;
    TrackContrast: TTrackBar;
    TrackSaturation: TTrackBar;
    
    // Filters
    btnGrayscale: TButton;
    btnSepia: TButton;
    btnBlur: TButton;
    btnSharpen: TButton;
    btnEdge: TButton;
    btnEmboss: TButton;
    
    // Info
    StatusBar1: TStatusBar;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnOpenClick(Sender: TObject);
    procedure btnSaveClick(Sender: TObject);
    procedure btnUndoClick(Sender: TObject);
    procedure btnRedoClick(Sender: TObject);
    procedure btnGrayscaleClick(Sender: TObject);
    procedure btnSepiaClick(Sender: TObject);
    procedure btnBlurClick(Sender: TObject);
    procedure btnSharpenClick(Sender: TObject);
    procedure btnEdgeClick(Sender: TObject);
    procedure btnEmbossClick(Sender: TObject);
    procedure TrackBrightnessChange(Sender: TObject);
    procedure TrackContrastChange(Sender: TObject);
    procedure TrackSaturationChange(Sender: TObject);
  private
    FOriginalBitmap: TBitmap;
    FCurrentBitmap: TBitmap;
    FHistory: TEditHistory;
    FUpdating: Boolean;
    
    procedure LoadImage(const FileName: string);
    procedure ShowCurrentImage;
    procedure ApplyFilter(ANewBitmap: TBitmap);
    procedure UpdateButtons;
    procedure ApplyAdjustments;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

constructor TEditHistory.Create(MaxHistory: Integer);
begin
  inherited Create;
  FMaxHistory := MaxHistory;
  FCurrentIndex := -1;
  SetLength(FHistory, 0);
end;

destructor TEditHistory.Destroy;
var
  i: Integer;
begin
  for i := 0 to High(FHistory) do
    if Assigned(FHistory[i]) then FHistory[i].Free;
  inherited;
end;

procedure TEditHistory.Push(ABitmap: TBitmap);
var
  NewBmp: TBitmap;
begin
  // ลบ history หลัง current index
  var i: Integer;
  for i := FCurrentIndex + 1 to High(FHistory) do
    if Assigned(FHistory[i]) then FHistory[i].Free;
  
  // Trim to max
  if FCurrentIndex >= FMaxHistory - 1 then
  begin
    FHistory[0].Free;
    for i := 1 to FCurrentIndex do
      FHistory[i - 1] := FHistory[i];
    Dec(FCurrentIndex);
  end;
  
  SetLength(FHistory, FCurrentIndex + 2);
  
  NewBmp := TBitmap.Create;
  NewBmp.Assign(ABitmap);
  FHistory[FCurrentIndex + 1] := NewBmp;
  Inc(FCurrentIndex);
end;

function TEditHistory.Undo: TBitmap;
begin
  if not CanUndo then begin Result := nil; Exit; end;
  Dec(FCurrentIndex);
  Result := FHistory[FCurrentIndex];
end;

function TEditHistory.Redo: TBitmap;
begin
  if not CanRedo then begin Result := nil; Exit; end;
  Inc(FCurrentIndex);
  Result := FHistory[FCurrentIndex];
end;

function TEditHistory.CanUndo: Boolean;
begin
  Result := FCurrentIndex > 0;
end;

function TEditHistory.CanRedo: Boolean;
begin
  Result := FCurrentIndex < High(FHistory);
end;

procedure TEditHistory.Clear;
var
  i: Integer;
begin
  for i := 0 to High(FHistory) do
    if Assigned(FHistory[i]) then FHistory[i].Free;
  SetLength(FHistory, 0);
  FCurrentIndex := -1;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FOriginalBitmap := nil;
  FCurrentBitmap := nil;
  FHistory := TEditHistory.Create(20);
  FUpdating := False;
  
  TrackBrightness.Min := -100;
  TrackBrightness.Max := 100;
  TrackBrightness.Position := 0;
  
  TrackContrast.Min := -100;
  TrackContrast.Max := 100;
  TrackContrast.Position := 0;
  
  TrackSaturation.Min := -100;
  TrackSaturation.Max := 100;
  TrackSaturation.Position := 0;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FOriginalBitmap.Free;
  FCurrentBitmap.Free;
  FHistory.Free;
end;

procedure TForm1.LoadImage(const FileName: string);
var
  Pic: TPicture;
begin
  Pic := TPicture.Create;
  try
    Pic.LoadFromFile(FileName);
    
    FreeAndNil(FOriginalBitmap);
    FreeAndNil(FCurrentBitmap);
    
    FOriginalBitmap := TBitmap.Create;
    FOriginalBitmap.Width := Pic.Width;
    FOriginalBitmap.Height := Pic.Height;
    FOriginalBitmap.Canvas.Draw(0, 0, Pic.Graphic);
    
    FCurrentBitmap := TBitmap.Create;
    FCurrentBitmap.Assign(FOriginalBitmap);
    
    FHistory.Clear;
    FHistory.Push(FCurrentBitmap);
    
    FUpdating := True;
    TrackBrightness.Position := 0;
    TrackContrast.Position := 0;
    TrackSaturation.Position := 0;
    FUpdating := False;
    
    ShowCurrentImage;
    UpdateButtons;
    
    StatusBar1.SimpleText := Format('%s (%dx%d)', [
      ExtractFileName(FileName), Pic.Width, Pic.Height
    ]);
  finally
    Pic.Free;
  end;
end;

procedure TForm1.ShowCurrentImage;
begin
  if Assigned(FCurrentBitmap) then
  begin
    Image1.Picture.Assign(FCurrentBitmap);
    Image1.Width := FCurrentBitmap.Width;
    Image1.Height := FCurrentBitmap.Height;
  end;
end;

procedure TForm1.ApplyFilter(ANewBitmap: TBitmap);
begin
  FCurrentBitmap.Free;
  FCurrentBitmap := ANewBitmap;
  FHistory.Push(FCurrentBitmap);
  ShowCurrentImage;
  UpdateButtons;
end;

procedure TForm1.UpdateButtons;
begin
  btnUndo.Enabled := FHistory.CanUndo;
  btnRedo.Enabled := FHistory.CanRedo;
  btnSave.Enabled := Assigned(FCurrentBitmap);
end;

procedure TForm1.ApplyAdjustments;
begin
  if FUpdating or not Assigned(FOriginalBitmap) then Exit;
  
  var Temp1 := AdjustBrightness(FOriginalBitmap, 
                                  Round(TrackBrightness.Position * 2.55));
  var Temp2 := AdjustContrast(Temp1, TrackContrast.Position);
  Temp1.Free;
  var Temp3 := AdjustSaturation(Temp2, TrackSaturation.Position);
  Temp2.Free;
  
  FCurrentBitmap.Free;
  FCurrentBitmap := Temp3;
  ShowCurrentImage;
end;

procedure TForm1.btnOpenClick(Sender: TObject);
var
  OD: TOpenDialog;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.Filter := 'Images|*.png;*.jpg;*.jpeg;*.bmp|All|*.*';
    if OD.Execute then
      LoadImage(OD.FileName);
  finally
    OD.Free;
  end;
end;

procedure TForm1.btnSaveClick(Sender: TObject);
var
  SD: TSaveDialog;
begin
  if not Assigned(FCurrentBitmap) then Exit;
  SD := TSaveDialog.Create(nil);
  try
    SD.Filter := 'PNG|*.png|JPEG|*.jpg|BMP|*.bmp';
    if SD.Execute then
      FCurrentBitmap.SaveToFile(SD.FileName);
  finally
    SD.Free;
  end;
end;

procedure TForm1.btnUndoClick(Sender: TObject);
var
  B: TBitmap;
begin
  B := FHistory.Undo;
  if Assigned(B) then
  begin
    FCurrentBitmap.Free;
    FCurrentBitmap := TBitmap.Create;
    FCurrentBitmap.Assign(B);
    ShowCurrentImage;
    UpdateButtons;
  end;
end;

procedure TForm1.btnRedoClick(Sender: TObject);
var
  B: TBitmap;
begin
  B := FHistory.Redo;
  if Assigned(B) then
  begin
    FCurrentBitmap.Free;
    FCurrentBitmap := TBitmap.Create;
    FCurrentBitmap.Assign(B);
    ShowCurrentImage;
    UpdateButtons;
  end;
end;

procedure TForm1.btnGrayscaleClick(Sender: TObject);
begin
  if not Assigned(FCurrentBitmap) then Exit;
  ApplyFilter(ToGrayscaleLuminance(FCurrentBitmap));
end;

procedure TForm1.btnSepiaClick(Sender: TObject);
begin
  if not Assigned(FCurrentBitmap) then Exit;
  ApplyFilter(ToSepia(FCurrentBitmap));
end;

procedure TForm1.btnBlurClick(Sender: TObject);
begin
  if not Assigned(FCurrentBitmap) then Exit;
  ApplyFilter(GaussianBlur(FCurrentBitmap, 3));
end;

procedure TForm1.btnSharpenClick(Sender: TObject);
begin
  if not Assigned(FCurrentBitmap) then Exit;
  ApplyFilter(UnsharpMask(FCurrentBitmap, 2, 1.5, 10));
end;

procedure TForm1.btnEdgeClick(Sender: TObject);
begin
  if not Assigned(FCurrentBitmap) then Exit;
  ApplyFilter(SobelEdgeDetection(FCurrentBitmap, 80));
end;

procedure TForm1.btnEmbossClick(Sender: TObject);
begin
  if not Assigned(FCurrentBitmap) then Exit;
  ApplyFilter(EmbossFilter(FCurrentBitmap));
end;

procedure TForm1.TrackBrightnessChange(Sender: TObject);
begin
  ApplyAdjustments;
end;

procedure TForm1.TrackContrastChange(Sender: TObject);
begin
  ApplyAdjustments;
end;

procedure TForm1.TrackSaturationChange(Sender: TObject);
begin
  ApplyAdjustments;
end;

end.
```

---

## แบบฝึกหัดบทที่ 37

### ข้อ 1: Image Viewer
สร้าง Image Viewer ที่รองรับ PNG, JPEG, BMP, GIF พร้อม Zoom In/Out

### ข้อ 2: Batch Resize
สร้างโปรแกรม Batch Resize ที่ปรับขนาดภาพทั้ง folder พร้อมแสดง Progress bar

### ข้อ 3: Color Picker
สร้าง Color Picker ที่คลิกที่ภาพแล้วแสดงค่า RGB, HSL, Hex ของสีที่คลิก

### ข้อ 4: Image Comparison
สร้างโปรแกรม Compare Images ที่แสดงภาพสองภาพและความแตกต่าง

```pascal
// แนวทาง:
function ImageDifference(A, B: TBitmap): TBitmap;
var
  x, y: Integer;
  CA, CB: TColor;
  Diff: Integer;
begin
  Result := TBitmap.Create;
  Result.Width := Min(A.Width, B.Width);
  Result.Height := Min(A.Height, B.Height);
  
  for y := 0 to Result.Height - 1 do
    for x := 0 to Result.Width - 1 do
    begin
      CA := A.Canvas.Pixels[x, y];
      CB := B.Canvas.Pixels[x, y];
      Diff := Abs(GetRValue(CA) - GetRValue(CB)) +
               Abs(GetGValue(CA) - GetGValue(CB)) +
               Abs(GetBValue(CA) - GetBValue(CB));
      Diff := Min(255, Diff);
      Result.Canvas.Pixels[x, y] := RGB(Diff, Diff, Diff);
    end;
end;
```

### ข้อ 5: EXIF Reader
อ่านข้อมูล EXIF จากไฟล์ JPEG และแสดง Camera Settings

### ข้อ 6: Image Collage
สร้างโปรแกรมรวมหลายภาพเป็น Collage

### ข้อ 7: Noise Reduction
ใช้ Median Filter เพื่อลด Noise ในภาพ

### ข้อ 8: Image to ASCII Art
แปลงภาพเป็น ASCII Art text

```pascal
// แนวทาง:
const
  ASCIIChars = ' .:-=+*#%@';

function ImageToASCII(ABitmap: TBitmap; Cols, Rows: Integer): string;
var
  x, y: Integer;
  PixW, PixH: Integer;
  R, G, B: Integer;
  Gray: Integer;
  CharIdx: Integer;
  C: TColor;
begin
  Result := '';
  PixW := ABitmap.Width div Cols;
  PixH := ABitmap.Height div Rows;
  
  for y := 0 to Rows - 1 do
  begin
    for x := 0 to Cols - 1 do
    begin
      C := ABitmap.Canvas.Pixels[x * PixW, y * PixH];
      Gray := Round(0.2126 * GetRValue(C) + 0.7152 * GetGValue(C) + 0.0722 * GetBValue(C));
      CharIdx := Round((Length(ASCIIChars) - 1) * Gray / 255);
      Result := Result + ASCIIChars[CharIdx + 1];
    end;
    Result := Result + #13#10;
  end;
end;
```

### ข้อ 9: Sticker Overlay
สร้างโปรแกรมใส่ Sticker (ภาพ PNG แบบโปร่งใส) ลงบนภาพ

### ข้อ 10: Image Slideshow
สร้าง Slideshow แสดงภาพหลายๆ ภาพพร้อม Transition effects

---

## สรุปบทที่ 37

ในบทนี้เราได้เรียนรู้:
1. **การโหลด/บันทึกภาพ** - PNG, JPEG, BMP, GIF, TIFF
2. **Image Resizing** - Nearest, Bilinear, Bicubic interpolation
3. **Image Cropping** - การตัดภาพและ Center Crop
4. **Color Adjustments** - Brightness, Contrast, Saturation, Hue, Gamma
5. **Grayscale** - หลายวิธีในการแปลงขาวดำ
6. **Histogram** - การวิเคราะห์การกระจายของสี
7. **Edge Detection** - Sobel, Roberts, Laplacian, Canny
8. **Blur/Sharpen** - Gaussian, Box Blur, Unsharp Mask
9. **Color Replacement** - การเปลี่ยนสี, Chroma Key
10. **Watermark** - ข้อความและรูปภาพ
11. **Thumbnail** - การสร้าง Thumbnail อย่างมีประสิทธิภาพ
12. **Batch Processing** - การประมวลผลภาพจำนวนมาก
