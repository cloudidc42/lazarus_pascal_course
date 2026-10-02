# ตอนที่ 98: Open Source Contribution กับ Pascal

## บทนำ: การมีส่วนร่วมใน Open Source

การ contribute ต่อ Open Source ช่วยพัฒนาทักษะและ network ในชุมชน Pascal/Lazarus

## 1. Git Workflow สำหรับ Open Source

```bash
# Fork และ Clone Repository
git clone https://github.com/YOUR_USERNAME/lazarus.git
cd lazarus
git remote add upstream https://github.com/graemeg/lazarus.git

# Sync กับ upstream ก่อนเริ่มงาน
git fetch upstream
git checkout main
git merge upstream/main

# สร้าง feature branch
git checkout -b fix/issue-1234-listbox-selection

# ทำงาน...
git add -p  # Add changes ทีละ hunk (แนะนำ)
git commit -m "Fix: ListBox selection lost after Refresh

When Refresh was called on TListBox, the selected item
index was reset to -1 even if the same items were present.

This fix saves and restores the selection state around
the Refresh operation.

Fixes #1234"

# Push to fork
git push origin fix/issue-1234-listbox-selection

# Create Pull Request บน GitHub
```

## 2. Code Review Checklist

```pascal
// ตัวอย่าง Code ที่ดีสำหรับ Open Source Contribution

// BAD: ไม่มี comments, ชื่อตัวแปรไม่ชัดเจน
function f(x: Integer): Boolean;
var a, b: Integer;
begin
  a := x * 2;
  b := a mod 3;
  Result := b = 0;
end;

// GOOD: มี documentation, ชื่อชัดเจน, มี test
// IsEvenMultipleOfThree - Returns True if AValue is divisible by 6.
// AValue: Integer value to check
// Returns: True if AValue mod 6 = 0
function IsEvenMultipleOfThree(AValue: Integer): Boolean;
var
  Doubled: Integer;
begin
  // x is divisible by 6 iff 2x is divisible by 3
  Doubled := AValue * 2;
  Result := Doubled mod 3 = 0;
end;
```

## 3. Writing Unit Tests

```pascal
// tests/TestListBoxFix.pas - Unit Tests
unit TestListBoxFix;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpcunit, testregistry,
  StdCtrls, Forms;

type
  TListBoxSelectionTest = class(TTestCase)
  private
    FForm: TForm;
    FListBox: TListBox;

  protected
    procedure SetUp; override;
    procedure TearDown; override;

  published
    // Tests that verify the fix
    procedure TestSelectionPreservedAfterRefresh;
    procedure TestMultipleSelectionPreservedAfterRefresh;
    procedure TestEmptyListAfterRefresh;
    procedure TestSelectionWithFilteredItems;
  end;

implementation

procedure TListBoxSelectionTest.SetUp;
begin
  FForm := TForm.Create(nil);
  FListBox := TListBox.Create(FForm);
  FListBox.Parent := FForm;
  FListBox.Items.Add('Apple');
  FListBox.Items.Add('Banana');
  FListBox.Items.Add('Cherry');
end;

procedure TListBoxSelectionTest.TearDown;
begin
  FForm.Free;
end;

procedure TListBoxSelectionTest.TestSelectionPreservedAfterRefresh;
begin
  // Arrange
  FListBox.ItemIndex := 1; // Select "Banana"
  AssertEquals('Setup: Initial selection', 1, FListBox.ItemIndex);

  // Act
  FListBox.Refresh;

  // Assert
  AssertEquals('Selection preserved after Refresh',
    1, FListBox.ItemIndex);
  AssertEquals('Selected item text correct',
    'Banana', FListBox.Items[FListBox.ItemIndex]);
end;

procedure TListBoxSelectionTest.TestMultipleSelectionPreservedAfterRefresh;
var
  I: Integer;
begin
  FListBox.MultiSelect := True;

  // Select items 0 and 2
  FListBox.Selected[0] := True;
  FListBox.Selected[2] := True;

  FListBox.Refresh;

  AssertTrue('Item 0 still selected', FListBox.Selected[0]);
  AssertFalse('Item 1 not selected', FListBox.Selected[1]);
  AssertTrue('Item 2 still selected', FListBox.Selected[2]);
end;

procedure TListBoxSelectionTest.TestEmptyListAfterRefresh;
begin
  FListBox.ItemIndex := -1; // No selection
  FListBox.Refresh;
  AssertEquals('Empty selection preserved', -1, FListBox.ItemIndex);
end;

initialization
  RegisterTest(TListBoxSelectionTest);

end.
```

## 4. OPM (Online Package Manager) Component

