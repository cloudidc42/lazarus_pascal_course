# ตอนที่ 79: Animations ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้าง animations ในแอปพลิเคชัน Pascal/Lazarus ตั้งแต่ timer-based animations, property animations ไปจนถึง easing functions

---

## 79.1 Animation Engine พื้นฐาน

```pascal
unit animation_engine;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, ExtCtrls, Math, Graphics;

type
  // Easing functions
  TEasingType = (
    etLinear,
    etEaseIn, etEaseOut, etEaseInOut,
    etBounceIn, etBounceOut, etBounceInOut,
    etElasticIn, etElasticOut, etElasticInOut,
    etBackIn, etBackOut, etBackInOut,
    etSineIn, etSineOut, etSineInOut,
    etExpoIn, etExpoOut, etExpoInOut
  );

  // Easing function type
  TEasingFunc = function(T: Double): Double;

  TAnimationState = (asIdle, asRunning, asPaused, asCompleted);

  TAnimationCallback = procedure(AValue: Double; AProgress: Double) of object;
  TAnimationComplete = procedure of object;

  // Base animation class
  TAnimation = class
  private
    FStartValue: Double;
    FEndValue: Double;
    FDuration: Integer;  // milliseconds
    FElapsed: Integer;
    FState: TAnimationState;
    FEasing: TEasingType;
    FLoop: Boolean;
    FAutoReverse: Boolean;
    FReversing: Boolean;
    
    FOnUpdate: TAnimationCallback;
    FOnComplete: TAnimationComplete;
    
    FTimer: TTimer;
    FLastTick: Int64;
    
    procedure TimerTick(Sender: TObject);
    function ApplyEasing(T: Double): Double;
    
  public
    constructor Create(AStartValue, AEndValue: Double; ADuration: Integer;
      AEasing: TEasingType = etEaseInOut);
    destructor Destroy; override;
    
    procedure Start;
    procedure Pause;
    procedure Resume;
    procedure Stop;
    procedure Reset;
    
    function GetCurrentValue: Double;
    function GetProgress: Double;
    
    property StartValue: Double read FStartValue write FStartValue;
    property EndValue: Double read FEndValue write FEndValue;
    property Duration: Integer read FDuration write FDuration;
    property State: TAnimationState read FState;
    property Easing: TEasingType read FEasing write FEasing;
    property Loop: Boolean read FLoop write FLoop;
    property AutoReverse: Boolean read FAutoReverse write FAutoReverse;
    property OnUpdate: TAnimationCallback read FOnUpdate write FOnUpdate;
    property OnComplete: TAnimationComplete read FOnComplete write FOnComplete;
  end;

  // Property animation (animate any numeric property)
  TPropertyAnimation = class(TAnimation)
  private
    FTarget: TObject;
    FPropertyName: string;
    
    procedure UpdateProperty(AValue: Double; AProgress: Double);
    
  public
    constructor Create(ATarget: TObject; const AProperty: string;
      AFrom, ATo: Double; ADuration: Integer; AEasing: TEasingType = etEaseInOut);
  end;

  // Sequence of animations
  TAnimationSequence = class
  private
    FAnimations: TList;
    FCurrentIndex: Integer;
    FOnComplete: TAnimationComplete;
    
    procedure AnimComplete;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Add(AAnimation: TAnimation);
    procedure Start;
    procedure Stop;
    
    property OnComplete: TAnimationComplete read FOnComplete write FOnComplete;
  end;

  // Parallel animations
  TAnimationGroup = class
  private
    FAnimations: TList;
    FCompletedCount: Integer;
    FOnComplete: TAnimationComplete;
    
    procedure AnimComplete;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Add(AAnimation: TAnimation);
    procedure StartAll;
    procedure StopAll;
    
    property OnComplete: TAnimationComplete read FOnComplete write FOnComplete;
  end;

// Easing functions
function EaseLinear(T: Double): Double;
function EaseIn(T: Double): Double;
function EaseOut(T: Double): Double;
function EaseInOut(T: Double): Double;
function EaseBounceOut(T: Double): Double;
function EaseBounceIn(T: Double): Double;
function EaseElasticOut(T: Double): Double;

implementation

uses
  LCLIntf, RTTI;

// === Easing Functions ===

function EaseLinear(T: Double): Double;
begin
  Result := T;
end;

function EaseIn(T: Double): Double;
begin
  Result := T * T;
end;

function EaseOut(T: Double): Double;
begin
  Result := -T * (T - 2);
end;

function EaseInOut(T: Double): Double;
begin
  if T < 0.5 then
    Result := 2 * T * T
  else
  begin
    T := T * 2 - 1;
    Result := 0.5 * (-T * (T - 2) + 1);
  end;
end;

function EaseBounceOut(T: Double): Double;
const
  N1 = 7.5625;
  D1 = 2.75;
begin
  if T < 1 / D1 then
    Result := N1 * T * T
  else if T < 2 / D1 then
  begin
    T := T - 1.5 / D1;
    Result := N1 * T * T + 0.75;
  end
  else if T < 2.5 / D1 then
  begin
    T := T - 2.25 / D1;
    Result := N1 * T * T + 0.9375;
  end
  else
  begin
    T := T - 2.625 / D1;
    Result := N1 * T * T + 0.984375;
  end;
end;

function EaseBounceIn(T: Double): Double;
begin
  Result := 1 - EaseBounceOut(1 - T);
end;

function EaseElasticOut(T: Double): Double;
const
  C4 = 2 * Pi / 3;
begin
  if T <= 0 then Result := 0
  else if T >= 1 then Result := 1
  else
    Result := Power(2, -10 * T) * Sin((T * 10 - 0.75) * C4) + 1;
end;

function EaseBackIn(T: Double): Double;
const
  C1 = 1.70158;
  C3 = C1 + 1;
begin
  Result := C3 * T * T * T - C1 * T * T;
end;

function EaseBackOut(T: Double): Double;
const
  C1 = 1.70158;
  C3 = C1 + 1;
begin
  T := T - 1;
  Result := 1 + C3 * Power(T, 3) + C1 * Power(T, 2);
end;

function EaseSineInOut(T: Double): Double;
begin
  Result := -(Cos(Pi * T) - 1) / 2;
end;

function EaseExpoOut(T: Double): Double;
begin
  if T >= 1 then Result := 1
  else Result := 1 - Power(2, -10 * T);
end;

{ TAnimation }

constructor TAnimation.Create(AStartValue, AEndValue: Double; ADuration: Integer;
  AEasing: TEasingType);
begin
  inherited Create;
  FStartValue := AStartValue;
  FEndValue := AEndValue;
  FDuration := ADuration;
  FEasing := AEasing;
  FState := asIdle;
  FElapsed := 0;
  FLoop := False;
  FAutoReverse := False;
  FReversing := False;
  
  FTimer := TTimer.Create(nil);
  FTimer.Interval := 16;  // ~60fps
  FTimer.Enabled := False;
  FTimer.OnTimer := @TimerTick;
  
  FLastTick := GetTickCount64;
end;

destructor TAnimation.Destroy;
begin
  FTimer.Free;
  inherited Destroy;
end;

function TAnimation.ApplyEasing(T: Double): Double;
begin
  case FEasing of
    etLinear:       Result := EaseLinear(T);
    etEaseIn:       Result := EaseIn(T);
    etEaseOut:      Result := EaseOut(T);
    etEaseInOut:    Result := EaseInOut(T);
    etBounceIn:     Result := EaseBounceIn(T);
    etBounceOut:    Result := EaseBounceOut(T);
    etBounceInOut:
      if T < 0.5 then Result := EaseBounceIn(T * 2) / 2
      else Result := (EaseBounceOut(T * 2 - 1) + 1) / 2;
    etElasticOut:   Result := EaseElasticOut(T);
    etBackIn:       Result := EaseBackIn(T);
    etBackOut:      Result := EaseBackOut(T);
    etSineInOut:    Result := EaseSineInOut(T);
    etExpoOut:      Result := EaseExpoOut(T);
    else            Result := T;
  end;
end;

procedure TAnimation.TimerTick(Sender: TObject);
var
  Now64: Int64;
  Delta: Integer;
  T, EasedT, Value: Double;
begin
  Now64 := GetTickCount64;
  Delta := Now64 - FLastTick;
  FLastTick := Now64;
  
  Inc(FElapsed, Delta);
  
  if FElapsed >= FDuration then
  begin
    FElapsed := FDuration;
    FState := asCompleted;
    FTimer.Enabled := False;
    
    // Set final value
    var FinalValue := IfThen(FReversing, FStartValue, FEndValue);
    if Assigned(FOnUpdate) then
      FOnUpdate(FinalValue, 1.0);
      
    if FLoop then
    begin
      if FAutoReverse then
        FReversing := not FReversing;
      FElapsed := 0;
      FState := asRunning;
      FTimer.Enabled := True;
    end
    else if Assigned(FOnComplete) then
      FOnComplete;
      
    Exit;
  end;
  
  // Calculate progress and value
  T := FElapsed / FDuration;
  EasedT := ApplyEasing(T);
  
  if FReversing then
    Value := FEndValue + (FStartValue - FEndValue) * EasedT
  else
    Value := FStartValue + (FEndValue - FStartValue) * EasedT;
  
  if Assigned(FOnUpdate) then
    FOnUpdate(Value, T);
end;

procedure TAnimation.Start;
begin
  FState := asRunning;
  FElapsed := 0;
  FReversing := False;
  FLastTick := GetTickCount64;
  FTimer.Enabled := True;
end;

procedure TAnimation.Pause;
begin
  if FState = asRunning then
  begin
    FState := asPaused;
    FTimer.Enabled := False;
  end;
end;

procedure TAnimation.Resume;
begin
  if FState = asPaused then
  begin
    FState := asRunning;
    FLastTick := GetTickCount64;
    FTimer.Enabled := True;
  end;
end;

procedure TAnimation.Stop;
begin
  FTimer.Enabled := False;
  FState := asIdle;
  FElapsed := 0;
end;

procedure TAnimation.Reset;
begin
  Stop;
  FElapsed := 0;
  FReversing := False;
end;

function TAnimation.GetCurrentValue: Double;
var
  T: Double;
begin
  if FDuration = 0 then
    Result := FEndValue
  else
  begin
    T := FElapsed / FDuration;
    Result := FStartValue + (FEndValue - FStartValue) * ApplyEasing(T);
  end;
end;

function TAnimation.GetProgress: Double;
begin
  if FDuration = 0 then Result := 1.0
  else Result := FElapsed / FDuration;
end;

end.
```

