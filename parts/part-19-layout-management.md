# Part 19 - Layout Management

## บทนำ

Layout Management เป็นกระบวนการจัดวาง Controls บน Form อย่างเป็นระเบียบ และทำให้ Form สามารถปรับตัวได้เมื่อขนาดเปลี่ยนหรือใช้งานบน Screen Resolution ต่างๆ ใน Lazarus มีเครื่องมือหลายอย่างที่ช่วยในเรื่องนี้

---

## 19.1 Anchors

### 19.1.1 พื้นฐาน Anchors

```pascal
// Anchors กำหนดว่า Control ยึดติดกับขอบ Form ด้านใดบ้าง
// เมื่อ Form ถูก Resize Control จะปรับตัวตาม

// Anchor Types:
// akLeft   - ยึดซ้าย (ระยะห่างจากขอบซ้ายคงที่)
// akRight  - ยึดขวา (ระยะห่างจากขอบขวาคงที่)
// akTop    - ยึดบน (ระยะห่างจากขอบบนคงที่)
// akBottom - ยึดล่าง (ระยะห่างจากขอบล่างคงที่)

procedure TForm1.SetupAnchors;
begin
  // ปุ่มที่ตำแหน่งคงที่ (ยึดบนซ้าย - ค่าเริ่มต้น)
  ButtonTopLeft.Anchors := [akTop, akLeft];
  
  // ปุ่มที่ยึดมุมล่างขวา (จะเลื่อนตามเมื่อ Resize)
  ButtonBottomRight.Anchors := [akBottom, akRight];
  
  // Edit ที่ยืดตามความกว้าง
  EditName.Anchors := [akTop, akLeft, akRight];
  // ความกว้างจะเพิ่มขึ้นเมื่อ Form กว้างขึ้น
  
  // Memo ที่ยืดตามทั้งสองทิศทาง
  MemoContent.Anchors := [akTop, akLeft, akRight, akBottom];
  // ทั้งกว้างและสูงจะปรับตาม Form
  
  // ปุ่มที่อยู่กลาง-ล่าง
  ButtonCenter.Anchors := [akBottom];
  // จะเลื่อนลงตาม Form แต่ตำแหน่ง X คงที่ (สัมพัทธ์กับขอบซ้าย)
end;
```

### 19.1.2 ตัวอย่างการใช้ Anchors ใน Form

```pascal
// ตัวอย่าง Layout ที่ใช้ Anchors
// +------------------------------------------+
// | [Label][    Edit                        ] | <- Edit ยืดตามกว้าง
// | [Toolbar Panel                          ] | <- Panel เต็มกว้าง
// | [List  |    Content Area                ] | <- ทั้งคู่ยืดตามสูง
// |        |                                  |
// | [OK] [Cancel]                             | <- ปุ่มชิดขวา-ล่าง
// +------------------------------------------+

procedure TLayoutForm.SetupLayout;
begin
  // Search Label
  LabelSearch.Anchors := [akTop, akLeft];
  LabelSearch.Left := 10;
  LabelSearch.Top := 10;
  
  // Search Edit - ยืดตามความกว้าง
  EditSearch.Anchors := [akTop, akLeft, akRight];
  EditSearch.Left := 80;
  EditSearch.Top := 8;
  EditSearch.Width := ClientWidth - 90; // ความกว้างเริ่มต้น
  
  // Toolbar Panel
  PanelToolbar.Anchors := [akTop, akLeft, akRight];
  PanelToolbar.Align := alTop;
  PanelToolbar.Height := 35;
  
  // Left Panel (List)
  PanelList.Anchors := [akTop, akLeft, akBottom];
  PanelList.Align := alLeft;
  PanelList.Width := 200;
  
  // Main Content Area
  MemoContent.Anchors := [akTop, akLeft, akRight, akBottom];
  MemoContent.Align := alClient;
  
  // OK Button - ชิดขวาล่าง
  ButtonOK.Anchors := [akBottom, akRight];
  ButtonOK.Left := ClientWidth - ButtonOK.Width - 10;
  ButtonOK.Top := ClientHeight - ButtonOK.Height - 10;
  
  // Cancel Button - ชิดขวาล่าง ซ้าย OK
  ButtonCancel.Anchors := [akBottom, akRight];
  ButtonCancel.Left := ButtonOK.Left - ButtonCancel.Width - 5;
  ButtonCancel.Top := ButtonOK.Top;
end;
```

---

## 19.2 Constraints

### 19.2.1 การใช้ Constraints

```pascal
// Constraints กำหนดขนาดสูงสุดและต่ำสุด
// ทั้งสำหรับ Form และ Controls

procedure TConstraintsForm.SetupConstraints;
begin
  // Form Constraints
  Constraints.MinWidth := 400;
  Constraints.MinHeight := 300;
  Constraints.MaxWidth := 1200;
  Constraints.MaxHeight := 900;
  
  // Control Constraints
  PanelSidebar.Constraints.MinWidth := 100;
  PanelSidebar.Constraints.MaxWidth := 400;
  
  // Splitter ใช้ร่วมกับ Constraints เพื่อจำกัดการ Resize
  
  // ตัวอย่าง: Form ที่ Resize ได้แต่มีขนาดต่ำสุด
  OnResize := FormResize;
end;

procedure TConstraintsForm.FormResize(Sender: TObject);
begin
  // บางครั้งต้องปรับ Layout เมื่อ Resize
  if ClientWidth < 600 then
  begin
    // ซ่อน Panel ซ้าย
    PanelSidebar.Visible := False;
    MemoContent.Align := alClient;
  end
  else
  begin
    PanelSidebar.Visible := True;
    PanelSidebar.Align := alLeft;
    MemoContent.Align := alClient;
  end;
end;
```

---

## 19.3 Align Property

### 19.3.1 ค่า Align ต่างๆ