```pascal
// TStarRating/uStarRating.pas - Custom Component สำหรับ OPM
unit uStarRating;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, Graphics, Forms,
  LCLIntf, LCLType;

type
  TStarRatingChangeEvent = procedure(ASender: TObject;
    ANewRating: Integer) of object;

  { TStarRating - A star rating component for Lazarus }
  TStarRating = class(TCustomControl)
  private
    FMaxStars: Integer;
    FCurrentRating: Integer;
    FHoverRating: Integer;
    FStarSize: Integer;
    FStarSpacing: Integer;
    FFilledColor: TColor;
    FEmptyColor: TColor;
    FHoverColor: TColor;
    FReadOnly: Boolean;
    FOnChange: TStarRatingChangeEvent;

    procedure SetMaxStars(AValue: Integer);
    procedure SetCurrentRating(AValue: Integer);
    procedure SetStarSize(AValue: Integer);
    procedure SetFilledColor(AValue: TColor);
    procedure DrawStar(ACanvas: TCanvas; AX, AY, ASize: Integer;
      AFilled: Boolean; AHover: Boolean);
    function GetStarAtPos(AX: Integer): Integer;
    procedure UpdateSize;

  protected
    procedure Paint; override;
    procedure MouseMove(Shift: TShiftState; X, Y: Integer); override;
    procedure MouseDown(Button: TMouseButton; Shift: TShiftState;
      X, Y: Integer); override;
    procedure MouseLeave; override;
    procedure KeyDown(var Key: Word; Shift: TShiftState); override;

  public
    constructor Create(AOwner: TComponent); override;

    procedure ResetRating;

    property CurrentRating: Integer read FCurrentRating
      write SetCurrentRating;

  published
    // Published for Object Inspector
    property MaxStars: Integer read FMaxStars write SetMaxStars
      default 5;
    property StarSize: Integer read FStarSize write SetStarSize
      default 24;
    property StarSpacing: Integer read FStarSpacing
      write FStarSpacing default 4;
    property FilledColor: TColor read FFilledColor
      write SetFilledColor default clYellow;
    property EmptyColor: TColor read FEmptyColor
      write FEmptyColor default clGray;
    property HoverColor: TColor read FHoverColor
      write FHoverColor default clGold;
    property ReadOnly: Boolean read FReadOnly write FReadOnly
      default False;
    property OnChange: TStarRatingChangeEvent read FOnChange
      write FOnChange;

    // Inherited
    property Align;
    property Anchors;
    property Color;
    property Cursor;
    property Enabled;
    property Font;
    property Hint;
    property ShowHint;
    property TabOrder;
    property TabStop;
    property Visible;
    property OnClick;
    property OnDblClick;
    property OnEnter;
    property OnExit;
    property OnMouseMove;
  end;

procedure Register;

implementation

constructor TStarRating.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FMaxStars := 5;
  FCurrentRating := 0;
  FHoverRating := 0;
  FStarSize := 24;
  FStarSpacing := 4;
  FFilledColor := clYellow;
  FEmptyColor := clGray;
  FHoverColor := clGold;
  FReadOnly := False;

  TabStop := True;
  UpdateSize;
end;

procedure TStarRating.UpdateSize;
begin
  Width := FMaxStars * (FStarSize + FStarSpacing) - FStarSpacing;
  Height := FStarSize;
end;

procedure TStarRating.DrawStar(ACanvas: TCanvas; AX, AY, ASize: Integer;
  AFilled: Boolean; AHover: Boolean);
var
  Points: array[0..4] of TPoint;
  CX, CY, Outer, Inner: Integer;
  Angle: Double;
  I: Integer;
begin
  CX := AX + ASize div 2;
  CY := AY + ASize div 2;
  Outer := ASize div 2;
  Inner := Outer * 4 div 10; // Inner radius = 40% of outer

  // Calculate 5-pointed star vertices
  for I := 0 to 4 do
  begin
    Angle := (I * 72 - 90) * Pi / 180; // Start from top
    Points[I].X := CX + Round(Outer * Cos(Angle));
    Points[I].Y := CY + Round(Outer * Sin(Angle));
  end;

  if AFilled then
  begin
    if AHover then
      ACanvas.Brush.Color := FHoverColor
    else
      ACanvas.Brush.Color := FFilledColor;
  end
  else
    ACanvas.Brush.Color := FEmptyColor;

  ACanvas.Pen.Color := ACanvas.Brush.Color;

  // Draw using polygon
  ACanvas.Polygon(Points);
end;

procedure TStarRating.Paint;
var
  I: Integer;
  X: Integer;
  DisplayRating: Integer;
begin
  Canvas.Brush.Color := Color;
  Canvas.FillRect(ClientRect);

  if FHoverRating > 0 then
    DisplayRating := FHoverRating
  else
    DisplayRating := FCurrentRating;

  for I := 1 to FMaxStars do
  begin
    X := (I - 1) * (FStarSize + FStarSpacing);
    DrawStar(Canvas, X, 0, FStarSize,
      I <= DisplayRating,       // Is filled?
      (FHoverRating > 0) and (I <= FHoverRating)); // Is hover?
  end;
end;

procedure TStarRating.MouseMove(Shift: TShiftState; X, Y: Integer);
var
  StarIdx: Integer;
begin
  inherited MouseMove(Shift, X, Y);

  if FReadOnly then Exit;

  StarIdx := GetStarAtPos(X);
  if StarIdx <> FHoverRating then
  begin
    FHoverRating := StarIdx;
    Invalidate;
  end;
end;

procedure TStarRating.MouseDown(Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
var
  StarIdx: Integer;
begin
  inherited MouseDown(Button, Shift, X, Y);

  if FReadOnly or (Button <> mbLeft) then Exit;

  StarIdx := GetStarAtPos(X);

  // Click same star = deselect (toggle off)
  if StarIdx = FCurrentRating then
    StarIdx := 0;

  SetCurrentRating(StarIdx);
end;

procedure TStarRating.MouseLeave;
begin
  inherited MouseLeave;
  if FHoverRating <> 0 then
  begin
    FHoverRating := 0;
    Invalidate;
  end;
end;

function TStarRating.GetStarAtPos(AX: Integer): Integer;
var
  StarWidth: Integer;
begin
  StarWidth := FStarSize + FStarSpacing;
  Result := (AX div StarWidth) + 1;
  if Result > FMaxStars then Result := FMaxStars;
  if Result < 1 then Result := 0;
end;

procedure TStarRating.SetCurrentRating(AValue: Integer);
begin
  if AValue < 0 then AValue := 0;
  if AValue > FMaxStars then AValue := FMaxStars;

  if FCurrentRating <> AValue then
  begin
    FCurrentRating := AValue;
    Invalidate;

    if Assigned(FOnChange) then
      FOnChange(Self, FCurrentRating);
  end;
end;

procedure Register;
begin
  RegisterComponents('Extra Controls',
    [TStarRating]);
end;

end.
```