---

## 79.2 UI Transition Effects

```pascal
unit ui_transitions;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, Forms, Graphics, ExtCtrls,
  animation_engine;

type
  TTransitionType = (
    ttFadeIn, ttFadeOut,
    ttSlideLeft, ttSlideRight, ttSlideUp, ttSlideDown,
    ttScale, ttZoom,
    ttFlip
  );

  TUITransition = class
  private
    FControl: TControl;
    FAnimation: TAnimation;
    FTransType: TTransitionType;
    FOrigRect: TRect;
    FOnComplete: TNotifyEvent;
    
    procedure AnimUpdate(AValue: Double; AProgress: Double);
    procedure AnimComplete;
    
    procedure ApplyFade(AValue: Double);
    procedure ApplySlide(AValue: Double; ADir: TTransitionType);
    procedure ApplyScale(AValue: Double);
    
  public
    constructor Create(AControl: TControl);
    destructor Destroy; override;
    
    procedure Play(AType: TTransitionType; ADuration: Integer = 300;
      AEasing: TEasingType = etEaseInOut);
    procedure Stop;
    
    class procedure Animate(AControl: TControl; AType: TTransitionType;
      ADuration: Integer = 300; AOnComplete: TNotifyEvent = nil);
    
    property OnComplete: TNotifyEvent read FOnComplete write FOnComplete;
  end;

  // Page transitions (switch between panels)
  TPageTransition = class
  private
    FFromPage: TControl;
    FToPage: TControl;
    FTransType: TTransitionType;
    FDuration: Integer;
    FAnimFrom: TAnimation;
    FAnimTo: TAnimation;
    FOnComplete: TNotifyEvent;
    
    procedure AnimComplete;
    procedure AnimFromUpdate(AValue: Double; AProgress: Double);
    procedure AnimToUpdate(AValue: Double; AProgress: Double);
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Transition(AFrom, ATo: TControl; AType: TTransitionType;
      ADuration: Integer = 400);
    
    property OnComplete: TNotifyEvent read FOnComplete write FOnComplete;
  end;

  // Notification pop-in
  TNotificationPopup = class
  private
    FPanel: TPanel;
    FTimer: TTimer;
    FAnimation: TAnimation;
    FDisplayTime: Integer;
    
    procedure ShowAnimation(AValue: Double; AProgress: Double);
    procedure HideAnimation(AValue: Double; AProgress: Double);
    procedure TimerTick(Sender: TObject);
    
  public
    constructor Create(AParent: TWinControl);
    destructor Destroy; override;
    
    procedure Show(const AMessage: string; ADisplayTimeMS: Integer = 3000;
      AType: string = 'info');
    procedure Hide;
  end;

implementation

{ TUITransition }

constructor TUITransition.Create(AControl: TControl);
begin
  inherited Create;
  FControl := AControl;
  FOrigRect := AControl.BoundsRect;
end;

destructor TUITransition.Destroy;
begin
  FAnimation.Free;
  inherited Destroy;
end;

procedure TUITransition.AnimUpdate(AValue: Double; AProgress: Double);
begin
  case FTransType of
    ttFadeIn, ttFadeOut:
      ApplyFade(AValue);
    ttSlideLeft, ttSlideRight, ttSlideUp, ttSlideDown:
      ApplySlide(AValue, FTransType);
    ttScale, ttZoom:
      ApplyScale(AValue);
  end;
end;

procedure TUITransition.ApplyFade(AValue: Double);
begin
  // Lazarus uses AlphaBlend for transparency
  if FControl is TForm then
  begin
    var F := TForm(FControl);
    F.AlphaBlend := True;
    F.AlphaBlendValue := Round(AValue * 255);
  end
  else
  begin
    // For regular controls, show/hide with opacity simulation
    FControl.Visible := AValue > 0.1;
  end;
end;

procedure TUITransition.ApplySlide(AValue: Double; ADir: TTransitionType);
var
  NewX, NewY: Integer;
begin
  case ADir of
    ttSlideLeft:
    begin
      NewX := Round(FOrigRect.Left - AValue);
      FControl.Left := NewX;
    end;
    ttSlideRight:
    begin
      NewX := Round(FOrigRect.Left + AValue);
      FControl.Left := NewX;
    end;
    ttSlideUp:
    begin
      NewY := Round(FOrigRect.Top - AValue);
      FControl.Top := NewY;
    end;
    ttSlideDown:
    begin
      NewY := Round(FOrigRect.Top + AValue);
      FControl.Top := NewY;
    end;
  end;
end;

procedure TUITransition.ApplyScale(AValue: Double);
begin
  var CenterX := FOrigRect.Left + FOrigRect.Width div 2;
  var CenterY := FOrigRect.Top + FOrigRect.Height div 2;
  
  var NewW := Round(FOrigRect.Width * AValue);
  var NewH := Round(FOrigRect.Height * AValue);
  
  FControl.SetBounds(
    CenterX - NewW div 2,
    CenterY - NewH div 2,
    NewW, NewH
  );
end;

procedure TUITransition.AnimComplete;
begin
  // Reset to original state
  FControl.BoundsRect := FOrigRect;
  
  if Assigned(FOnComplete) then FOnComplete(Self);
end;

procedure TUITransition.Play(AType: TTransitionType; ADuration: Integer;
  AEasing: TEasingType);
begin
  FTransType := AType;
  FOrigRect := FControl.BoundsRect;
  
  FAnimation.Free;
  
  case AType of
    ttFadeIn:
    begin
      FAnimation := TAnimation.Create(0, 1, ADuration, AEasing);
    end;
    ttFadeOut:
    begin
      FAnimation := TAnimation.Create(1, 0, ADuration, AEasing);
    end;
    ttSlideLeft:
    begin
      FAnimation := TAnimation.Create(0, FControl.Width, ADuration, AEasing);
    end;
    ttSlideRight:
    begin
      FAnimation := TAnimation.Create(0, FControl.Width, ADuration, AEasing);
    end;
    ttSlideUp:
    begin
      FAnimation := TAnimation.Create(0, FControl.Height, ADuration, AEasing);
    end;
    ttSlideDown:
    begin
      FAnimation := TAnimation.Create(0, FControl.Height, ADuration, AEasing);
    end;
    ttScale, ttZoom:
    begin
      FAnimation := TAnimation.Create(0, 1, ADuration, AEasing);
    end;
  end;
  
  FAnimation.OnUpdate := @AnimUpdate;
  FAnimation.OnComplete := @AnimComplete;
  FAnimation.Start;
end;

class procedure TUITransition.Animate(AControl: TControl; AType: TTransitionType;
  ADuration: Integer; AOnComplete: TNotifyEvent);
var
  Trans: TUITransition;
begin
  Trans := TUITransition.Create(AControl);
  Trans.OnComplete := AOnComplete;
  Trans.Play(AType, ADuration);
  // Trans will be freed in AnimComplete
end;

{ TNotificationPopup }

constructor TNotificationPopup.Create(AParent: TWinControl);
begin
  inherited Create;
  FDisplayTime := 3000;
  
  FPanel := TPanel.Create(AParent);
  FPanel.Parent := AParent;
  FPanel.Width := 300;
  FPanel.Height := 60;
  FPanel.Left := AParent.Width - 320;
  FPanel.Top := -70;  // Start off screen
  FPanel.Visible := False;
  FPanel.BevelOuter := bvNone;
  FPanel.Color := $00333333;
  
  FTimer := TTimer.Create(FPanel);
  FTimer.Enabled := False;
  FTimer.OnTimer := @TimerTick;
  
  FAnimation := TAnimation.Create(0, 1, 300, etEaseOut);
  FAnimation.OnUpdate := @ShowAnimation;
end;

destructor TNotificationPopup.Destroy;
begin
  FAnimation.Free;
  inherited Destroy;
end;

procedure TNotificationPopup.ShowAnimation(AValue: Double; AProgress: Double);
var
  ParentH: Integer;
begin
  ParentH := FPanel.Parent.Height;
  FPanel.Top := Round(ParentH - 80 - AValue * 80);
end;

procedure TNotificationPopup.HideAnimation(AValue: Double; AProgress: Double);
var
  ParentH: Integer;
begin
  ParentH := FPanel.Parent.Height;
  FPanel.Top := Round(ParentH - 80 + (1 - AValue) * 80);
  
  if AValue <= 0.01 then
    FPanel.Visible := False;
end;

procedure TNotificationPopup.TimerTick(Sender: TObject);
begin
  FTimer.Enabled := False;
  Hide;
end;

procedure TNotificationPopup.Show(const AMessage: string; ADisplayTimeMS: Integer;
  AType: string);
var
  Lbl: TLabel;
begin
  FDisplayTime := ADisplayTimeMS;
  
  // Set color based on type
  case AType of
    'error':   FPanel.Color := $002020CC;  // Red-ish
    'warning': FPanel.Color := $000099CC;  // Orange-ish
    'success': FPanel.Color := $00207020;  // Green-ish
    else       FPanel.Color := $00444444;  // Dark gray
  end;
  
  // Add/update label
  if FPanel.ControlCount = 0 then
  begin
    Lbl := TLabel.Create(FPanel);
    Lbl.Parent := FPanel;
    Lbl.Font.Color := clWhite;
    Lbl.Font.Size := 10;
    Lbl.Left := 12;
    Lbl.Top := 18;
    Lbl.Width := 276;
    Lbl.AutoSize := False;
  end;
  
  TLabel(FPanel.Controls[0]).Caption := AMessage;
  
  // Show with animation
  FPanel.Visible := True;
  FPanel.Top := FPanel.Parent.Height;  // Start off screen
  FAnimation.OnUpdate := @ShowAnimation;
  FAnimation.Start;
  
  // Auto-hide after display time
  FTimer.Interval := ADisplayTimeMS;
  FTimer.Enabled := True;
end;

procedure TNotificationPopup.Hide;
var
  HideAnim: TAnimation;
begin
  HideAnim := TAnimation.Create(1, 0, 200, etEaseIn);
  HideAnim.OnUpdate := @HideAnimation;
  HideAnim.Start;
end;

end.
```