```pascal
// Align กำหนดการจัดวาง Control ใน Parent

// alNone    - ไม่ Align (ตำแหน่ง Manual)
// alTop     - ชิดด้านบน (ความกว้างเต็ม Parent)
// alBottom  - ชิดด้านล่าง (ความกว้างเต็ม Parent)
// alLeft    - ชิดซ้าย (ความสูงเต็ม Parent)
// alRight   - ชิดขวา (ความสูงเต็ม Parent)
// alClient  - เต็ม Parent ที่เหลือ (หลังจาก Control อื่น Align แล้ว)
// alCustom  - กำหนด Position Custom

procedure TAlignForm.SetupAlignLayout;
begin
  // Toolbar (Top)
  PanelToolbar.Align := alTop;
  PanelToolbar.Height := 40;
  
  // Status Bar (Bottom)
  PanelStatus.Align := alBottom;
  PanelStatus.Height := 25;
  
  // Sidebar (Left)
  PanelSidebar.Align := alLeft;
  PanelSidebar.Width := 200;
  
  // Properties Panel (Right)
  PanelProperties.Align := alRight;
  PanelProperties.Width := 250;
  
  // Main Content (Client - ส่วนที่เหลือ)
  PanelContent.Align := alClient;
end;
```

### 19.3.2 ลำดับของ Controls ที่ Align

```pascal
// ลำดับสำคัญมาก! Controls ที่ Align จะกินพื้นที่ตามลำดับ Z-Order
// Controls ที่อยู่ด้านหลัง (Z-Order ต่ำ) จะถูกขนาบก่อน

// ตัวอย่างลำดับที่ถูกต้อง:
procedure TForm1.SetupAlignOrder;
begin
  // 1. Status Bar ก่อน (Bottom) - ต้องอยู่ด้านหน้า Z-Order
  PanelStatus.Align := alBottom;
  PanelStatus.BringToFront; // หรือจัด Z-Order ใน Designer
  
  // 2. Toolbar (Top)
  PanelToolbar.Align := alTop;
  PanelToolbar.BringToFront;
  
  // 3. Sidebar (Left)
  PanelSidebar.Align := alLeft;
  PanelSidebar.BringToFront;
  
  // 4. Content (Client) - อยู่หลังสุด
  MemoContent.Align := alClient;
  MemoContent.SendToBack;
end;
```

---

## 19.4 AutoSize

### 19.4.1 การใช้ AutoSize

```pascal
// AutoSize ทำให้ Control ปรับขนาดตาม Content อัตโนมัติ

procedure TAutoSizeForm.SetupAutoSize;
begin
  // Label - AutoSize ค่าเริ่มต้น True
  Label1.AutoSize := True;
  Label1.Caption := 'ข้อความยาวๆ ที่จะทำให้ Label ขยาย';
  // Label จะขยายกว้างตาม Caption อัตโนมัติ
  
  // Button - AutoSize
  Button1.AutoSize := True;
  Button1.Caption := 'ปุ่มที่ Caption ยาว';
  
  // Panel - AutoSize ปรับตาม Children
  Panel1.AutoSize := True;
  // Panel จะขยายตาม Controls ที่อยู่ข้างใน
  
  // ปิด AutoSize เพื่อกำหนดขนาดเอง
  Label1.AutoSize := False;
  Label1.Width := 300;
  Label1.Height := 60;
  Label1.WordWrap := True; // ต้องปิด AutoSize ก่อน
end;
```

---

## 19.5 TPanel as Container

### 19.5.1 TPanel สำหรับ Group Controls

```pascal
// TPanel ใช้เป็น Container สำหรับกลุ่ม Controls ที่เกี่ยวข้อง
// ทำให้จัดการ Layout ได้ง่ายขึ้น

procedure TPanelForm.SetupPanelLayout;
begin
  // Header Panel
  PanelHeader.Align := alTop;
  PanelHeader.Height := 60;
  PanelHeader.Color := $00996633; // สีน้ำตาล
  PanelHeader.BevelOuter := bvNone;
  
  // Controls ใน Header
  LabelTitle.Parent := PanelHeader;
  LabelTitle.Caption := 'ชื่อโปรแกรม';
  LabelTitle.Font.Size := 18;
  LabelTitle.Font.Color := clWhite;
  LabelTitle.Font.Bold := True;
  LabelTitle.Align := alClient;
  LabelTitle.Alignment := taCenter;
  LabelTitle.Layout := tlCenter; // แนวตั้ง
  
  // Content Panel
  PanelContent.Align := alClient;
  PanelContent.BevelOuter := bvNone;
  
  // Left Nav Panel
  PanelNav.Parent := PanelContent;
  PanelNav.Align := alLeft;
  PanelNav.Width := 180;
  PanelNav.Color := $00F0F0F0;
  PanelNav.BevelOuter := bvNone;
  
  // Footer Panel
  PanelFooter.Align := alBottom;
  PanelFooter.Height := 30;
  PanelFooter.Color := $00F0F0F0;
  PanelFooter.BevelOuter := bvNone;
  
  // Status Label ใน Footer
  LabelStatus.Parent := PanelFooter;
  LabelStatus.Align := alClient;
  LabelStatus.Caption := 'พร้อมใช้งาน';
end;
```

---

## 19.6 TFlowPanel

### 19.6.1 การใช้ TFlowPanel

```pascal
// TFlowPanel จัดเรียง Controls อัตโนมัติจากซ้ายไปขวา
// เมื่อไม่พอ จะขึ้นบรรทัดใหม่

procedure TFlowPanelForm.SetupFlowPanel;
begin
  FlowPanel1.AutoSize := True;     // ปรับขนาดตาม Controls
  FlowPanel1.WrapControls := True; // ขึ้นบรรทัดใหม่เมื่อพื้นที่ไม่พอ
  FlowPanel1.FlowStyle := fsLeftRightTopBottom; // ทิศทางการไหล
  
  // เพิ่มปุ่มลงใน FlowPanel
  AddButtonToFlow('ปุ่ม 1');
  AddButtonToFlow('ปุ่มที่ 2 ยาวกว่า');
  AddButtonToFlow('ปุ่ม 3');
  AddButtonToFlow('ปุ่ม 4');
end;

procedure TFlowPanelForm.AddButtonToFlow(const Caption: string);
var
  btn: TButton;
begin
  btn := TButton.Create(FlowPanel1);
  btn.Parent := FlowPanel1;
  btn.Caption := Caption;
  btn.AutoSize := True;
  btn.Margins.SetBounds(2, 2, 2, 2);
  btn.OnClick := FlowButtonClick;
end;

// ตัวอย่าง FlowPanel สำหรับ Tag Cloud
procedure TFlowPanelForm.CreateTagCloud(Tags: TStringList);
var
  i: Integer;
  btn: TButton;
begin
  FlowPanelTags.BeginUpdate;
  try
    FlowPanelTags.DestroyComponents;
    
    for i := 0 to Tags.Count - 1 do
    begin
      btn := TButton.Create(FlowPanelTags);
      btn.Parent := FlowPanelTags;
      btn.Caption := Tags[i];
      btn.AutoSize := True;
      btn.Flat := True;
      btn.Margins.SetBounds(3, 3, 3, 3);
      btn.Color := clSkyBlue;
      btn.Cursor := crHandPoint;
      btn.OnClick := TagClick;
    end;
  finally
    FlowPanelTags.EndUpdate;
  end;
end;
```