## 5. Package Registration สำหรับ Lazarus OPM

```xml
<!-- TStarRating.lpk - Lazarus Package File -->
<?xml version="1.0" encoding="UTF-8"?>
<CONFIG>
  <Package Version="4">
    <Name Value="TStarRatingPkg"/>
    <Type Value="RunAndDesignTime"/>
    <Version Major="1" Minor="0" Release="0"/>
    <Author Value="Your Name"/>
    <CompilerOptions>
      <Version Value="11"/>
      <SearchPaths>
        <SrcFiles Value="src/"/>
      </SearchPaths>
    </CompilerOptions>
    <Description Value="Star Rating component for Lazarus"/>
    <License Value="MIT"/>
    <Files Count="2">
      <Item1>
        <Filename Value="src/uStarRating.pas"/>
        <HasRegisterProc Value="True"/>
        <UnitName Value="uStarRating"/>
      </Item1>
      <Item2>
        <Filename Value="TStarRatingPkg.pas"/>
        <Type Value="Main Unit"/>
      </Item2>
    </Files>
    <RequiredPkgs Count="2">
      <Item1>
        <PackageName Value="LCL"/>
      </Item1>
      <Item2>
        <PackageName Value="FCL"/>
      </Item2>
    </RequiredPkgs>
  </Package>
</CONFIG>
```

## 6. Contribution Guidelines

```markdown
# Contributing to TStarRating

## Getting Started

1. Fork the repository
2. Clone your fork: `git clone https://github.com/yourname/TStarRating.git`
3. Install Lazarus IDE
4. Open `TStarRatingPkg.lpk` in Lazarus

## Development Process

1. Check existing issues and PRs
2. Create an issue for discussion (for major changes)
3. Create a feature branch: `git checkout -b feature/my-feature`
4. Write tests first (TDD recommended)
5. Implement the feature
6. Run all tests: `./runtests.sh`
7. Update documentation
8. Create Pull Request

## Code Style

- Use `{$mode objfpc}{$H+}` at top of every unit
- Prefix fields with `F`, parameters with `A`
- Use descriptive names in English
- Document public APIs with comments
- Follow existing code style

## Pull Request Template

**Description:**
Brief description of what this PR does.

**Related Issue:** #XXX

**Type of Change:**
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

**Testing:**
How did you test this?

**Checklist:**
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Code follows style guidelines
- [ ] PR title is descriptive
```

## 7. สรุป Open Source Contribution

**ขั้นตอนการ Contribute:**
1. **Find a project** - Lazarus, FPC, fpweb, castle-engine
2. **Start small** - Bug reports, documentation, small fixes
3. **Learn the codebase** - Read existing code style
4. **Write tests** - พิสูจน์ bug และ verify fix
5. **Submit PR** - Clear description, follow guidelines

**Resources:**
- **Lazarus Forum**: forum.lazarus.freepascal.org
- **FPC Bugtracker**: bugs.freepascal.org
- **Lazarus GitHub**: github.com/graemeg/lazarus
- **OPM Registry**: packages.lazarus-ide.org

**Pascal Open Source Projects:**
- **Lazarus IDE** - IDE itself
- **castle-engine** - 3D/2D game engine
- **mORMot2** - ORM, REST, Crypto
- **fpweb** - Web framework
- **ZeosLib** - Database access
- **fpjson** - JSON library (built-in)
- **synapse** - Network library
