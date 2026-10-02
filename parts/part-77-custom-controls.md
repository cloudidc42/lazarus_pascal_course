# ตอนที่ 77: Custom Controls ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้าง custom components สำหรับ Lazarus รวมถึงการลงทะเบียนใน component palette และ design-time support

---

## 77.1 Custom Component พื้นฐาน

```pascal
unit custom_button;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, Graphics, LCLType, LMessages,
  LCLIntf, Forms;

type
  TButtonStyle = (bsRounded, bsFlat, bsGradient, bsOutline);
  TButtonState = (bsNormal, bsHover, bsPressed, bsDisabled);

  { TRoundedButton - Custom Button ที่มีลักษณะสวยงาม }
  TRoundedButton = class(TCustomControl)
  private
    FCaption: string;
    FStyle: TButtonStyle;
    FState: TButtonState;
    
    FColorNormal: TColor;
    FColorHover: TColor;
    FColorPressed: TColor;
    FColorText: TColor;
    FColorBorder: TColor;
    
    FCornerRadius: Integer;
    FBorderWidth: Integer;
    FFontBold: Boolean;
    FIconLeft: TBitmap;
    FIconRight: TBitmap;
    
    FOnClick: TNotifyEvent;
    
    procedure SetCaption(const AValue: string);
    procedure SetStyle(AValue: TButtonStyle);
    procedure SetColorNormal(AValue: TColor);
    procedure SetColorHover(AValue: TColor);
    procedure SetCornerRadius(AValue: Integer);
    
    function GetCurrentColor: TColor;
    procedure DrawBackground(ACanvas: TCanvas; const ARect: TRect);
    procedure DrawText(ACanvas: TCanvas; const ARect: TRect);
    procedure DrawBorder(ACanvas: TCanvas; const ARect: TRect);
    procedure DrawRoundedRect(ACanvas: TCanvas; const ARect: TRect; ARadius: Integer);
    
  protected
    procedure Paint; override;
    procedure MouseEnter(var Msg: TLMessage); message LM_MOUSEENTER;
    procedure MouseLeave(var Msg: TLMessage); message LM_MOUSELEAVE;
    procedure MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer); override;
    procedure MouseUp(Button: TMouseButton; Shift: TShiftState; X, Y: Integer); override;
    procedure Click; override;
    procedure KeyDown(var Key: Word; Shift: TShiftState); override;
    
    procedure CMEnabledChanged(var Msg: TLMessage); message CM_ENABLEDCHANGED;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure ApplyStyle(AStyle: TButtonStyle);
    
  published
    property Caption: string read FCaption write SetCaption;
    property Style: TButtonStyle read FStyle write SetStyle default bsRounded;
    property ColorNormal: TColor read FColorNormal write SetColorNormal;
    property ColorHover: TColor read FColorHover write SetColorHover;
    property ColorPressed: TColor read FColorPressed write FColorPressed;
    property ColorText: TColor read FColorText write FColorText;
    property ColorBorder: TColor read FColorBorder write FColorBorder;
    property CornerRadius: Integer read FCornerRadius write SetCornerRadius default 8;
    property BorderWidth: Integer read FBorderWidth write FBorderWidth default 1;
    property FontBold: Boolean read FFontBold write FFontBold default False;
    
    property OnClick: TNotifyEvent read FOnClick write FOnClick;
    property Width default 120;
    property Height default 36;
    property TabStop default True;
    property Cursor default crHandPoint;
  end;

procedure Register;

implementation

procedure Register;
begin
  RegisterComponents('Custom Controls', [TRoundedButton]);
end;

{ TRoundedButton }

constructor TRoundedButton.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  // Defaults
  Width := 120;
  Height := 36;
  TabStop := True;
  Cursor := crHandPoint;
  
  FStyle := bsRounded;
  FState := bsNormal;
  FCaption := 'Button';
  FCornerRadius := 8;
  FBorderWidth := 1;
  FFontBold := False;
  
  // Default colors
  FColorNormal := $00E87722;    // Orange
  FColorHover := $00F09040;     // Lighter orange
  FColorPressed := $00C05010;   // Darker orange
  FColorText := clWhite;
  FColorBorder := $00C05010;
  
  // Enable keyboard focus
  ControlStyle := ControlStyle + [csCaptureMouse];
end;

destructor TRoundedButton.Destroy;
begin
  FIconLeft.Free;
  FIconRight.Free;
  inherited Destroy;
end;

procedure TRoundedButton.SetCaption(const AValue: string);
begin
  if FCaption <> AValue then
  begin
    FCaption := AValue;
    Invalidate;
  end;
end;

procedure TRoundedButton.SetStyle(AValue: TButtonStyle);
begin
  if FStyle <> AValue then
  begin
    FStyle := AValue;
    ApplyStyle(AValue);
    Invalidate;
  end;
end;

procedure TRoundedButton.SetColorNormal(AValue: TColor);
begin
  if FColorNormal <> AValue then
  begin
    FColorNormal := AValue;
    Invalidate;
  end;
end;

procedure TRoundedButton.SetColorHover(AValue: TColor);
begin
  if FColorHover <> AValue then
  begin
    FColorHover := AValue;
    Invalidate;
  end;
end;

procedure TRoundedButton.SetCornerRadius(AValue: Integer);
begin
  if FCornerRadius <> AValue then
  begin
    FCornerRadius := AValue;
    Invalidate;
  end;
end;

procedure TRoundedButton.ApplyStyle(AStyle: TButtonStyle);
begin
  case AStyle of
    bsRounded:
    begin
      FColorNormal := $00E87722;
      FColorHover := $00F09040;
      FColorPressed := $00C05010;
      FColorText := clWhite;
      FCornerRadius := 8;
    end;
    
    bsFlat:
    begin
      FColorNormal := $00F0F0F0;
      FColorHover := $00E0E0E0;
      FColorPressed := $00C0C0C0;
      FColorText := clBlack;
      FCornerRadius := 4;
      FBorderWidth := 0;
    end;
    
    bsGradient:
    begin
      FColorNormal := $00DD5500;
      FColorHover := $00EE7700;
      FColorText := clWhite;
      FCornerRadius := 6;
    end;
    
    bsOutline:
    begin
      FColorNormal := clWhite;
      FColorHover := $00FFF0E8;
      FColorText := $00E87722;
      FColorBorder := $00E87722;
      FCornerRadius := 8;
      FBorderWidth := 2;
    end;
  end;
end;

function TRoundedButton.GetCurrentColor: TColor;
begin
  if not Enabled then
    Result := $00D0D0D0
  else
    case FState of
      bsHover:   Result := FColorHover;
      bsPressed: Result := FColorPressed;
      else       Result := FColorNormal;
    end;
end;

procedure TRoundedButton.DrawRoundedRect(ACanvas: TCanvas; const ARect: TRect; 
  ARadius: Integer);
begin
  with ACanvas, ARect do
  begin
    // Draw rounded rectangle path
    MoveTo(Left + ARadius, Top);
    LineTo(Right - ARadius, Top);
    
    // Top right corner
    ACanvas.Arc(Right - 2 * ARadius, Top, Right, Top + 2 * ARadius, 90, -90);
    
    LineTo(Right, Bottom - ARadius);
    
    // Bottom right corner
    ACanvas.Arc(Right - 2 * ARadius, Bottom - 2 * ARadius, Right, Bottom, 0, -90);
    
    LineTo(Left + ARadius, Bottom);
    
    // Bottom left corner
    ACanvas.Arc(Left, Bottom - 2 * ARadius, Left + 2 * ARadius, Bottom, 270, -90);
    
    LineTo(Left, Top + ARadius);
    
    // Top left corner
    ACanvas.Arc(Left, Top, Left + 2 * ARadius, Top + 2 * ARadius, 180, -90);
  end;
end;

procedure TRoundedButton.DrawBackground(ACanvas: TCanvas; const ARect: TRect);
var
  BGColor: TColor;
  R, G, B: Byte;
  R2, G2, B2: Byte;
  i, Steps: Integer;
  StepRect: TRect;
begin
  BGColor := GetCurrentColor;
  
  if FStyle = bsGradient then
  begin
    // Draw gradient
    Steps := ARect.Height;
    R := Red(BGColor);
    G := Green(BGColor);
    B := Blue(BGColor);
    
    // Lighter at top
    R2 := Min(255, R + 40);
    G2 := Min(255, G + 30);
    B2 := Min(255, B + 20);
    
    for i := 0 to Steps - 1 do
    begin
      var Factor := i / Steps;
      var RC := Round(R2 + (R - R2) * Factor);
      var GC := Round(G2 + (G - G2) * Factor);
      var BC := Round(B2 + (B - B2) * Factor);
      
      ACanvas.Pen.Color := RGBToColor(RC, GC, BC);
      ACanvas.Brush.Color := ACanvas.Pen.Color;
      
      StepRect.Left := ARect.Left;
      StepRect.Top := ARect.Top + i;
      StepRect.Right := ARect.Right;
      StepRect.Bottom := StepRect.Top + 1;
      
      ACanvas.FillRect(StepRect);
    end;
  end
  else
  begin
    ACanvas.Brush.Color := BGColor;
    ACanvas.Brush.Style := bsSolid;
    ACanvas.FillRect(ARect);
  end;
end;

procedure TRoundedButton.DrawBorder(ACanvas: TCanvas; const ARect: TRect);
var
  BorderColor: TColor;
begin
  if FBorderWidth <= 0 then Exit;
  
  if not Enabled then
    BorderColor := clGray
  else if Focused then
    BorderColor := clHighlight
  else
    BorderColor := FColorBorder;
    
  ACanvas.Pen.Color := BorderColor;
  ACanvas.Pen.Width := FBorderWidth;
  ACanvas.Brush.Style := bsClear;
  
  var R := ARect;
  InflateRect(R, -FBorderWidth div 2, -FBorderWidth div 2);
  
  if FCornerRadius > 0 then
    ACanvas.RoundRect(R.Left, R.Top, R.Right, R.Bottom, 
      FCornerRadius * 2, FCornerRadius * 2)
  else
    ACanvas.Rectangle(R);
end;

procedure TRoundedButton.DrawText(ACanvas: TCanvas; const ARect: TRect);
var
  TextRect: TRect;
  Flags: Cardinal;
begin
  ACanvas.Font.Assign(Font);
  ACanvas.Font.Color := IfThen(not Enabled, clGray, FColorText);
  
  if FFontBold then
    ACanvas.Font.Style := ACanvas.Font.Style + [fsBold];
  
  TextRect := ARect;
  Flags := DT_CENTER or DT_VCENTER or DT_SINGLELINE;
  
  DrawText(ACanvas.Handle, PChar(FCaption), Length(FCaption), TextRect, Flags);
end;

procedure TRoundedButton.Paint;
var
  R: TRect;
  Bmp: TBitmap;
begin
  // Double-buffer painting
  Bmp := TBitmap.Create;
  try
    Bmp.Width := Width;
    Bmp.Height := Height;
    Bmp.Canvas.Brush.Color := Parent.Brush.Color;
    Bmp.Canvas.FillRect(Rect(0, 0, Width, Height));
    
    R := Rect(0, 0, Width, Height);
    
    // Clip to rounded corners
    if FCornerRadius > 0 then
    begin
      Bmp.Canvas.Pen.Style := psNull;
      Bmp.Canvas.Brush.Color := GetCurrentColor;
      Bmp.Canvas.RoundRect(R.Left, R.Top, R.Right, R.Bottom,
        FCornerRadius * 2, FCornerRadius * 2);
    end
    else
      DrawBackground(Bmp.Canvas, R);
    
    DrawBorder(Bmp.Canvas, R);
    DrawText(Bmp.Canvas, R);
    
    // Draw focus indicator
    if Focused and Enabled then
    begin
      var FocusR := R;
      InflateRect(FocusR, -4, -4);
      Bmp.Canvas.Pen.Color := clWhite;
      Bmp.Canvas.Pen.Width := 1;
      Bmp.Canvas.Pen.Style := psDot;
      Bmp.Canvas.Brush.Style := bsClear;
      Bmp.Canvas.Rectangle(FocusR);
    end;
    
    Canvas.Draw(0, 0, Bmp);
    
  finally
    Bmp.Free;
  end;
end;

procedure TRoundedButton.MouseEnter(var Msg: TLMessage);
begin
  if Enabled then
  begin
    FState := bsHover;
    Invalidate;
  end;
end;

procedure TRoundedButton.MouseLeave(var Msg: TLMessage);
begin
  FState := bsNormal;
  Invalidate;
end;

procedure TRoundedButton.MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  inherited MouseDown(Button, Shift, X, Y);
  
  if Button = mbLeft then
  begin
    FState := bsPressed;
    Invalidate;
    SetFocus;
  end;
end;

procedure TRoundedButton.MouseUp(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  inherited MouseUp(Button, Shift, X, Y);
  
  if Button = mbLeft then
  begin
    FState := IfThen(PtInRect(ClientRect, Point(X, Y)), bsHover, bsNormal);
    Invalidate;
  end;
end;

procedure TRoundedButton.Click;
begin
  inherited Click;
  if Assigned(FOnClick) then
    FOnClick(Self);
end;

procedure TRoundedButton.KeyDown(var Key: Word; Shift: TShiftState);
begin
  if Key in [VK_SPACE, VK_RETURN] then
  begin
    FState := bsPressed;
    Invalidate;
    Click;
    FState := bsNormal;
    Invalidate;
    Key := 0;
  end
  else
    inherited KeyDown(Key, Shift);
end;

procedure TRoundedButton.CMEnabledChanged(var Msg: TLMessage);
begin
  if not Enabled then
    FState := bsDisabled
  else
    FState := bsNormal;
  Invalidate;
end;

end.
```