---

## 19.7 TGridPanel

### 19.7.1 การใช้ TGridPanel

```pascal
// TGridPanel จัดเรียง Controls เป็น Grid (ตาราง)
// มีประโยชน์สำหรับ Form-based Layout

procedure TGridPanelForm.SetupGridPanel;
begin
  GridPanel1.ColumnCount := 2;   // 2 คอลัมน์
  GridPanel1.RowCount := 3;      // 3 แถว
  
  // กำหนดขนาด Column
  GridPanel1.ColumnCollection[0].SizeStyle := ssPercent;
  GridPanel1.ColumnCollection[0].Value := 30; // 30%
  GridPanel1.ColumnCollection[1].SizeStyle := ssPercent;
  GridPanel1.ColumnCollection[1].Value := 70; // 70%
  
  // กำหนดขนาด Row
  GridPanel1.RowCollection[0].SizeStyle := ssAuto;
  GridPanel1.RowCollection[1].SizeStyle := ssAuto;
  GridPanel1.RowCollection[2].SizeStyle := ssAuto;
  
  // วาง Controls
  // จะวางอัตโนมัติตามลำดับ Z-Order
end;

// ตัวอย่าง Form Layout ด้วย GridPanel
procedure TRegistrationForm.SetupFormLayout;
var
  lbl: TLabel;
  edt: TEdit;
  fields: array of string;
  i: Integer;
begin
  fields := ['ชื่อ:', 'นามสกุล:', 'อีเมล:', 'โทรศัพท์:', 'ที่อยู่:'];
  
  GridPanel1.ColumnCount := 2;
  GridPanel1.RowCount := Length(fields) + 1; // +1 สำหรับปุ่ม
  
  GridPanel1.ColumnCollection[0].SizeStyle := ssAbsolute;
  GridPanel1.ColumnCollection[0].Value := 120; // Label width
  GridPanel1.ColumnCollection[1].SizeStyle := ssPercent;
  GridPanel1.ColumnCollection[1].Value := 100; // Input takes rest
  
  for i := 0 to High(fields) do
  begin
    // Label
    lbl := TLabel.Create(GridPanel1);
    lbl.Parent := GridPanel1;
    lbl.Caption := fields[i];
    lbl.Alignment := taRightJustify;
    lbl.Layout := tlCenter;
    lbl.Align := alClient;
    
    // Edit
    edt := TEdit.Create(GridPanel1);
    edt.Parent := GridPanel1;
    edt.Align := alClient;
    edt.Margins.SetBounds(2, 2, 2, 2);
  end;
end;
```

---

## 19.8 TScrollBox

### 19.8.1 การใช้ TScrollBox สำหรับ Content ขนาดใหญ่

```pascal
// TScrollBox ให้ Scrolling สำหรับ Content ที่ใหญ่กว่า View
procedure TScrollForm.SetupScrollBox;
begin
  ScrollBox1.AutoScroll := True;
  ScrollBox1.Align := alClient;
  
  // Controls ใน ScrollBox ใช้ขนาดจริง
  // ถ้าใหญ่กว่า ScrollBox จะมี ScrollBar อัตโนมัติ
  
  // กำหนด Scroll Position
  ScrollBox1.HorzScrollBar.Position := 0;
  ScrollBox1.VertScrollBar.Position := 0;
  
  // Scroll ไปตำแหน่งที่ต้องการ
  ScrollBox1.ScrollBy(100, 50); // เลื่อนขวา 100, ลง 50
end;

// ตัวอย่าง Dynamic Form ใน ScrollBox
procedure TDynamicForm.CreateDynamicFields(Count: Integer);
var
  i: Integer;
  lbl: TLabel;
  edt: TEdit;
  yPos: Integer;
begin
  // Clear เก่า
  ScrollBox1.DestroyComponents;
  
  yPos := 10;
  for i := 1 to Count do
  begin
    lbl := TLabel.Create(ScrollBox1);
    lbl.Parent := ScrollBox1;
    lbl.Caption := 'ข้อมูลที่ ' + IntToStr(i) + ':';
    lbl.Left := 10;
    lbl.Top := yPos + 3;
    
    edt := TEdit.Create(ScrollBox1);
    edt.Parent := ScrollBox1;
    edt.Left := 120;
    edt.Top := yPos;
    edt.Width := 250;
    edt.Name := 'EditField' + IntToStr(i);
    
    yPos := yPos + 35;
  end;
end;
```

---

## 19.9 Splitter

### 19.9.1 การใช้ TSplitter

```pascal
// TSplitter ให้ผู้ใช้ปรับขนาด Panel ได้โดยการลาก

procedure TSplitterForm.SetupSplitter;
begin
  // Panel ซ้าย
  PanelLeft.Align := alLeft;
  PanelLeft.Width := 200;
  
  // Splitter ต้องอยู่ถัดจาก Panel ที่จะ Resize
  Splitter1.Align := alLeft;   // ต้อง Align เหมือน Panel ที่อยู่ข้างๆ
  Splitter1.Width := 5;        // ความกว้างของ Splitter
  Splitter1.MinSize := 100;    // ขนาดต่ำสุดของ Panel ซ้าย
  Splitter1.MaxSize := 400;    // ขนาดสูงสุดของ Panel ซ้าย
  Splitter1.ResizeCursor := crHSplit; // Cursor เมื่อ Hover
  
  // Panel ขวา
  PanelRight.Align := alClient;
  
  // Horizontal Splitter (แนวนอน)
  PanelTop.Align := alTop;
  PanelTop.Height := 200;
  
  Splitter2.Align := alTop;
  Splitter2.Height := 5;
  Splitter2.MinSize := 100;
  
  PanelBottom.Align := alClient;
end;

// Event OnMoved - เมื่อ Splitter ถูกลาก
procedure TSplitterForm.Splitter1Moved(Sender: TObject);
begin
  // อัปเดต Status Bar แสดงขนาด
  StatusBar1.Panels[0].Text := 
    'ซ้าย: ' + IntToStr(PanelLeft.Width) + 'px' +
    '  ขวา: ' + IntToStr(PanelRight.Width) + 'px';
end;
```