---

## 79.3 ตัวอย่างโปรแกรม Animation Demo

```pascal
program animation_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls, Graphics,
  animation_engine, ui_transitions;

type
  TDemoForm = class(TForm)
  private
    // Animated circle
    PaintBox: TPaintBox;
    CircleX, CircleY: Double;
    CircleAnim: TAnimation;
    
    // Bounce box
    BouncePanel: TPanel;
    BounceAnim: TAnimation;
    
    // Color animation
    ColorPanel: TPanel;
    ColorAnim: TAnimation;
    
    // Notification
    Notif: TNotificationPopup;
    
    btnStart: TButton;
    btnBounce: TButton;
    btnColor: TButton;
    btnNotify: TButton;
    
    procedure SetupControls;
    procedure PaintBoxPaint(Sender: TObject);
    procedure CircleUpdate(AValue: Double; AProgress: Double);
    procedure BounceUpdate(AValue: Double; AProgress: Double);
    procedure ColorUpdate(AValue: Double; AProgress: Double);
    procedure btnStartClick(Sender: TObject);
    procedure btnBounceClick(Sender: TObject);
    procedure btnColorClick(Sender: TObject);
    procedure btnNotifyClick(Sender: TObject);
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
  end;

constructor TDemoForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  Caption := 'Animation Demo';
  Width := 700;
  Height := 500;
  Position := poScreenCenter;
  
  SetupControls;
end;

destructor TDemoForm.Destroy;
begin
  CircleAnim.Free;
  BounceAnim.Free;
  ColorAnim.Free;
  Notif.Free;
  inherited Destroy;
end;

procedure TDemoForm.SetupControls;
begin
  // Paint box for circle
  PaintBox := TPaintBox.Create(Self);
  PaintBox.Parent := Self;
  PaintBox.Left := 20;
  PaintBox.Top := 20;
  PaintBox.Width := 300;
  PaintBox.Height := 200;
  PaintBox.Color := $00F0F0F0;
  PaintBox.OnPaint := @PaintBoxPaint;
  
  // Bounce panel
  BouncePanel := TPanel.Create(Self);
  BouncePanel.Parent := Self;
  BouncePanel.Left := 50;
  BouncePanel.Top := 280;
  BouncePanel.Width := 60;
  BouncePanel.Height := 60;
  BouncePanel.Color := $00E87722;
  BouncePanel.BevelOuter := bvNone;
  
  // Color panel
  ColorPanel := TPanel.Create(Self);
  ColorPanel.Parent := Self;
  ColorPanel.Left := 350;
  ColorPanel.Top := 20;
  ColorPanel.Width := 200;
  ColorPanel.Height := 100;
  ColorPanel.BevelOuter := bvNone;
  ColorPanel.Color := $00E87722;
  
  // Buttons
  btnStart := TButton.Create(Self);
  btnStart.Parent := Self;
  btnStart.Caption := 'Circle Animation';
  btnStart.Left := 350;
  btnStart.Top := 140;
  btnStart.Width := 150;
  btnStart.OnClick := @btnStartClick;
  
  btnBounce := TButton.Create(Self);
  btnBounce.Parent := Self;
  btnBounce.Caption := 'Bounce!';
  btnBounce.Left := 350;
  btnBounce.Top := 180;
  btnBounce.Width := 150;
  btnBounce.OnClick := @btnBounceClick;
  
  btnColor := TButton.Create(Self);
  btnColor.Parent := Self;
  btnColor.Caption := 'Color Fade';
  btnColor.Left := 350;
  btnColor.Top := 220;
  btnColor.Width := 150;
  btnColor.OnClick := @btnColorClick;
  
  btnNotify := TButton.Create(Self);
  btnNotify.Parent := Self;
  btnNotify.Caption := 'Show Notification';
  btnNotify.Left := 350;
  btnNotify.Top := 260;
  btnNotify.Width := 150;
  btnNotify.OnClick := @btnNotifyClick;
  
  // Initialize
  CircleX := 150;
  CircleY := 100;
  
  // Circle animation (X position)
  CircleAnim := TAnimation.Create(20, 280, 2000, etSineInOut);
  CircleAnim.Loop := True;
  CircleAnim.AutoReverse := True;
  CircleAnim.OnUpdate := @CircleUpdate;
  
  // Bounce animation
  BounceAnim := TAnimation.Create(400, 280, 800, etBounceOut);
  BounceAnim.OnUpdate := @BounceUpdate;
  
  // Color animation
  ColorAnim := TAnimation.Create(0, 1, 1500, etEaseInOut);
  ColorAnim.Loop := True;
  ColorAnim.AutoReverse := True;
  ColorAnim.OnUpdate := @ColorUpdate;
  
  // Notification
  Notif := TNotificationPopup.Create(Self);
end;

procedure TDemoForm.PaintBoxPaint(Sender: TObject);
begin
  PaintBox.Canvas.Brush.Color := $00F0F0F0;
  PaintBox.Canvas.FillRect(PaintBox.ClientRect);
  
  // Draw circle
  PaintBox.Canvas.Brush.Color := $00E87722;
  PaintBox.Canvas.Pen.Color := $00C05010;
  PaintBox.Canvas.Pen.Width := 2;
  PaintBox.Canvas.Ellipse(
    Round(CircleX) - 20,
    Round(CircleY) - 20,
    Round(CircleX) + 20,
    Round(CircleY) + 20
  );
end;

procedure TDemoForm.CircleUpdate(AValue: Double; AProgress: Double);
begin
  CircleX := AValue;
  PaintBox.Invalidate;
end;

procedure TDemoForm.BounceUpdate(AValue: Double; AProgress: Double);
begin
  BouncePanel.Top := Round(AValue);
end;

procedure TDemoForm.ColorUpdate(AValue: Double; AProgress: Double);
var
  R, G, B: Byte;
  StartR, StartG, StartB: Byte;
  EndR, EndG, EndB: Byte;
begin
  // Interpolate from orange to blue
  StartR := $E8; StartG := $77; StartB := $22;  // Orange
  EndR := $22; EndG := $77; EndB := $E8;         // Blue
  
  R := Round(StartR + (EndR - StartR) * AValue);
  G := Round(StartG + (EndG - StartG) * AValue);
  B := Round(StartB + (EndB - StartB) * AValue);
  
  ColorPanel.Color := RGBToColor(R, G, B);
end;

procedure TDemoForm.btnStartClick(Sender: TObject);
begin
  if CircleAnim.State = asRunning then
    CircleAnim.Pause
  else if CircleAnim.State = asPaused then
    CircleAnim.Resume
  else
    CircleAnim.Start;
end;

procedure TDemoForm.btnBounceClick(Sender: TObject);
begin
  BouncePanel.Top := -60;  // Start off screen
  BounceAnim.Start;
end;

procedure TDemoForm.btnColorClick(Sender: TObject);
begin
  if ColorAnim.State = asRunning then
    ColorAnim.Stop
  else
    ColorAnim.Start;
end;

procedure TDemoForm.btnNotifyClick(Sender: TObject);
begin
  Notif.Show('This is a notification message!', 3000, 'success');
end;

var
  App: TApplication;
  Form: TDemoForm;
begin
  Application.Initialize;
  
  Form := TDemoForm.Create(nil);
  try
    Form.ShowModal;
  finally
    Form.Free;
  end;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Animation Engine** - Engine สำหรับ animate values
2. **Easing Functions** - Linear, Ease, Bounce, Elastic, Back
3. **UI Transitions** - Fade, Slide, Scale transitions
4. **Property Animation** - Animate properties แบบ declarative
5. **Sequences & Groups** - รวม animations
6. **Notifications** - Popup notifications ที่มี animation

Animations ที่ดีทำให้ UX ดีขึ้นอย่างมาก ช่วยให้ผู้ใช้เข้าใจว่าเกิดอะไรขึ้นในแอปและรู้สึกว่าแอปมีความลื่นไหล