---

## 77.2 Custom Rating Stars Control

```pascal
unit rating_control;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, Graphics, LCLType;

type
  TRatingControl = class(TCustomControl)
  private
    FMaxRating: Integer;
    FRating: Integer;
    FHoverRating: Integer;
    FStarSize: Integer;
    FSpacing: Integer;
    FColorFilled: TColor;
    FColorEmpty: TColor;
    FColorHover: TColor;
    FReadOnly: Boolean;
    FOnChange: TNotifyEvent;
    
    procedure SetMaxRating(AValue: Integer);
    procedure SetRating(AValue: Integer);
    procedure SetStarSize(AValue: Integer);
    
    procedure DrawStar(ACanvas: TCanvas; ACenterX, ACenterY, ARadius: Integer; 
      AColor: TColor; AFilled: Boolean);
    function GetStarAtX(AX: Integer): Integer;
    
  protected
    procedure Paint; override;
    procedure MouseMove(Shift: TShiftState; X, Y: Integer); override;
    procedure MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer); override;
    procedure MouseLeave; override;
    
  public
    constructor Create(AOwner: TComponent); override;
    
  published
    property MaxRating: Integer read FMaxRating write SetMaxRating default 5;
    property Rating: Integer read FRating write SetRating default 0;
    property StarSize: Integer read FStarSize write SetStarSize default 24;
    property Spacing: Integer read FSpacing write FSpacing default 4;
    property ColorFilled: TColor read FColorFilled write FColorFilled default $0000BBFF;
    property ColorEmpty: TColor read FColorEmpty write FColorEmpty default $00C0C0C0;
    property ColorHover: TColor read FColorHover write FColorHover default $0040DDFF;
    property ReadOnly: Boolean read FReadOnly write FReadOnly default False;
    property OnChange: TNotifyEvent read FOnChange write FOnChange;
    property Height default 32;
  end;

procedure Register;

implementation

uses
  Math;

procedure Register;
begin
  RegisterComponents('Custom Controls', [TRatingControl]);
end;

constructor TRatingControl.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FMaxRating := 5;
  FRating := 0;
  FHoverRating := -1;
  FStarSize := 24;
  FSpacing := 4;
  FColorFilled := $0000BBFF;  // Gold-ish yellow
  FColorEmpty := $00C0C0C0;
  FColorHover := $0040DDFF;
  FReadOnly := False;
  
  Height := 32;
  Cursor := crHandPoint;
  
  // Recalculate width
  SetMaxRating(5);
end;

procedure TRatingControl.SetMaxRating(AValue: Integer);
begin
  if AValue < 1 then AValue := 1;
  FMaxRating := AValue;
  Width := AValue * (FStarSize + FSpacing) + FSpacing;
  Invalidate;
end;

procedure TRatingControl.SetRating(AValue: Integer);
begin
  if AValue < 0 then AValue := 0;
  if AValue > FMaxRating then AValue := FMaxRating;
  
  if FRating <> AValue then
  begin
    FRating := AValue;
    Invalidate;
    if Assigned(FOnChange) then FOnChange(Self);
  end;
end;

procedure TRatingControl.SetStarSize(AValue: Integer);
begin
  if AValue < 8 then AValue := 8;
  FStarSize := AValue;
  Width := FMaxRating * (FStarSize + FSpacing) + FSpacing;
  Height := FStarSize + 8;
  Invalidate;
end;

procedure TRatingControl.DrawStar(ACanvas: TCanvas; ACenterX, ACenterY, ARadius: Integer;
  AColor: TColor; AFilled: Boolean);
var
  Points: array[0..9] of TPoint;
  Angle: Double;
  i: Integer;
  InnerRadius: Integer;
begin
  InnerRadius := Round(ARadius * 0.4);
  
  for i := 0 to 4 do
  begin
    // Outer point
    Angle := (i * 72 - 90) * Pi / 180;
    Points[i * 2].X := Round(ACenterX + ARadius * Cos(Angle));
    Points[i * 2].Y := Round(ACenterY + ARadius * Sin(Angle));
    
    // Inner point
    Angle := ((i * 72 + 36) - 90) * Pi / 180;
    Points[i * 2 + 1].X := Round(ACenterX + InnerRadius * Cos(Angle));
    Points[i * 2 + 1].Y := Round(ACenterY + InnerRadius * Sin(Angle));
  end;
  
  ACanvas.Pen.Color := AColor;
  ACanvas.Pen.Width := 1;
  
  if AFilled then
  begin
    ACanvas.Brush.Color := AColor;
    ACanvas.Brush.Style := bsSolid;
  end
  else
  begin
    ACanvas.Brush.Style := bsClear;
    ACanvas.Pen.Color := FColorEmpty;
  end;
  
  ACanvas.Polygon(Points);
end;

procedure TRatingControl.Paint;
var
  i: Integer;
  CenterX, CenterY: Integer;
  StarColor: TColor;
  IsFilled: Boolean;
  ActiveRating: Integer;
begin
  Canvas.Brush.Color := Parent.Brush.Color;
  Canvas.FillRect(ClientRect);
  
  CenterY := Height div 2;
  ActiveRating := IfThen(FHoverRating >= 0, FHoverRating, FRating);
  
  for i := 1 to FMaxRating do
  begin
    CenterX := FSpacing + (i - 1) * (FStarSize + FSpacing) + FStarSize div 2;
    
    IsFilled := i <= ActiveRating;
    
    if FHoverRating >= 0 then
    begin
      if i <= FHoverRating then
        StarColor := FColorHover
      else
        StarColor := FColorEmpty;
      IsFilled := i <= FHoverRating;
    end
    else
    begin
      StarColor := IfThen(IsFilled, FColorFilled, FColorEmpty);
    end;
    
    DrawStar(Canvas, CenterX, CenterY, FStarSize div 2, StarColor, IsFilled);
  end;
end;

function TRatingControl.GetStarAtX(AX: Integer): Integer;
begin
  Result := (AX - FSpacing) div (FStarSize + FSpacing) + 1;
  if (Result < 1) or (Result > FMaxRating) then
    Result := -1;
end;

procedure TRatingControl.MouseMove(Shift: TShiftState; X, Y: Integer);
begin
  inherited MouseMove(Shift, X, Y);
  
  if not FReadOnly then
  begin
    var Star := GetStarAtX(X);
    if Star <> FHoverRating then
    begin
      FHoverRating := Star;
      Invalidate;
    end;
  end;
end;

procedure TRatingControl.MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  inherited MouseDown(Button, Shift, X, Y);
  
  if (Button = mbLeft) and not FReadOnly then
  begin
    var Star := GetStarAtX(X);
    if Star > 0 then
    begin
      if Star = FRating then
        SetRating(0)  // Click same rating to clear
      else
        SetRating(Star);
    end;
  end;
end;

procedure TRatingControl.MouseLeave;
begin
  FHoverRating := -1;
  Invalidate;
end;

end.
```