### 19.9.2 Multi-Pane Layout ด้วย Splitters

```pascal
// สร้าง Layout แบบ 3 Panel (เหมือน Explorer)
procedure TExplorerForm.SetupExplorerLayout;
begin
  // Panel ซ้าย (Tree)
  PanelTree.Align := alLeft;
  PanelTree.Width := 200;
  
  // Splitter 1
  SplitterLeft.Align := alLeft;
  SplitterLeft.Width := 4;
  SplitterLeft.MinSize := 100;
  
  // Panel กลาง (List)
  PanelList.Align := alLeft;
  PanelList.Width := 400;
  
  // Splitter 2
  SplitterRight.Align := alLeft;
  SplitterRight.Width := 4;
  SplitterRight.MinSize := 200;
  
  // Panel ขวา (Detail)
  PanelDetail.Align := alClient;
  
  // Tree ใน PanelTree
  TreeView1.Parent := PanelTree;
  TreeView1.Align := alClient;
  
  // List ใน PanelList
  ListView1.Parent := PanelList;
  ListView1.Align := alClient;
  
  // Rich Edit ใน PanelDetail
  RichEdit1.Parent := PanelDetail;
  RichEdit1.Align := alClient;
end;
```

---

## 19.10 Responsive Design Principles

### 19.10.1 หลักการ Responsive Form

```pascal
// Responsive Form ปรับตัวตามขนาด Window

unit ResponsiveForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls;

type
  TScreenLayout = (slSmall, slMedium, slLarge);

  TResponsiveForm = class(TForm)
    PanelSidebar: TPanel;
    PanelContent: TPanel;
    PanelToolbar: TPanel;
    SplitterMain: TSplitter;
    ButtonToggleSidebar: TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure FormResize(Sender: TObject);
    procedure ButtonToggleSidebarClick(Sender: TObject);
    
  private
    FCurrentLayout: TScreenLayout;
    FSidebarVisible: Boolean;
    FSidebarWidth: Integer;
    
    procedure ApplyLayout(Layout: TScreenLayout);
    function DetermineLayout: TScreenLayout;
    procedure SetupSmallLayout;
    procedure SetupMediumLayout;
    procedure SetupLargeLayout;
    procedure SaveSidebarState;
    procedure RestoreSidebarState;
  end;

implementation

{$R *.lfm}

procedure TResponsiveForm.FormCreate(Sender: TObject);
begin
  Caption := 'Responsive Form';
  Position := poScreenCenter;
  Width := 900;
  Height := 600;
  
  FSidebarVisible := True;
  FSidebarWidth := 220;
  FCurrentLayout := slLarge;
  
  SetupLargeLayout;
end;

function TResponsiveForm.DetermineLayout: TScreenLayout;
begin
  if ClientWidth < 500 then
    Result := slSmall
  else if ClientWidth < 800 then
    Result := slMedium
  else
    Result := slLarge;
end;

procedure TResponsiveForm.FormResize(Sender: TObject);
var
  newLayout: TScreenLayout;
begin
  newLayout := DetermineLayout;
  if newLayout <> FCurrentLayout then
    ApplyLayout(newLayout);
end;

procedure TResponsiveForm.ApplyLayout(Layout: TScreenLayout);
begin
  FCurrentLayout := Layout;
  case Layout of
    slSmall:  SetupSmallLayout;
    slMedium: SetupMediumLayout;
    slLarge:  SetupLargeLayout;
  end;
end;

procedure TResponsiveForm.SetupSmallLayout;
begin
  // Small: ซ่อน Sidebar, ซ่อน Toolbar Text
  PanelSidebar.Visible := False;
  SplitterMain.Visible := False;
  PanelContent.Align := alClient;
  ButtonToggleSidebar.Visible := True;
  ButtonToggleSidebar.Caption := '☰'; // Hamburger Menu
end;

procedure TResponsiveForm.SetupMediumLayout;
begin
  // Medium: Sidebar แบบ Overlay
  PanelSidebar.Visible := FSidebarVisible;
  SplitterMain.Visible := FSidebarVisible;
  PanelSidebar.Width := Min(200, ClientWidth div 3);
  ButtonToggleSidebar.Visible := True;
end;

procedure TResponsiveForm.SetupLargeLayout;
begin
  // Large: แสดงทุกอย่างเต็มที่
  PanelSidebar.Visible := True;
  SplitterMain.Visible := True;
  PanelSidebar.Width := FSidebarWidth;
  ButtonToggleSidebar.Visible := False;
end;

procedure TResponsiveForm.ButtonToggleSidebarClick(Sender: TObject);
begin
  FSidebarVisible := not FSidebarVisible;
  PanelSidebar.Visible := FSidebarVisible;
  SplitterMain.Visible := FSidebarVisible;
end;
```

---

## 19.11 DPI Awareness และ High DPI Support

### 19.11.1 DPI ใน Lazarus

```pascal
// DPI (Dots Per Inch) ส่งผลต่อขนาดของ Controls
// High DPI (HiDPI) = Screen ที่มี Pixel Density สูง (เช่น 4K Monitor)

unit DPIAwareForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, LazDeviceApis;

type
  TDPIAwareForm = class(TForm)
    procedure FormCreate(Sender: TObject);
    
  private
    FBaseDPI: Integer;
    
    function GetCurrentDPI: Integer;
    function ScaleByDPI(Value: Integer): Integer;
    procedure ScaleAllControls;
    procedure ScaleFont(AFont: TFont; AScale: Double);
  end;

implementation

const
  BASE_DPI = 96; // Standard DPI

procedure TDPIAwareForm.FormCreate(Sender: TObject);
begin
  FBaseDPI := BASE_DPI;
  
  // ตรวจสอบ DPI ปัจจุบัน
  if GetCurrentDPI > FBaseDPI then
    ScaleAllControls;
end;

function TDPIAwareForm.GetCurrentDPI: Integer;
begin
  // ใน Lazarus ดึงค่า DPI จาก Screen
  Result := Screen.PixelsPerInch;
end;

function TDPIAwareForm.ScaleByDPI(Value: Integer): Integer;
begin
  Result := MulDiv(Value, GetCurrentDPI, FBaseDPI);
end;

procedure TDPIAwareForm.ScaleAllControls;
var
  scale: Double;
  i: Integer;
begin
  scale := GetCurrentDPI / FBaseDPI;
  
  // Scale Font ของ Form
  ScaleFont(Font, scale);
  
  // Scale ขนาด Controls
  for i := 0 to ControlCount - 1 do
  begin
    Controls[i].Left := Round(Controls[i].Left * scale);
    Controls[i].Top := Round(Controls[i].Top * scale);
    
    if Controls[i].Anchors = [akLeft, akTop] then
    begin
      // Controls ที่ไม่ได้ Anchor จะต้อง Scale ขนาดด้วย
      Controls[i].Width := Round(Controls[i].Width * scale);
      Controls[i].Height := Round(Controls[i].Height * scale);
    end;
  end;
end;

procedure TDPIAwareForm.ScaleFont(AFont: TFont; AScale: Double);
begin
  AFont.Size := Round(AFont.Size * AScale);
end;
```

### 19.11.2 การจัดการ Screen Resolution

```pascal
// การตรวจสอบและปรับตาม Screen Resolution
procedure TMainForm.HandleScreenResolution;
var
  screenWidth, screenHeight: Integer;
begin
  screenWidth := Screen.Width;
  screenHeight := Screen.Height;
  
  // ปรับขนาด Form ตาม Resolution
  if screenWidth >= 1920 then
  begin
    // Full HD และสูงกว่า
    Width := 1200;
    Height := 800;
  end
  else if screenWidth >= 1366 then
  begin
    // HD+
    Width := 1000;
    Height := 700;
  end
  else
  begin
    // ความละเอียดต่ำ
    Width := Min(800, screenWidth - 50);
    Height := Min(600, screenHeight - 80);
  end;
  
  Position := poScreenCenter;
end;

// ตรวจสอบ Multi-Monitor
procedure TMainForm.SetupForCurrentMonitor;
var
  monitor: TMonitor;
  workArea: TRect;
begin
  monitor := Screen.MonitorFromWindow(Handle);
  workArea := monitor.WorkareaRect;
  
  // จำกัดขนาด Form ไม่ให้เกิน Work Area
  if Width > workArea.Width - 50 then
    Width := workArea.Width - 50;
  if Height > workArea.Height - 80 then
    Height := workArea.Height - 80;
end;
```

---

## 19.12 โปรแกรมตัวอย่าง: Responsive Form