---

## 77.3 Package Registration

```pascal
unit custom_controls_reg;

{$mode objfpc}{$H+}

interface

procedure Register;

implementation

uses
  Classes,
  custom_button,
  rating_control;

procedure Register;
begin
  RegisterComponents('Custom Controls', [
    TRoundedButton,
    TRatingControl
  ]);
end;

end.
```

```xml
<!-- custom_controls.lpk - Lazarus Package -->
<?xml version="1.0" encoding="UTF-8"?>
<CONFIG>
  <Package Version="5">
    <PathDelim Value="/"/>
    <Name Value="custom_controls"/>
    <Author Value="Your Name"/>
    <CompilerOptions>
      <Version Value="11"/>
      <SearchPaths>
        <UnitOutputDirectory Value="lib/$(TargetCPU)-$(TargetOS)"/>
      </SearchPaths>
    </CompilerOptions>
    <Description Value="Custom Controls Package for Lazarus"/>
    <License Value="MIT"/>
    <Version Major="1"/>
    <Files>
      <Item>
        <Filename Value="custom_button.pas"/>
        <UnitName Value="custom_button"/>
      </Item>
      <Item>
        <Filename Value="rating_control.pas"/>
        <UnitName Value="rating_control"/>
      </Item>
      <Item>
        <Filename Value="custom_controls_reg.pas"/>
        <HasRegisterProc Value="True"/>
        <UnitName Value="custom_controls_reg"/>
      </Item>
    </Files>
    <RequiredPkgs>
      <Item>
        <PackageName Value="LCL"/>
      </Item>
    </RequiredPkgs>
    <UsageOptions>
      <UnitPath Value="$(PkgOutDir)"/>
    </UsageOptions>
    <PublishOptions>
      <Version Value="2"/>
    </PublishOptions>
  </Package>
</CONFIG>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Custom Component** - สร้าง TRoundedButton ที่มีลักษณะสวยงาม
2. **Double Buffering** - ป้องกัน flickering ขณะ paint
3. **Mouse Events** - จัดการ hover, click states
4. **Drawing** - วาด rounded rect, gradient, stars
5. **Component Registration** - ลงทะเบียนใน component palette
6. **Lazarus Package** - บรรจุ components ใน .lpk

Custom components ช่วยให้ทีมสร้าง UI library ที่ consistent และนำกลับมาใช้ใหม่ได้ทั่วทั้ง project