```pascal
unit ResponsiveFormApp;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls, Menus;

type
  TDashboardForm = class(TForm)
    // Layout Panels
    PanelHeader: TPanel;
    PanelNav: TPanel;
    PanelMain: TPanel;
    PanelFooter: TPanel;
    SplitterNav: TSplitter;
    
    // Header Controls
    LabelAppTitle: TLabel;
    PanelSearch: TPanel;
    EditSearch: TEdit;
    ButtonSearch: TButton;
    ButtonMenu: TButton;
    
    // Navigation
    TreeViewNav: TTreeView;
    
    // Main Content
    PageControlMain: TPageControl;
    TabDashboard: TTabSheet;
    TabDocuments: TTabSheet;
    TabSettings: TTabSheet;
    
    // Footer
    LabelStatus: TLabel;
    LabelDate: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure FormResize(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure ButtonMenuClick(Sender: TObject);
    procedure TreeViewNavClick(Sender: TObject);
    procedure EditSearchChange(Sender: TObject);
    procedure TimerTick(Sender: TObject);
    
  private
    FTimer: TTimer;
    FNavVisible: Boolean;
    FSavedNavWidth: Integer;
    FCurrentSection: string;
    
    procedure SetupLayout;
    procedure SetupHeader;
    procedure SetupNavigation;
    procedure SetupMainContent;
    procedure SetupFooter;
    procedure SetupNavTree;
    procedure ToggleNavigation;
    procedure NavigateTo(const Section: string);
    procedure UpdateStatus(const Msg: string);
    procedure ApplyResponsiveLayout;
    procedure StyleHeader;
    procedure StyleNav;
    
  public
    procedure SelectSection(const Name: string);
  end;

var
  DashboardForm: TDashboardForm;

implementation

{$R *.lfm}

procedure TDashboardForm.FormCreate(Sender: TObject);
begin
  Caption := 'Dashboard Application';
  Width := 1000;
  Height := 650;
  Position := poScreenCenter;
  
  FNavVisible := True;
  FSavedNavWidth := 220;
  FCurrentSection := 'Dashboard';
  
  SetupLayout;
  SetupHeader;
  SetupNavigation;
  SetupMainContent;
  SetupFooter;
  
  // Timer สำหรับอัปเดต Clock
  FTimer := TTimer.Create(Self);
  FTimer.Interval := 1000;
  FTimer.OnTimer := TimerTick;
  FTimer.Enabled := True;
  
  UpdateStatus('ระบบพร้อมใช้งาน');
end;

procedure TDashboardForm.SetupLayout;
begin
  // Header ด้านบน
  PanelHeader.Align := alTop;
  PanelHeader.Height := 50;
  PanelHeader.BevelOuter := bvNone;
  
  // Footer ด้านล่าง
  PanelFooter.Align := alBottom;
  PanelFooter.Height := 25;
  PanelFooter.BevelOuter := bvNone;
  PanelFooter.BevelInner := bvLowered;
  
  // Navigation ด้านซ้าย
  PanelNav.Align := alLeft;
  PanelNav.Width := FSavedNavWidth;
  PanelNav.BevelOuter := bvNone;
  
  // Splitter
  SplitterNav.Align := alLeft;
  SplitterNav.Width := 4;
  SplitterNav.MinSize := 120;
  SplitterNav.MaxSize := 400;
  
  // Main Content
  PanelMain.Align := alClient;
  PanelMain.BevelOuter := bvNone;
end;

procedure TDashboardForm.StyleHeader;
begin
  PanelHeader.Color := $00333333; // Dark Background
  
  LabelAppTitle.Parent := PanelHeader;
  LabelAppTitle.Caption := 'Dashboard';
  LabelAppTitle.Font.Size := 16;
  LabelAppTitle.Font.Bold := True;
  LabelAppTitle.Font.Color := clWhite;
  LabelAppTitle.Left := 50;
  LabelAppTitle.Top := 12;
  LabelAppTitle.AutoSize := True;
  
  ButtonMenu.Parent := PanelHeader;
  ButtonMenu.Caption := '☰';
  ButtonMenu.Left := 5;
  ButtonMenu.Top := 10;
  ButtonMenu.Width := 35;
  ButtonMenu.Height := 30;
  ButtonMenu.Flat := True;
  ButtonMenu.Font.Color := clWhite;
  ButtonMenu.Font.Size := 14;
  ButtonMenu.OnClick := ButtonMenuClick;
end;

procedure TDashboardForm.StyleNav;
begin
  PanelNav.Color := $00444444;
  TreeViewNav.Parent := PanelNav;
  TreeViewNav.Align := alClient;
  TreeViewNav.Color := $00444444;
  TreeViewNav.Font.Color := clWhite;
  TreeViewNav.Font.Size := 11;
  TreeViewNav.BorderStyle := bsNone;
  TreeViewNav.ShowLines := False;
  TreeViewNav.ShowRoot := False;
  TreeViewNav.ShowButtons := False;
end;

procedure TDashboardForm.SetupHeader;
begin
  StyleHeader;
  
  // Search Area
  PanelSearch.Parent := PanelHeader;
  PanelSearch.Align := alRight;
  PanelSearch.Width := 300;
  PanelSearch.Color := PanelHeader.Color;
  PanelSearch.BevelOuter := bvNone;
  
  EditSearch.Parent := PanelSearch;
  EditSearch.Left := 10;
  EditSearch.Top := 13;
  EditSearch.Width := 200;
  EditSearch.Height := 25;
  
  ButtonSearch.Parent := PanelSearch;
  ButtonSearch.Left := 215;
  ButtonSearch.Top := 13;
  ButtonSearch.Width := 60;
  ButtonSearch.Height := 25;
  ButtonSearch.Caption := 'ค้นหา';
  ButtonSearch.Font.Size := 9;
end;

procedure TDashboardForm.SetupNavigation;
begin
  StyleNav;
  SetupNavTree;
end;

procedure TDashboardForm.SetupNavTree;
var
  root: TTreeNode;
begin
  TreeViewNav.Items.Clear;
  
  // เพิ่มรายการ Nav
  TreeViewNav.Items.Add(nil, '📊 Dashboard');
  TreeViewNav.Items.Add(nil, '📁 เอกสาร');
  TreeViewNav.Items.Add(nil, '👥 ผู้ใช้งาน');
  TreeViewNav.Items.Add(nil, '📈 รายงาน');
  TreeViewNav.Items.Add(nil, '⚙️ ตั้งค่า');
  
  // Select รายการแรก
  if TreeViewNav.Items.Count > 0 then
    TreeViewNav.Items[0].Selected := True;
  
  TreeViewNav.OnClick := TreeViewNavClick;
end;

procedure TDashboardForm.SetupMainContent;
begin
  PageControlMain.Parent := PanelMain;
  PageControlMain.Align := alClient;
  PageControlMain.ShowTabs := False; // ซ่อน Tabs (Nav จัดการแทน)
  
  // Dashboard Tab
  TabDashboard.Caption := 'Dashboard';
  CreateDashboardContent;
  
  // Documents Tab
  TabDocuments.Caption := 'เอกสาร';
  CreateDocumentsContent;
  
  // Settings Tab
  TabSettings.Caption := 'ตั้งค่า';
  CreateSettingsContent;
end;

procedure TDashboardForm.CreateDashboardContent;
var
  flowPanel: TFlowPanel;
  card: TPanel;
  i: Integer;
  titles: array[0..3] of string;
  values: array[0..3] of string;
begin
  titles := ['ผู้ใช้ทั้งหมด', 'เอกสาร', 'งานที่ทำ', 'ยังค้างอยู่'];
  values := ['1,234', '567', '890', '45'];
  
  flowPanel := TFlowPanel.Create(TabDashboard);
  flowPanel.Parent := TabDashboard;
  flowPanel.Align := alTop;
  flowPanel.AutoSize := True;
  flowPanel.WrapControls := True;
  
  for i := 0 to 3 do
  begin
    card := TPanel.Create(flowPanel);
    card.Parent := flowPanel;
    card.Width := 180;
    card.Height := 100;
    card.Color := clWhite;
    card.BevelOuter := bvRaised;
    card.Margins.SetBounds(10, 10, 10, 10);
    
    with TLabel.Create(card) do
    begin
      Parent := card;
      Caption := titles[i];
      Font.Size := 10;
      Font.Color := $00666666;
      Left := 10;
      Top := 15;
      AutoSize := True;
    end;
    
    with TLabel.Create(card) do
    begin
      Parent := card;
      Caption := values[i];
      Font.Size := 24;
      Font.Bold := True;
      Font.Color := $00336699;
      Left := 10;
      Top := 40;
      AutoSize := True;
    end;
  end;
end;

procedure TDashboardForm.CreateDocumentsContent;
var
  listView: TListView;
begin
  listView := TListView.Create(TabDocuments);
  listView.Parent := TabDocuments;
  listView.Align := alClient;
  listView.ViewStyle := vsReport;
  
  with listView.Columns.Add do
  begin
    Caption := 'ชื่อเอกสาร';
    Width := 300;
  end;
  with listView.Columns.Add do
  begin
    Caption := 'วันที่';
    Width := 150;
  end;
  with listView.Columns.Add do
  begin
    Caption := 'ขนาด';
    Width := 100;
  end;
  
  // เพิ่มข้อมูลตัวอย่าง
  with listView.Items.Add do
  begin
    Caption := 'รายงานประจำปี 2024.pdf';
    SubItems.Add('01/01/2024');
    SubItems.Add('2.4 MB');
  end;
  
  with listView.Items.Add do
  begin
    Caption := 'ใบเสร็จรับเงิน.docx';
    SubItems.Add('15/01/2024');
    SubItems.Add('125 KB');
  end;
end;

procedure TDashboardForm.CreateSettingsContent;
var
  gridPanel: TGridPanel;
  lbl: TLabel;
  edt: TEdit;
begin
  gridPanel := TGridPanel.Create(TabSettings);
  gridPanel.Parent := TabSettings;
  gridPanel.Align := alTop;
  gridPanel.Height := 200;
  gridPanel.ColumnCount := 2;
  gridPanel.RowCount := 3;
  gridPanel.ColumnCollection[0].SizeStyle := ssAbsolute;
  gridPanel.ColumnCollection[0].Value := 150;
  gridPanel.ColumnCollection[1].SizeStyle := ssPercent;
  gridPanel.ColumnCollection[1].Value := 100;
  
  // แถวที่ 1
  lbl := TLabel.Create(gridPanel);
  lbl.Parent := gridPanel;
  lbl.Caption := 'ชื่อผู้ใช้:';
  lbl.Align := alClient;
  lbl.Layout := tlCenter;
  
  edt := TEdit.Create(gridPanel);
  edt.Parent := gridPanel;
  edt.Align := alClient;
  edt.Margins.SetBounds(5, 5, 5, 5);
  
  // แถวที่ 2
  lbl := TLabel.Create(gridPanel);
  lbl.Parent := gridPanel;
  lbl.Caption := 'อีเมล:';
  lbl.Align := alClient;
  lbl.Layout := tlCenter;
  
  edt := TEdit.Create(gridPanel);
  edt.Parent := gridPanel;
  edt.Align := alClient;
  edt.Margins.SetBounds(5, 5, 5, 5);
end;

procedure TDashboardForm.SetupFooter;
begin
  PanelFooter.Color := $00F0F0F0;
  
  LabelStatus.Parent := PanelFooter;
  LabelStatus.Align := alLeft;
  LabelStatus.Width := 400;
  LabelStatus.Caption := 'พร้อมใช้งาน';
  LabelStatus.Layout := tlCenter;
  
  LabelDate.Parent := PanelFooter;
  LabelDate.Align := alRight;
  LabelDate.Width := 200;
  LabelDate.Alignment := taRightJustify;
  LabelDate.Layout := tlCenter;
end;

procedure TDashboardForm.ToggleNavigation;
begin
  FNavVisible := not FNavVisible;
  
  if FNavVisible then
  begin
    PanelNav.Width := FSavedNavWidth;
    PanelNav.Visible := True;
    SplitterNav.Visible := True;
  end
  else
  begin
    FSavedNavWidth := PanelNav.Width;
    PanelNav.Visible := False;
    SplitterNav.Visible := False;
  end;
end;

procedure TDashboardForm.NavigateTo(const Section: string);
begin
  FCurrentSection := Section;
  
  if Pos('Dashboard', Section) > 0 then
    PageControlMain.ActivePage := TabDashboard
  else if Pos('เอกสาร', Section) > 0 then
    PageControlMain.ActivePage := TabDocuments
  else if Pos('ตั้งค่า', Section) > 0 then
    PageControlMain.ActivePage := TabSettings;
    
  UpdateStatus('ส่วน: ' + Section);
end;

procedure TDashboardForm.UpdateStatus(const Msg: string);
begin
  LabelStatus.Caption := '  ' + Msg;
end;

procedure TDashboardForm.ApplyResponsiveLayout;
begin
  if ClientWidth < 700 then
  begin
    // Mobile-like Layout
    if FNavVisible then
      ToggleNavigation;
    ButtonMenu.Visible := True;
  end
  else
  begin
    // Normal Layout
    if not FNavVisible then
      ToggleNavigation;
    ButtonMenu.Visible := False;
  end;
end;

procedure TDashboardForm.ButtonMenuClick(Sender: TObject);
begin
  ToggleNavigation;
end;

procedure TDashboardForm.TreeViewNavClick(Sender: TObject);
begin
  if Assigned(TreeViewNav.Selected) then
    NavigateTo(TreeViewNav.Selected.Text);
end;

procedure TDashboardForm.EditSearchChange(Sender: TObject);
begin
  // Real-time Search
  if Length(EditSearch.Text) > 2 then
    UpdateStatus('ค้นหา: ' + EditSearch.Text);
end;

procedure TDashboardForm.TimerTick(Sender: TObject);
begin
  LabelDate.Caption := FormatDateTime('dd/mm/yyyy hh:nn:ss', Now) + '  ';
end;

procedure TDashboardForm.FormResize(Sender: TObject);
begin
  ApplyResponsiveLayout;
end;

procedure TDashboardForm.FormDestroy(Sender: TObject);
begin
  FTimer.Free;
end;

procedure TDashboardForm.SelectSection(const Name: string);
begin
  NavigateTo(Name);
end;

end.
```

---

## 19.13 โปรแกรมตัวอย่าง: Docking Panels

```pascal
unit DockingPanels;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, ExtCtrls, StdCtrls, ComCtrls;

type
  TDockPanelForm = class(TForm)
    MainPanel: TPanel;
    SplitterH: TSplitter;
    SplitterV: TSplitter;
    PanelTop: TPanel;
    PanelLeft: TPanel;
    PanelRight: TPanel;
    PanelBottom: TPanel;
    PanelCenter: TPanel;
    
    procedure FormCreate(Sender: TObject);
    procedure SetupDockPanels;
    
  private
    procedure CreatePanelHeader(APanel: TPanel; const Title: string; 
      AColor: TColor);
    procedure SetupSplitters;
  end;

implementation

{$R *.lfm}

procedure TDockPanelForm.FormCreate(Sender: TObject);
begin
  Caption := 'Docking Panels Demo';
  Width := 1000;
  Height := 700;
  Position := poScreenCenter;
  SetupDockPanels;
end;

procedure TDockPanelForm.CreatePanelHeader(APanel: TPanel; 
  const Title: string; AColor: TColor);
var
  header: TPanel;
  lbl: TLabel;
begin
  header := TPanel.Create(APanel);
  header.Parent := APanel;
  header.Align := alTop;
  header.Height := 25;
  header.Color := AColor;
  header.BevelOuter := bvNone;
  header.Caption := '';
  
  lbl := TLabel.Create(header);
  lbl.Parent := header;
  lbl.Caption := Title;
  lbl.Font.Bold := True;
  lbl.Font.Color := clWhite;
  lbl.Left := 8;
  lbl.Top := 5;
  lbl.AutoSize := True;
end;

procedure TDockPanelForm.SetupDockPanels;
begin
  // Top Panel
  PanelTop.Align := alTop;
  PanelTop.Height := 150;
  PanelTop.Color := $00FFFFFF;
  CreatePanelHeader(PanelTop, 'ข้อมูลส่วนหัว (Top Panel)', $004499CC);
  
  // Bottom Panel
  PanelBottom.Align := alBottom;
  PanelBottom.Height := 120;
  PanelBottom.Color := $00FFFFFF;
  CreatePanelHeader(PanelBottom, 'ข้อมูลส่วนล่าง (Bottom Panel)', $007799AA);
  
  // Horizontal Splitter (ระหว่าง Top และ Bottom)
  SplitterH.Align := alBottom;
  SplitterH.Height := 5;
  SplitterH.Cursor := crVSplit;
  SplitterH.MinSize := 80;
  
  // Left Panel
  PanelLeft.Align := alLeft;
  PanelLeft.Width := 200;
  PanelLeft.Color := $00FFFFFF;
  CreatePanelHeader(PanelLeft, 'แผงซ้าย (Left)', $00558866);
  
  // Right Panel
  PanelRight.Align := alRight;
  PanelRight.Width := 220;
  PanelRight.Color := $00FFFFFF;
  CreatePanelHeader(PanelRight, 'แผงขวา (Right)', $00885566);
  
  // Vertical Splitters
  SplitterV.Align := alLeft;
  SplitterV.Width := 5;
  SplitterV.Cursor := crHSplit;
  SplitterV.MinSize := 100;
  
  // Center Panel
  PanelCenter.Align := alClient;
  PanelCenter.Color := $00FFFFFF;
  CreatePanelHeader(PanelCenter, 'พื้นที่หลัก (Center)', $00996633);
  
  // Add Content to Center
  with TMemo.Create(PanelCenter) do
  begin
    Parent := PanelCenter;
    Align := alClient;
    Text := 'พื้นที่ทำงานหลัก' + #13#10 +
            'ลอง Drag Splitters เพื่อปรับขนาด Panel';
    Font.Size := 12;
    ReadOnly := True;
    Color := $00FAFAFA;
  end;
end;

procedure TDockPanelForm.SetupSplitters;
begin
  // จัดการ Splitters
end;

end.
```

---

## 19.14 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Anchors Test
สร้าง Form ที่มี Controls ดังนี้ และปรับ Anchors ให้ถูกต้อง:
- Title Label ด้านบน (ยึดบนกลาง)
- Search Bar ด้านบน (ยึดบนซ้ายขวา)
- ListBox ตรงกลาง (ยึดทุกด้าน)
- OK/Cancel ด้านล่างขวา (ยึดล่างขวา)

```pascal
procedure TAnchorTestForm.SetupAnchors;
begin
  LabelTitle.Anchors := [akTop];
  LabelTitle.AnchorHorizontalCenterTo(Self);
  
  EditSearch.Anchors := [akTop, akLeft, akRight];
  
  ListBox1.Anchors := [akTop, akLeft, akRight, akBottom];
  
  ButtonOK.Anchors := [akRight, akBottom];
  ButtonCancel.Anchors := [akRight, akBottom];
end;
```

### แบบฝึกหัดที่ 2: Responsive Layout
สร้าง Form ที่ปรับ Layout เมื่อขนาดเปลี่ยน:
- > 900px: แสดง 3 Column
- 600-900px: แสดง 2 Column  
- < 600px: แสดง 1 Column

```pascal
procedure TResponsiveGridForm.FormResize(Sender: TObject);
begin
  if ClientWidth > 900 then
    SetThreeColumnLayout
  else if ClientWidth > 600 then
    SetTwoColumnLayout
  else
    SetOneColumnLayout;
end;
```

### แบบฝึกหัดที่ 3: Splitter Application
สร้างโปรแกรม Text Diff ที่มี 2 Panel แบ่งด้วย Splitter

### แบบฝึกหัดที่ 4: FlowPanel Tag System
สร้าง Tag System ที่เพิ่ม/ลบ Tag ได้ใน FlowPanel

```pascal
procedure TTagForm.AddTag(const TagText: string);
var
  panel: TPanel;
  lbl: TLabel;
  btn: TButton;
begin
  panel := TPanel.Create(FlowPanelTags);
  panel.Parent := FlowPanelTags;
  panel.AutoSize := True;
  panel.Height := 28;
  panel.BevelOuter := bvRaised;
  panel.Color := clSkyBlue;
  panel.Caption := '';
  panel.Margins.SetBounds(3, 3, 3, 3);
  
  lbl := TLabel.Create(panel);
  lbl.Parent := panel;
  lbl.Caption := TagText;
  lbl.Left := 5;
  lbl.Top := 6;
  lbl.AutoSize := True;
  
  btn := TButton.Create(panel);
  btn.Parent := panel;
  btn.Caption := '×';
  btn.Width := 20;
  btn.Height := 20;
  btn.Left := lbl.Left + lbl.Width + 3;
  btn.Top := 4;
  btn.Flat := True;
  btn.Tag := PtrInt(panel);
  btn.OnClick := TagRemoveClick;
end;

procedure TTagForm.TagRemoveClick(Sender: TObject);
begin
  TPanel(TButton(Sender).Tag).Free;
end;
```

### แบบฝึกหัดที่ 5-10: หัวข้อเพิ่มเติม

- แบบฝึกหัดที่ 5: GridPanel สำหรับ Contact Form
- แบบฝึกหัดที่ 6: ScrollBox ที่มี Dynamic Content
- แบบฝึกหัดที่ 7: Dashboard Layout ที่ซับซ้อน
- แบบฝึกหัดที่ 8: Mobile-style Navigation
- แบบฝึกหัดที่ 9: DPI-aware Controls
- แบบฝึกหัดที่ 10: Resizable Panels ที่บันทึก State

---

## สรุปบทที่ 19

ในบทนี้เราได้เรียนรู้:

1. **Anchors** - การยึด Controls กับขอบ Form
2. **Constraints** - การกำหนดขนาดสูงสุด/ต่ำสุด
3. **Align** - การจัดเรียง Controls แบบอัตโนมัติ
4. **AutoSize** - การปรับขนาดตาม Content
5. **TPanel** - Container สำหรับจัดกลุ่ม Controls
6. **TFlowPanel** - การจัดเรียงแบบ Flow
7. **TGridPanel** - การจัดเรียงแบบ Grid
8. **TScrollBox** - การ Scroll Content
9. **TSplitter** - การแบ่งพื้นที่แบบ Resizable
10. **Responsive Design** - การออกแบบ UI ที่ปรับตาม Screen
11. **DPI Awareness** - การรองรับ High DPI Displays

บทถัดไปจะเรียนรู้การสร้าง **โปรเจคพื้นฐาน: Text Editor** ที่รวมทุกสิ่งที่เรียนมา
