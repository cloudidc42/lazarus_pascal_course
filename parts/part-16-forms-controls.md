# Part 16 - Forms และ Controls พื้นฐาน

## บทนำ

ใน Lazarus การสร้าง GUI Application จะเริ่มต้นจาก Form ซึ่งเป็น Container หลักที่ใช้วาง Controls ต่างๆ ในบทนี้เราจะเรียนรู้เกี่ยวกับ Form และ Controls พื้นฐานที่ใช้บ่อยในการพัฒนา Application

---

## 16.1 TForm - พื้นฐาน

### 16.1.1 คุณสมบัติ (Properties) ของ TForm

TForm คือ Window หลักของ Application แต่ละ Form มี Properties ที่สำคัญดังนี้:

```pascal
// ตัวอย่างการกำหนด Properties ของ Form ใน Code
procedure TMainForm.FormCreate(Sender: TObject);
begin
  // กำหนดชื่อ Form ที่แสดงใน Title Bar
  Caption := 'โปรแกรมตัวอย่าง';
  
  // กำหนดขนาด Form
  Width := 800;
  Height := 600;
  
  // กำหนดตำแหน่งการแสดงผล
  Position := poScreenCenter; // แสดงตรงกลางหน้าจอ
  
  // กำหนดสีพื้นหลัง
  Color := clWhite;
  
  // กำหนดว่า Form สามารถ Resize ได้หรือไม่
  BorderStyle := bsSizeable; // หรือ bsDialog, bsNone, bsFixed3D
  
  // กำหนด Font เริ่มต้นของ Form
  Font.Name := 'Tahoma';
  Font.Size := 10;
  
  // กำหนดไอคอน Form
  // Icon.LoadFromFile('myicon.ico');
end;
```

### 16.1.2 Border Styles

```pascal
// Border Style ต่างๆ ที่ใช้ได้
// bsNone - ไม่มี Border
// bsSingle - Border แบบ Single (ไม่สามารถ Resize ได้)
// bsSizeable - Border แบบ Sizeable (สามารถ Resize ได้) - ค่าเริ่มต้น
// bsDialog - Border แบบ Dialog (ปุ่ม Maximize/Minimize ถูกซ่อน)
// bsToolWindow - Border แบบ Tool Window (ขนาดเล็ก, ไม่ปรากฏใน Taskbar)
// bsSizeToolWin - Border แบบ Sizable Tool Window

procedure TForm1.SetFormStyle;
begin
  BorderStyle := bsDialog; // เหมาะสำหรับ Dialog Box
  BorderIcons := [biSystemMenu]; // แสดงเฉพาะ System Menu
end;
```

### 16.1.3 เมธอด (Methods) ของ TForm

```pascal
procedure TMainForm.ButtonExampleClick(Sender: TObject);
begin
  // การแสดง Form ใหม่
  Form2.Show;        // แสดง Form แบบ Non-modal
  Form2.ShowModal;   // แสดง Form แบบ Modal (รอจนกว่าจะปิด)
  
  // การปิด Form
  Form2.Hide;        // ซ่อน Form
  Form2.Close;       // ปิด Form
  
  // การ Refresh Form
  Refresh;           // วาด Form ใหม่
  Invalidate;        // สั่งให้วาดใหม่ในรอบถัดไป
  
  // การ Focus
  SetFocus;          // กำหนด Focus ให้ Form นี้
  BringToFront;      // นำ Form มาแสดงด้านหน้าสุด
  SendToBack;        // ส่ง Form ไปอยู่ด้านหลังสุด
end;
```

### 16.1.4 Events ของ TForm

```pascal
// OnCreate - เกิดขึ้นเมื่อ Form ถูกสร้าง
procedure TMainForm.FormCreate(Sender: TObject);
begin
  // เริ่มต้นตัวแปรและโหลดข้อมูล
  Caption := 'Application เริ่มต้น';
  LoadSettings;
end;

// OnShow - เกิดขึ้นเมื่อ Form ถูกแสดง
procedure TMainForm.FormShow(Sender: TObject);
begin
  // ทำสิ่งที่ต้องการเมื่อ Form แสดง
  StatusBar1.Panels[0].Text := 'พร้อมใช้งาน';
end;

// OnClose - เกิดขึ้นเมื่อกำลังจะปิด Form
procedure TMainForm.FormClose(Sender: TObject; var CloseAction: TCloseAction);
begin
  // บันทึกข้อมูลก่อนปิด
  if MessageDlg('ต้องการบันทึกข้อมูลก่อนออกหรือไม่?', 
                mtConfirmation, [mbYes, mbNo, mbCancel], 0) = mrCancel then
  begin
    CloseAction := caNone; // ยกเลิกการปิด Form
  end
  else
  begin
    CloseAction := caFree; // ปิดและ Free Form
    SaveSettings;
  end;
end;

// OnResize - เกิดขึ้นเมื่อขนาด Form เปลี่ยน
procedure TMainForm.FormResize(Sender: TObject);
begin
  // ปรับตำแหน่ง Controls เมื่อ Form ถูก Resize
  Panel1.Width := ClientWidth;
  Memo1.Height := ClientHeight - Panel1.Height - StatusBar1.Height;
end;
```

---

## 16.2 TButton, TBitBtn, TSpeedButton

### 16.2.1 TButton - ปุ่มพื้นฐาน

```pascal
// การสร้าง TButton ด้วย Code
procedure TMainForm.CreateDynamicButton;
var
  btn: TButton;
begin
  btn := TButton.Create(Self);
  btn.Parent := Self;           // กำหนด Parent
  btn.Caption := 'คลิกฉัน';    // ข้อความบนปุ่ม
  btn.Left := 100;              // ตำแหน่ง X
  btn.Top := 100;               // ตำแหน่ง Y
  btn.Width := 120;             // ความกว้าง
  btn.Height := 35;             // ความสูง
  btn.Font.Size := 10;
  btn.OnClick := ButtonClickHandler; // กำหนด Event Handler
end;

// Event Handler
procedure TMainForm.ButtonClickHandler(Sender: TObject);
begin
  ShowMessage('ปุ่มถูกคลิก!');
end;

// Properties ที่สำคัญของ TButton
procedure TMainForm.ButtonPropertiesExample;
begin
  Button1.Caption := 'บันทึก';       // ข้อความ
  Button1.Enabled := True;            // เปิด/ปิดใช้งาน
  Button1.Visible := True;            // แสดง/ซ่อน
  Button1.Default := True;            // ปุ่ม Default (ตอบสนองต่อ Enter)
  Button1.Cancel := False;            // ปุ่ม Cancel (ตอบสนองต่อ Escape)
  Button1.ModalResult := mrOk;        // ผลลัพธ์เมื่อใช้ใน Dialog
  Button1.TabStop := True;            // รับ Focus ด้วย Tab
  Button1.TabOrder := 0;              // ลำดับ Tab
end;
```

### 16.2.2 TBitBtn - ปุ่มพร้อมรูปภาพ

```pascal
// TBitBtn - ปุ่มที่มีรูปภาพ
procedure TMainForm.SetupBitBtns;
begin
  // กำหนด Kind ของ TBitBtn (จะได้รูปและข้อความอัตโนมัติ)
  BitBtn1.Kind := bkOK;       // ปุ่ม OK
  BitBtn2.Kind := bkCancel;   // ปุ่ม Cancel
  BitBtn3.Kind := bkYes;      // ปุ่ม Yes
  BitBtn4.Kind := bkNo;       // ปุ่ม No
  BitBtn5.Kind := bkHelp;     // ปุ่ม Help
  BitBtn6.Kind := bkClose;    // ปุ่ม Close
  
  // กำหนดรูปภาพเอง
  BitBtn7.Glyph.LoadFromFile('save.bmp');
  BitBtn7.Caption := 'บันทึก';
  BitBtn7.NumGlyphs := 2; // รูปภาพ 2 สถานะ (ปกติ และ Disabled)
  
  // ตำแหน่งของรูปภาพ
  BitBtn7.Layout := blGlyphLeft;   // รูปอยู่ซ้าย
  // หรือ blGlyphRight, blGlyphTop, blGlyphBottom
  
  BitBtn7.Spacing := 8; // ระยะห่างระหว่างรูปและข้อความ
end;
```

### 16.2.3 TSpeedButton - ปุ่มสำหรับ Toolbar

```pascal
// TSpeedButton - ปุ่มแบบ Flat สำหรับ Toolbar
procedure TMainForm.SetupSpeedButtons;
begin
  SpeedButton1.Flat := True;        // แบบ Flat (ไม่มีเส้นขอบ)
  SpeedButton1.AllowAllUp := True;  // อนุญาตให้ไม่มีปุ่มใดถูกกด
  SpeedButton1.GroupIndex := 1;     // กลุ่มของปุ่ม (เลือกได้ทีละปุ่ม)
  SpeedButton1.Down := False;       // สถานะกด/ไม่กด
  
  // โหลดรูปภาพ
  SpeedButton1.Glyph.LoadFromFile('bold.bmp');
  SpeedButton1.Hint := 'ตัวหนา (Ctrl+B)'; // Tooltip
  SpeedButton1.ShowHint := True;
end;

// การสร้าง Toolbar ด้วย SpeedButtons
procedure TMainForm.CreateToolbar;
var
  sb: TSpeedButton;
  i: Integer;
  captions: array[0..4] of string = ('ใหม่', 'เปิด', 'บันทึก', '', 'พิมพ์');
begin
  for i := 0 to 4 do
  begin
    if captions[i] = '' then Continue; // ข้าม Separator
    
    sb := TSpeedButton.Create(Self);
    sb.Parent := Panel1; // วางบน Panel
    sb.Left := i * 30 + 5;
    sb.Top := 5;
    sb.Width := 25;
    sb.Height := 25;
    sb.Caption := captions[i];
    sb.Flat := True;
    sb.ShowHint := True;
  end;
end;
```

---

## 16.3 TLabel และ TStaticText

### 16.3.1 TLabel - ป้ายข้อความ

```pascal
// การใช้งาน TLabel
procedure TMainForm.LabelExample;
begin
  // กำหนด Properties ของ Label
  Label1.Caption := 'ชื่อ:';
  Label1.Font.Bold := True;
  Label1.Font.Size := 12;
  Label1.Font.Color := clNavy;
  
  // กำหนดการ Align ของข้อความ
  Label1.Alignment := taLeftJustify;   // ชิดซ้าย
  // หรือ taRightJustify, taCenter
  
  // WordWrap - ตัดบรรทัดอัตโนมัติ
  Label1.WordWrap := True;
  Label1.AutoSize := False; // ต้องปิด AutoSize เพื่อใช้ WordWrap
  Label1.Width := 200;
  Label1.Height := 60;
  
  // Label แบบ Transparent
  Label1.Transparent := True;
  
  // Label แบบมีเส้นขอบ (ใช้แสดงข้อมูล)
  Label1.BorderSpacing.Around := 5;
end;

// การใช้ Label เป็น Hyperlink
procedure TMainForm.SetupHyperlinkLabel;
begin
  Label1.Caption := 'คลิกที่นี่';
  Label1.Font.Color := clBlue;
  Label1.Font.Style := [fsUnderline];
  Label1.Cursor := crHandPoint;
  Label1.OnClick := LabelLinkClick;
end;

procedure TMainForm.LabelLinkClick(Sender: TObject);
begin
  OpenURL('https://www.lazarus-ide.org');
end;
```

### 16.3.2 TStaticText

```pascal
// TStaticText - คล้าย TLabel แต่เป็น Windows Control จริง
procedure TMainForm.StaticTextExample;
begin
  StaticText1.Caption := 'ข้อความ Static';
  StaticText1.BorderStyle := sbsSingle; // มีเส้นขอบ
  // หรือ sbsNone - ไม่มีเส้นขอบ
  
  StaticText1.Alignment := taCenter;
end;
```

---

## 16.4 TEdit, TMemo, TRichEdit

### 16.4.1 TEdit - กล่องรับข้อความบรรทัดเดียว

```pascal
// การใช้งาน TEdit
procedure TMainForm.EditExample;
begin
  // กำหนด Properties
  Edit1.Text := 'ข้อความเริ่มต้น';
  Edit1.MaxLength := 50;         // จำกัดความยาวข้อความ
  Edit1.PasswordChar := '*';    // ใช้สำหรับรหัสผ่าน
  Edit1.ReadOnly := False;      // อ่านได้อย่างเดียว
  Edit1.CharCase := ecUpperCase; // ตัวพิมพ์ใหญ่ทั้งหมด
  // ecLowerCase, ecNormal
  
  // การดึงข้อมูล
  ShowMessage('ข้อความ: ' + Edit1.Text);
  
  // การเลือกข้อความ
  Edit1.SelectAll;              // เลือกทั้งหมด
  Edit1.ClearSelection;         // ลบส่วนที่เลือก
  Edit1.SelStart := 0;          // ตำแหน่งเริ่มต้นการเลือก
  Edit1.SelLength := 5;         // ความยาวที่เลือก
  Edit1.SelText := 'แทนที่';   // แทนที่ข้อความที่เลือก
  
  // Copy, Cut, Paste
  Edit1.CopyToClipboard;
  Edit1.CutToClipboard;
  Edit1.PasteFromClipboard;
end;

// Validation ใน TEdit
procedure TMainForm.Edit1Exit(Sender: TObject);
begin
  if Edit1.Text = '' then
  begin
    ShowMessage('กรุณากรอกข้อมูล');
    Edit1.SetFocus;
  end;
end;

// รับเฉพาะตัวเลข
procedure TMainForm.EditNumberKeyPress(Sender: TObject; var Key: Char);
begin
  if not (Key in ['0'..'9', #8, #13]) then
  begin
    Key := #0; // ยกเลิกการกด
    Beep;
  end;
end;
```

### 16.4.2 TMemo - กล่องรับข้อความหลายบรรทัด

```pascal
// การใช้งาน TMemo
procedure TMainForm.MemoExample;
begin
  // กำหนด Properties
  Memo1.Lines.Add('บรรทัดที่ 1');
  Memo1.Lines.Add('บรรทัดที่ 2');
  Memo1.Lines.Add('บรรทัดที่ 3');
  
  // การโหลดและบันทึกไฟล์
  Memo1.Lines.LoadFromFile('document.txt');
  Memo1.Lines.SaveToFile('document.txt');
  
  // Properties ต่างๆ
  Memo1.ScrollBars := ssVertical;  // แสดง Scrollbar แนวตั้ง
  // ssBoth, ssHorizontal, ssNone
  Memo1.WordWrap := True;          // ตัดบรรทัดอัตโนมัติ
  Memo1.ReadOnly := False;
  Memo1.MaxLength := 0;            // 0 = ไม่จำกัด
  
  // การเข้าถึงข้อมูล
  ShowMessage('จำนวนบรรทัด: ' + IntToStr(Memo1.Lines.Count));
  ShowMessage('บรรทัดที่ 1: ' + Memo1.Lines[0]);
  ShowMessage('ข้อความทั้งหมด: ' + Memo1.Text);
  
  // การเลือกและแก้ไข
  Memo1.SelectAll;
  Memo1.ClearSelection;
  
  // เพิ่มข้อความที่ตำแหน่ง Cursor
  Memo1.SelText := 'ข้อความใหม่';
  
  // Clear ทั้งหมด
  Memo1.Clear;
  
  // Find Text
  // ใช้ FindNext ใน Runtime
end;

// การใช้ Memo เป็น Log Viewer
procedure TMainForm.LogMessage(const Msg: string);
begin
  Memo1.Lines.Add(FormatDateTime('yyyy-mm-dd hh:nn:ss', Now) + ' - ' + Msg);
  // เลื่อนไปบรรทัดล่าสุด
  Memo1.SelStart := Length(Memo1.Text);
  Memo1.SelLength := 0;
  Memo1.ScrollBy(0, Memo1.Lines.Count * Memo1.Font.Height);
end;
```

### 16.4.3 TRichEdit - Rich Text Editor

```pascal
// การใช้งาน TRichEdit
procedure TMainForm.RichEditExample;
begin
  // โหลดและบันทึกไฟล์ RTF
  RichEdit1.Lines.LoadFromFile('document.rtf');
  RichEdit1.Lines.SaveToFile('document.rtf');
  
  // การจัดรูปแบบข้อความที่เลือก
  RichEdit1.SelAttributes.Bold := True;
  RichEdit1.SelAttributes.Italic := True;
  RichEdit1.SelAttributes.Underline := True;
  RichEdit1.SelAttributes.Color := clRed;
  RichEdit1.SelAttributes.Size := 14;
  RichEdit1.SelAttributes.Name := 'Arial';
  
  // การจัดรูปแบบย่อหน้า
  RichEdit1.Paragraph.Alignment := taCenter;
  RichEdit1.Paragraph.FirstIndent := 20;
  RichEdit1.Paragraph.LeftIndent := 10;
  RichEdit1.Paragraph.RightIndent := 10;
  
  // Color ของ Background
  RichEdit1.Color := clWhite;
  
  // การ Zoom
  // RichEdit ใน Lazarus ไม่รองรับ Zoom โดยตรง
  // ต้องใช้ Font Size แทน
end;

// การ Apply Format เมื่อ Select ข้อความ
procedure TMainForm.BoldButtonClick(Sender: TObject);
begin
  if RichEdit1.SelAttributes.Bold then
    RichEdit1.SelAttributes.Bold := False
  else
    RichEdit1.SelAttributes.Bold := True;
  RichEdit1.SetFocus;
end;

procedure TMainForm.FontSizeComboChange(Sender: TObject);
var
  size: Integer;
begin
  if TryStrToInt(FontSizeCombo.Text, size) then
  begin
    RichEdit1.SelAttributes.Size := size;
    RichEdit1.SetFocus;
  end;
end;
```

---

## 16.5 TCheckBox และ TRadioButton

### 16.5.1 TCheckBox

```pascal
// การใช้งาน TCheckBox
procedure TMainForm.CheckBoxExample;
begin
  // กำหนดสถานะ
  CheckBox1.Checked := True;
  CheckBox1.Caption := 'ยอมรับเงื่อนไข';
  
  // AllowGrayed - อนุญาต State ที่ 3 (ไม่แน่ใจ)
  CheckBox1.AllowGrayed := True;
  CheckBox1.State := cbChecked;   // cbUnchecked, cbGrayed
  
  // ตรวจสอบสถานะ
  if CheckBox1.Checked then
    ShowMessage('ยอมรับเงื่อนไขแล้ว');
end;

// Event OnClick ของ CheckBox
procedure TMainForm.CheckBox1Click(Sender: TObject);
begin
  Button1.Enabled := CheckBox1.Checked;
end;

// การทำ Select All Checkbox
procedure TMainForm.CheckBoxSelectAllClick(Sender: TObject);
var
  i: Integer;
begin
  for i := 0 to ComponentCount - 1 do
    if Components[i] is TCheckBox then
      TCheckBox(Components[i]).Checked := CheckBoxSelectAll.Checked;
end;
```

### 16.5.2 TRadioButton

```pascal
// การใช้งาน TRadioButton
procedure TMainForm.RadioButtonExample;
begin
  // RadioButton ในกลุ่มเดียวกัน (Parent เดียวกัน) เลือกได้ทีละปุ่ม
  RadioButton1.Caption := 'เลือกตัวเลือกที่ 1';
  RadioButton2.Caption := 'เลือกตัวเลือกที่ 2';
  RadioButton3.Caption := 'เลือกตัวเลือกที่ 3';
  
  // กำหนดค่าเริ่มต้น
  RadioButton1.Checked := True;
  
  // ตรวจสอบว่าเลือกอะไร
  if RadioButton1.Checked then
    ShowMessage('เลือกตัวเลือกที่ 1')
  else if RadioButton2.Checked then
    ShowMessage('เลือกตัวเลือกที่ 2')
  else if RadioButton3.Checked then
    ShowMessage('เลือกตัวเลือกที่ 3');
end;

// หาว่า RadioButton ไหนถูกเลือก
function TMainForm.GetSelectedOption: Integer;
var
  i: Integer;
begin
  Result := -1;
  for i := 0 to GroupBox1.ControlCount - 1 do
    if (GroupBox1.Controls[i] is TRadioButton) and
       TRadioButton(GroupBox1.Controls[i]).Checked then
    begin
      Result := i;
      Break;
    end;
end;
```

### 16.5.3 TGroupBox

```pascal
// การใช้งาน TGroupBox
procedure TMainForm.GroupBoxExample;
begin
  GroupBox1.Caption := 'เลือกเพศ';
  
  // RadioButton ใน GroupBox เดียวกัน จะแยกจาก GroupBox อื่น
  // ทำให้เลือกได้อิสระจากกัน
  
  // GroupBox สามารถ Disabled ทั้งกลุ่มได้
  GroupBox1.Enabled := False; // Disable ทุก Control ใน Group
end;
```

---

## 16.6 TListBox และ TComboBox

### 16.6.1 TListBox

```pascal
// การใช้งาน TListBox
procedure TMainForm.ListBoxExample;
begin
  // เพิ่มรายการ
  ListBox1.Items.Add('รายการที่ 1');
  ListBox1.Items.Add('รายการที่ 2');
  ListBox1.Items.Add('รายการที่ 3');
  
  // เพิ่มหลายรายการพร้อมกัน
  ListBox1.Items.BeginUpdate;
  try
    ListBox1.Items.AddStrings(['A', 'B', 'C', 'D']);
  finally
    ListBox1.Items.EndUpdate;
  end;
  
  // แทรกรายการ
  ListBox1.Items.Insert(0, 'รายการแรกสุด');
  
  // ลบรายการ
  ListBox1.Items.Delete(0);         // ลบรายการที่ Index 0
  ListBox1.Items.Remove('รายการที่ 1'); // ลบตามชื่อ
  ListBox1.Items.Clear;             // ลบทั้งหมด
  
  // MultiSelect
  ListBox1.MultiSelect := True;     // เลือกหลายรายการได้
  ListBox1.ExtendedSelect := True;  // ใช้ Shift/Ctrl เพื่อเลือก
  
  // ตรวจสอบการเลือก
  if ListBox1.ItemIndex >= 0 then
    ShowMessage('เลือก: ' + ListBox1.Items[ListBox1.ItemIndex]);
  
  // นับรายการที่เลือก (สำหรับ MultiSelect)
  ShowMessage('เลือก ' + IntToStr(ListBox1.SelCount) + ' รายการ');
  
  // Sort
  ListBox1.Sorted := True; // เรียงลำดับอัตโนมัติ
  
  // Style
  ListBox1.Style := lbOwnerDrawFixed; // Custom Draw
  // lbStandard, lbOwnerDrawFixed, lbOwnerDrawVariable
end;

// หา Index ของรายการ
function TMainForm.FindItemInListBox(const ItemText: string): Integer;
begin
  Result := ListBox1.Items.IndexOf(ItemText);
end;

// Event OnClick ของ ListBox
procedure TMainForm.ListBox1Click(Sender: TObject);
begin
  if ListBox1.ItemIndex >= 0 then
    Label1.Caption := 'เลือก: ' + ListBox1.Items[ListBox1.ItemIndex];
end;

// Event OnDblClick เปิดรายการ
procedure TMainForm.ListBox1DblClick(Sender: TObject);
begin
  if ListBox1.ItemIndex >= 0 then
    OpenItem(ListBox1.Items[ListBox1.ItemIndex]);
end;
```

### 16.6.2 TComboBox

```pascal
// การใช้งาน TComboBox
procedure TMainForm.ComboBoxExample;
begin
  // เพิ่มรายการ
  ComboBox1.Items.Add('ตัวเลือกที่ 1');
  ComboBox1.Items.Add('ตัวเลือกที่ 2');
  ComboBox1.Items.Add('ตัวเลือกที่ 3');
  
  // กำหนดค่าเริ่มต้น
  ComboBox1.ItemIndex := 0; // เลือกรายการแรก
  
  // Style ของ ComboBox
  ComboBox1.Style := csDropDown;       // พิมพ์ได้ + เลือกจาก List
  // csSimple - แสดง List ตลอดเวลา
  // csDropDownList - เลือกจาก List เท่านั้น (พิมพ์ไม่ได้)
  // csOwnerDrawFixed, csOwnerDrawVariable - Custom Draw
  
  // ดึงค่าที่เลือก
  ShowMessage('เลือก: ' + ComboBox1.Text);
  ShowMessage('Index: ' + IntToStr(ComboBox1.ItemIndex));
  
  // ค้นหา
  ComboBox1.ItemIndex := ComboBox1.Items.IndexOf('ตัวเลือกที่ 2');
  
  // AutoComplete
  ComboBox1.AutoComplete := True;
  
  // DropDownCount - จำนวนรายการที่แสดงใน Dropdown
  ComboBox1.DropDownCount := 8;
  
  // Sort
  ComboBox1.Sorted := True;
end;

// Event OnChange
procedure TMainForm.ComboBox1Change(Sender: TObject);
begin
  ShowMessage('เปลี่ยนเป็น: ' + ComboBox1.Text);
  UpdateFormBasedOnSelection(ComboBox1.ItemIndex);
end;

// ตัวอย่างการโหลดรายการจากฐานข้อมูล
procedure TMainForm.LoadCountriesToComboBox;
const
  Countries: array[0..4] of string = (
    'ไทย', 'ญี่ปุ่น', 'เกาหลี', 'จีน', 'สหรัฐอเมริกา'
  );
var
  i: Integer;
begin
  ComboBox1.Items.BeginUpdate;
  try
    ComboBox1.Items.Clear;
    for i := 0 to High(Countries) do
      ComboBox1.Items.Add(Countries[i]);
  finally
    ComboBox1.Items.EndUpdate;
  end;
  ComboBox1.ItemIndex := 0;
end;
```

---

## 16.7 TScrollBar, TTrackBar, TProgressBar

### 16.7.1 TScrollBar

```pascal
// การใช้งาน TScrollBar
procedure TMainForm.ScrollBarExample;
begin
  ScrollBar1.Kind := sbHorizontal; // หรือ sbVertical
  ScrollBar1.Min := 0;
  ScrollBar1.Max := 100;
  ScrollBar1.Position := 50;       // ตำแหน่งปัจจุบัน
  ScrollBar1.SmallChange := 1;     // การเปลี่ยนแปลงเล็กน้อย (Arrow Key)
  ScrollBar1.LargeChange := 10;    // การเปลี่ยนแปลงมาก (Page Up/Down)
  
  // อ่านค่า
  ShowMessage('ตำแหน่ง: ' + IntToStr(ScrollBar1.Position));
end;

// Event OnChange
procedure TMainForm.ScrollBar1Change(Sender: TObject);
begin
  Label1.Caption := 'ค่า: ' + IntToStr(ScrollBar1.Position);
  Image1.Left := -ScrollBar1.Position * 5; // เลื่อน Image
end;
```

### 16.7.2 TTrackBar

```pascal
// การใช้งาน TTrackBar
procedure TMainForm.TrackBarExample;
begin
  TrackBar1.Min := 0;
  TrackBar1.Max := 100;
  TrackBar1.Position := 50;
  TrackBar1.Frequency := 10;  // ระยะห่างของ Tick
  TrackBar1.TickStyle := tsAuto;
  TrackBar1.Orientation := trHorizontal; // หรือ trVertical
  
  // อ่านค่า
  ShowMessage('ค่า TrackBar: ' + IntToStr(TrackBar1.Position));
end;

// ใช้ TrackBar ควบคุม Volume
procedure TMainForm.TrackBar1Change(Sender: TObject);
begin
  // ตัวอย่างควบคุม Volume (Pseudo code)
  LabelVolume.Caption := IntToStr(TrackBar1.Position) + '%';
  // SetSystemVolume(TrackBar1.Position);
end;
```

### 16.7.3 TProgressBar

```pascal
// การใช้งาน TProgressBar
procedure TMainForm.ProgressBarExample;
begin
  ProgressBar1.Min := 0;
  ProgressBar1.Max := 100;
  ProgressBar1.Position := 0;
  ProgressBar1.Step := 10;    // ขนาดขั้น
  ProgressBar1.Style := pbstNormal; // หรือ pbstMarquee
  // pbstMarquee - แสดงการเคลื่อนไหวต่อเนื่อง (ไม่รู้เปอร์เซ็นต์)
  
  // เลื่อน Progress
  ProgressBar1.Position := 25;
  ProgressBar1.StepIt;         // เลื่อน 1 Step
  ProgressBar1.StepBy(5);      // เลื่อน N Step
end;

// ตัวอย่างการใช้ ProgressBar ระหว่างการประมวลผล
procedure TMainForm.ProcessWithProgress;
var
  i: Integer;
begin
  ProgressBar1.Min := 0;
  ProgressBar1.Max := 100;
  
  for i := 0 to 99 do
  begin
    // ทำงานบางอย่าง
    Sleep(50); // จำลองการทำงาน
    
    ProgressBar1.Position := i + 1;
    Application.ProcessMessages; // อัปเดต UI
  end;
  
  ShowMessage('เสร็จสิ้น!');
end;
```

---

## 16.8 TPanel, TScrollBox

### 16.8.1 TPanel - Container อเนกประสงค์

```pascal
// การใช้งาน TPanel
procedure TMainForm.PanelExample;
begin
  Panel1.Caption := '';          // ซ่อน Caption
  Panel1.Align := alTop;         // จัดตำแหน่งด้านบน
  Panel1.Height := 40;
  Panel1.BevelOuter := bvNone;   // ไม่มีเส้นขอบด้านนอก
  Panel1.BevelInner := bvNone;   // ไม่มีเส้นขอบด้านใน
  Panel1.Color := clBtnFace;
  
  // Panel เป็น Toolbar
  Panel1.Align := alTop;
  Panel1.Height := 35;
  
  // Panel เป็น Status Bar
  Panel2.Align := alBottom;
  Panel2.Height := 24;
  Panel2.BevelInner := bvLowered;
end;

// ใช้ Panel แบ่งพื้นที่
procedure TMainForm.SetupLayout;
begin
  // Toolbar Panel ด้านบน
  PanelToolbar.Align := alTop;
  PanelToolbar.Height := 35;
  
  // Panel ด้านซ้ายสำหรับ Tree/List
  PanelLeft.Align := alLeft;
  PanelLeft.Width := 200;
  
  // Panel หลักตรงกลาง
  PanelMain.Align := alClient;
  
  // Status Bar ด้านล่าง
  PanelStatus.Align := alBottom;
  PanelStatus.Height := 24;
end;
```

### 16.8.2 TScrollBox

```pascal
// การใช้งาน TScrollBox
procedure TMainForm.ScrollBoxExample;
begin
  ScrollBox1.AutoScroll := True; // เปิด Auto Scroll
  ScrollBox1.Color := clWhite;
  
  // เพิ่ม Controls ที่ขนาดใหญ่กว่า ScrollBox
  // Controls จะสามารถ Scroll ได้อัตโนมัติ
  
  // เลื่อนไปตำแหน่งที่ต้องการ
  ScrollBox1.ScrollBy(0, 100); // เลื่อนลง 100 pixel
  ScrollBox1.VertScrollBar.Position := 0; // เลื่อนกลับด้านบน
end;
```

---

## 16.9 TImage

```pascal
// การใช้งาน TImage
procedure TMainForm.ImageExample;
begin
  // โหลดรูปภาพ
  Image1.Picture.LoadFromFile('photo.jpg');
  Image1.Picture.LoadFromFile('icon.png');
  Image1.Picture.LoadFromFile('drawing.bmp');
  
  // การแสดงผล
  Image1.Stretch := True;       // ยืดรูปให้พอดี Image Control
  Image1.StretchInEnabled := False; // ยืดออก แต่ไม่ย่อ
  Image1.Proportional := True;  // ยืดแบบรักษาสัดส่วน
  Image1.Center := True;        // จัดรูปให้อยู่กลาง
  
  // ขนาดจริงของรูป
  ShowMessage('ขนาด: ' + IntToStr(Image1.Picture.Width) + 'x' +
              IntToStr(Image1.Picture.Height));
  
  // Clear รูป
  Image1.Picture.Clear;
  
  // วาดบน Image (ใช้ Canvas)
  Image1.Canvas.Pen.Color := clRed;
  Image1.Canvas.Pen.Width := 3;
  Image1.Canvas.MoveTo(0, 0);
  Image1.Canvas.LineTo(Image1.Width, Image1.Height);
end;

// โหลดรูปจาก Resource
procedure TMainForm.LoadImageFromResource;
var
  ResStream: TResourceStream;
begin
  ResStream := TResourceStream.Create(HInstance, 'MY_IMAGE', RT_RCDATA);
  try
    Image1.Picture.LoadFromStream(ResStream);
  finally
    ResStream.Free;
  end;
end;
```

---

## 16.10 TPageControl และ TNotebook

### 16.10.1 TPageControl - Tab Pages

```pascal
// การใช้งาน TPageControl
procedure TMainForm.PageControlExample;
begin
  // เพิ่ม Tab Page ใหม่
  TabSheet1.Caption := 'หน้าที่ 1';
  TabSheet2.Caption := 'หน้าที่ 2';
  TabSheet3.Caption := 'หน้าที่ 3';
  
  // เปลี่ยน Tab
  PageControl1.ActivePage := TabSheet2;
  PageControl1.ActivePageIndex := 1; // Index เริ่มที่ 0
  
  // ซ่อน Tab
  TabSheet3.TabVisible := False;
  
  // Tab ที่ Active
  ShowMessage('Tab ปัจจุบัน: ' + PageControl1.ActivePage.Caption);
  
  // Tab Style
  PageControl1.TabPosition := tpTop;    // Tab อยู่ด้านบน
  // tpBottom, tpLeft, tpRight
  
  PageControl1.Style := tsFlatButtons; // Style ของ Tab
  // tsNormal, tsOwnerDraw, tsTabs, tsButtons, tsFlatButtons
end;

// สร้าง TabSheet ด้วย Code
procedure TMainForm.AddTabDynamically;
var
  newTab: TTabSheet;
  memo: TMemo;
begin
  newTab := TTabSheet.Create(PageControl1);
  newTab.PageControl := PageControl1;
  newTab.Caption := 'เอกสาร ' + IntToStr(PageControl1.PageCount);
  
  // เพิ่ม Memo ใน Tab ใหม่
  memo := TMemo.Create(newTab);
  memo.Parent := newTab;
  memo.Align := alClient;
  memo.ScrollBars := ssBoth;
end;

// Event OnChange ของ PageControl
procedure TMainForm.PageControl1Change(Sender: TObject);
begin
  StatusBar1.Panels[0].Text := 'Tab: ' + PageControl1.ActivePage.Caption;
end;
```

---

## 16.11 Anchors และ Constraints

### 16.11.1 Anchors

```pascal
// Anchors ใช้กำหนดว่า Control ยึดติดกับขอบ Form ด้านใด
procedure TMainForm.AnchorExample;
begin
  // ยึดด้านบนและซ้าย (ค่าเริ่มต้น)
  Button1.Anchors := [akTop, akLeft];
  
  // ยึดด้านบน ซ้าย และขวา (จะขยายตามความกว้าง)
  Edit1.Anchors := [akTop, akLeft, akRight];
  
  // ยึดทุกด้าน (จะขยายตามทั้งความกว้างและความสูง)
  Memo1.Anchors := [akTop, akLeft, akRight, akBottom];
  
  // ยึดด้านล่างและขวา (จะเลื่อนตามมุมล่างขวา)
  Button2.Anchors := [akRight, akBottom];
end;
```

### 16.11.2 Constraints

```pascal
// Constraints ใช้กำหนดขนาดสูงสุดและต่ำสุดของ Control/Form
procedure TMainForm.ConstraintsExample;
begin
  // กำหนดขนาดขั้นต่ำของ Form
  Constraints.MinWidth := 400;
  Constraints.MinHeight := 300;
  
  // กำหนดขนาดสูงสุดของ Form
  Constraints.MaxWidth := 1200;
  Constraints.MaxHeight := 900;
  
  // กำหนดขนาดของ Control
  Panel1.Constraints.MinWidth := 100;
  Panel1.Constraints.MinHeight := 50;
end;
```

---

## 16.12 Z-Order การซ้อนทับของ Controls

```pascal
// การจัดการ Z-Order
procedure TMainForm.ZOrderExample;
begin
  // นำ Control มาด้านหน้า
  Panel1.BringToFront;
  
  // ส่ง Control ไปด้านหลัง
  Image1.SendToBack;
  
  // ตั้งค่า Z-Order
  // Controls ที่สร้างทีหลังจะอยู่ด้านหน้า
  
  // ใน Lazarus ใช้ SetZOrder ผ่าน WinControl
  // Button1.SetZOrder(True);  // นำขึ้นมาด้านหน้า
end;
```

---

## 16.13 การจัดการ Focus

```pascal
// การทำงานกับ Focus
procedure TMainForm.FocusExample;
begin
  // กำหนด Focus ให้ Control
  Edit1.SetFocus;
  
  // ตรวจสอบว่า Control ไหนมี Focus
  if ActiveControl = Edit1 then
    ShowMessage('Edit1 มี Focus');
  
  // ตรวจสอบว่า Control รับ Focus ได้หรือไม่
  if Edit1.CanFocus then
    Edit1.SetFocus;
  
  // Tab Order
  Edit1.TabOrder := 0;
  Edit2.TabOrder := 1;
  Button1.TabOrder := 2;
  
  // TabStop - กำหนดว่า Tab จะหยุดที่ Control นี้หรือไม่
  Label1.TabStop := False;
  Edit1.TabStop := True;
end;

// Event OnEnter - เมื่อ Control ได้ Focus
procedure TMainForm.Edit1Enter(Sender: TObject);
begin
  Edit1.Color := clYellow; // เปลี่ยนสีเมื่อ Active
end;

// Event OnExit - เมื่อ Control เสีย Focus
procedure TMainForm.Edit1Exit(Sender: TObject);
begin
  Edit1.Color := clWhite; // คืนสีเดิม
  ValidateEdit1;           // Validate ข้อมูล
end;
```

---

## 16.14 Validation

```pascal
// ตัวอย่าง Validation ครบถ้วน
unit ValidationExample;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, RegExpr;

type
  TValidationForm = class(TForm)
    EditName: TEdit;
    EditEmail: TEdit;
    EditPhone: TEdit;
    EditAge: TEdit;
    ButtonSubmit: TButton;
    LabelError: TLabel;
    procedure ButtonSubmitClick(Sender: TObject);
    procedure EditNameExit(Sender: TObject);
    procedure EditEmailExit(Sender: TObject);
  private
    function ValidateName(const Name: string): string;
    function ValidateEmail(const Email: string): string;
    function ValidatePhone(const Phone: string): string;
    function ValidateAge(const AgeStr: string): string;
    function IsValidEmail(const Email: string): Boolean;
    procedure ShowFieldError(Edit: TEdit; const Msg: string);
    procedure ClearFieldError(Edit: TEdit);
  end;

implementation

function TValidationForm.ValidateName(const Name: string): string;
begin
  Result := '';
  if Trim(Name) = '' then
    Result := 'กรุณากรอกชื่อ'
  else if Length(Trim(Name)) < 2 then
    Result := 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร'
  else if Length(Trim(Name)) > 100 then
    Result := 'ชื่อต้องไม่เกิน 100 ตัวอักษร';
end;

function TValidationForm.ValidateEmail(const Email: string): string;
begin
  Result := '';
  if Trim(Email) = '' then
    Result := 'กรุณากรอกอีเมล'
  else if not IsValidEmail(Email) then
    Result := 'รูปแบบอีเมลไม่ถูกต้อง';
end;

function TValidationForm.IsValidEmail(const Email: string): Boolean;
var
  AtPos, DotPos: Integer;
begin
  AtPos := Pos('@', Email);
  Result := False;
  if AtPos < 2 then Exit;
  DotPos := LastDelimiter('.', Email);
  if DotPos <= AtPos + 1 then Exit;
  if DotPos >= Length(Email) then Exit;
  Result := True;
end;

function TValidationForm.ValidatePhone(const Phone: string): string;
var
  i: Integer;
  digits: string;
begin
  Result := '';
  digits := '';
  for i := 1 to Length(Phone) do
    if Phone[i] in ['0'..'9'] then
      digits := digits + Phone[i];
      
  if Length(digits) < 9 then
    Result := 'เบอร์โทรศัพท์ต้องมีอย่างน้อย 9 หลัก'
  else if Length(digits) > 10 then
    Result := 'เบอร์โทรศัพท์ต้องไม่เกิน 10 หลัก';
end;

function TValidationForm.ValidateAge(const AgeStr: string): string;
var
  age: Integer;
begin
  Result := '';
  if not TryStrToInt(AgeStr, age) then
    Result := 'อายุต้องเป็นตัวเลขเท่านั้น'
  else if age < 1 then
    Result := 'อายุต้องมากกว่า 0'
  else if age > 150 then
    Result := 'อายุไม่ถูกต้อง';
end;

procedure TValidationForm.ShowFieldError(Edit: TEdit; const Msg: string);
begin
  Edit.Color := $00CCFFFF; // สีพื้นหลังแดงอ่อน
  LabelError.Caption := Msg;
  LabelError.Visible := True;
end;

procedure TValidationForm.ClearFieldError(Edit: TEdit);
begin
  Edit.Color := clWhite;
end;

procedure TValidationForm.ButtonSubmitClick(Sender: TObject);
var
  errors: TStringList;
  errMsg: string;
begin
  errors := TStringList.Create;
  try
    errMsg := ValidateName(EditName.Text);
    if errMsg <> '' then errors.Add('ชื่อ: ' + errMsg);
    
    errMsg := ValidateEmail(EditEmail.Text);
    if errMsg <> '' then errors.Add('อีเมล: ' + errMsg);
    
    errMsg := ValidatePhone(EditPhone.Text);
    if errMsg <> '' then errors.Add('โทรศัพท์: ' + errMsg);
    
    errMsg := ValidateAge(EditAge.Text);
    if errMsg <> '' then errors.Add('อายุ: ' + errMsg);
    
    if errors.Count > 0 then
    begin
      ShowMessage('พบข้อผิดพลาด:' + #13#10 + errors.Text);
    end
    else
    begin
      ShowMessage('ข้อมูลถูกต้องทั้งหมด! กำลังบันทึก...');
      SaveData;
    end;
  finally
    errors.Free;
  end;
end;

procedure TValidationForm.EditNameExit(Sender: TObject);
var
  errMsg: string;
begin
  errMsg := ValidateName(EditName.Text);
  if errMsg <> '' then
    ShowFieldError(EditName, errMsg)
  else
    ClearFieldError(EditName);
end;

procedure TValidationForm.EditEmailExit(Sender: TObject);
var
  errMsg: string;
begin
  errMsg := ValidateEmail(EditEmail.Text);
  if errMsg <> '' then
    ShowFieldError(EditEmail, errMsg)
  else
    ClearFieldError(EditEmail);
end;
```

---

## 16.15 โปรแกรมตัวอย่าง: User Registration Form

### ไฟล์: RegistrationForm.pas

```pascal
unit RegistrationForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls, Spin, DateTimePicker;

type
  TUserRegistration = record
    FirstName: string;
    LastName: string;
    Email: string;
    Phone: string;
    BirthDate: TDate;
    Gender: string;
    Address: string;
    City: string;
    Country: string;
    Password: string;
    AcceptTerms: Boolean;
  end;

  TRegistrationForm = class(TForm)
    // Header
    PanelHeader: TPanel;
    LabelTitle: TLabel;
    
    // Personal Info Group
    GroupBoxPersonal: TGroupBox;
    LabelFirstName: TLabel;
    EditFirstName: TEdit;
    LabelLastName: TLabel;
    EditLastName: TEdit;
    LabelBirthDate: TLabel;
    DatePickerBirth: TDateTimePicker;
    LabelGender: TLabel;
    RadioMale: TRadioButton;
    RadioFemale: TRadioButton;
    RadioOther: TRadioButton;
    
    // Contact Group
    GroupBoxContact: TGroupBox;
    LabelEmail: TLabel;
    EditEmail: TEdit;
    LabelPhone: TLabel;
    EditPhone: TEdit;
    LabelAddress: TLabel;
    MemoAddress: TMemo;
    LabelCity: TLabel;
    EditCity: TEdit;
    LabelCountry: TLabel;
    ComboCountry: TComboBox;
    
    // Account Group
    GroupBoxAccount: TGroupBox;
    LabelPassword: TLabel;
    EditPassword: TEdit;
    LabelConfirmPwd: TLabel;
    EditConfirmPwd: TEdit;
    
    // Terms
    CheckBoxTerms: TCheckBox;
    LabelTermsLink: TLabel;
    
    // Buttons
    PanelButtons: TPanel;
    ButtonRegister: TButton;
    ButtonClear: TButton;
    ButtonCancel: TButton;
    
    // Status
    LabelStatus: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure ButtonRegisterClick(Sender: TObject);
    procedure ButtonClearClick(Sender: TObject);
    procedure ButtonCancelClick(Sender: TObject);
    procedure EditEmailExit(Sender: TObject);
    procedure EditPasswordExit(Sender: TObject);
    procedure CheckBoxTermsClick(Sender: TObject);
    procedure LabelTermsLinkClick(Sender: TObject);
    
  private
    FIsEditing: Boolean;
    
    procedure InitializeForm;
    procedure LoadCountries;
    function ValidateForm: Boolean;
    function ValidateEmail(const Email: string): Boolean;
    function ValidatePassword(const Pwd: string): Boolean;
    procedure GetFormData(var RegData: TUserRegistration);
    procedure ClearForm;
    procedure ShowError(Control: TWinControl; const Msg: string);
    procedure ClearErrors;
    procedure SaveRegistration(const RegData: TUserRegistration);
    
  public
    property IsEditing: Boolean read FIsEditing write FIsEditing;
  end;

var
  RegistrationForm: TRegistrationForm;

implementation

{$R *.lfm}

procedure TRegistrationForm.FormCreate(Sender: TObject);
begin
  InitializeForm;
  LoadCountries;
end;

procedure TRegistrationForm.InitializeForm;
begin
  Caption := 'ลงทะเบียนสมาชิก';
  Width := 600;
  Height := 700;
  Position := poScreenCenter;
  BorderStyle := bsDialog;
  
  // Header
  PanelHeader.Color := $00CC4400; // สีส้มเข้ม
  PanelHeader.Height := 60;
  LabelTitle.Caption := 'ลงทะเบียนสมาชิกใหม่';
  LabelTitle.Font.Size := 16;
  LabelTitle.Font.Color := clWhite;
  LabelTitle.Font.Bold := True;
  
  // Password Fields
  EditPassword.PasswordChar := '●';
  EditConfirmPwd.PasswordChar := '●';
  
  // Terms
  CheckBoxTerms.Caption := ' ฉันยอมรับเงื่อนไขการใช้งาน';
  ButtonRegister.Enabled := False;
  
  // Status Label
  LabelStatus.Caption := '';
  LabelStatus.Font.Color := clRed;
  
  // Default Gender
  RadioMale.Checked := True;
  
  // Date
  DatePickerBirth.Date := EncodeDate(1990, 1, 1);
  
  FIsEditing := False;
end;

procedure TRegistrationForm.LoadCountries;
const
  COUNTRIES: array[0..19] of string = (
    'ไทย', 'ญี่ปุ่น', 'เกาหลีใต้', 'จีน', 'สิงคโปร์',
    'มาเลเซีย', 'อินโดนีเซีย', 'ฟิลิปปินส์', 'เวียดนาม', 'เมียนมา',
    'กัมพูชา', 'ลาว', 'สหรัฐอเมริกา', 'สหราชอาณาจักร', 'ออสเตรเลีย',
    'แคนาดา', 'เยอรมนี', 'ฝรั่งเศส', 'อิตาลี', 'อื่นๆ'
  );
var
  i: Integer;
begin
  ComboCountry.Items.BeginUpdate;
  try
    ComboCountry.Items.Clear;
    for i := 0 to High(COUNTRIES) do
      ComboCountry.Items.Add(COUNTRIES[i]);
  finally
    ComboCountry.Items.EndUpdate;
  end;
  ComboCountry.ItemIndex := 0; // ไทย
end;

function TRegistrationForm.ValidateEmail(const Email: string): Boolean;
var
  AtPos: Integer;
begin
  Result := False;
  AtPos := Pos('@', Email);
  if AtPos < 2 then Exit;
  if Pos('.', Copy(Email, AtPos + 1, MaxInt)) < 2 then Exit;
  Result := True;
end;

function TRegistrationForm.ValidatePassword(const Pwd: string): Boolean;
begin
  // รหัสผ่านต้องมีความยาวอย่างน้อย 8 ตัวอักษร
  // มีตัวเลขและตัวอักษรผสมกัน
  Result := Length(Pwd) >= 8;
end;

function TRegistrationForm.ValidateForm: Boolean;
var
  ErrorMsg: string;
begin
  Result := False;
  ClearErrors;
  ErrorMsg := '';
  
  // ตรวจสอบชื่อ
  if Trim(EditFirstName.Text) = '' then
  begin
    ShowError(EditFirstName, 'กรุณากรอกชื่อ');
    EditFirstName.SetFocus;
    Exit;
  end;
  
  // ตรวจสอบนามสกุล
  if Trim(EditLastName.Text) = '' then
  begin
    ShowError(EditLastName, 'กรุณากรอกนามสกุล');
    EditLastName.SetFocus;
    Exit;
  end;
  
  // ตรวจสอบอีเมล
  if Trim(EditEmail.Text) = '' then
  begin
    ShowError(EditEmail, 'กรุณากรอกอีเมล');
    EditEmail.SetFocus;
    Exit;
  end;
  
  if not ValidateEmail(EditEmail.Text) then
  begin
    ShowError(EditEmail, 'รูปแบบอีเมลไม่ถูกต้อง');
    EditEmail.SetFocus;
    Exit;
  end;
  
  // ตรวจสอบเบอร์โทร
  if Length(EditPhone.Text) < 9 then
  begin
    ShowError(EditPhone, 'กรุณากรอกเบอร์โทรศัพท์ให้ถูกต้อง');
    EditPhone.SetFocus;
    Exit;
  end;
  
  // ตรวจสอบรหัสผ่าน
  if not ValidatePassword(EditPassword.Text) then
  begin
    ShowError(EditPassword, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร');
    EditPassword.SetFocus;
    Exit;
  end;
  
  // ตรวจสอบรหัสผ่านยืนยัน
  if EditPassword.Text <> EditConfirmPwd.Text then
  begin
    ShowError(EditConfirmPwd, 'รหัสผ่านไม่ตรงกัน');
    EditConfirmPwd.SetFocus;
    Exit;
  end;
  
  // ตรวจสอบว่ายอมรับเงื่อนไข
  if not CheckBoxTerms.Checked then
  begin
    LabelStatus.Caption := 'กรุณายอมรับเงื่อนไขการใช้งาน';
    Exit;
  end;
  
  Result := True;
end;

procedure TRegistrationForm.ShowError(Control: TWinControl; const Msg: string);
begin
  if Control is TEdit then
    TEdit(Control).Color := $00CCFFFF; // สีแดงอ่อน
  LabelStatus.Caption := '* ' + Msg;
  LabelStatus.Font.Color := clRed;
end;

procedure TRegistrationForm.ClearErrors;
var
  i: Integer;
begin
  LabelStatus.Caption := '';
  // Reset สีของ Edit ทั้งหมด
  for i := 0 to ComponentCount - 1 do
    if Components[i] is TEdit then
      TEdit(Components[i]).Color := clWhite;
end;

procedure TRegistrationForm.GetFormData(var RegData: TUserRegistration);
begin
  RegData.FirstName := Trim(EditFirstName.Text);
  RegData.LastName := Trim(EditLastName.Text);
  RegData.Email := Trim(EditEmail.Text);
  RegData.Phone := EditPhone.Text;
  RegData.BirthDate := DatePickerBirth.Date;
  RegData.Address := MemoAddress.Text;
  RegData.City := EditCity.Text;
  RegData.Country := ComboCountry.Text;
  RegData.Password := EditPassword.Text;
  RegData.AcceptTerms := CheckBoxTerms.Checked;
  
  if RadioMale.Checked then RegData.Gender := 'ชาย'
  else if RadioFemale.Checked then RegData.Gender := 'หญิง'
  else RegData.Gender := 'อื่นๆ';
end;

procedure TRegistrationForm.SaveRegistration(const RegData: TUserRegistration);
begin
  // ในระบบจริงจะบันทึกลงฐานข้อมูล
  // นี่เป็นตัวอย่างแสดงข้อมูลที่บันทึก
  ShowMessage(
    'ลงทะเบียนสำเร็จ!' + #13#10 +
    'ชื่อ: ' + RegData.FirstName + ' ' + RegData.LastName + #13#10 +
    'อีเมล: ' + RegData.Email + #13#10 +
    'โทร: ' + RegData.Phone + #13#10 +
    'เพศ: ' + RegData.Gender + #13#10 +
    'ประเทศ: ' + RegData.Country
  );
end;

procedure TRegistrationForm.ButtonRegisterClick(Sender: TObject);
var
  RegData: TUserRegistration;
begin
  if ValidateForm then
  begin
    GetFormData(RegData);
    
    if MessageDlg('ยืนยันการลงทะเบียน?' + #13#10 +
                  'ชื่อ: ' + RegData.FirstName + ' ' + RegData.LastName + #13#10 +
                  'อีเมล: ' + RegData.Email,
                  mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    begin
      try
        SaveRegistration(RegData);
        LabelStatus.Caption := 'ลงทะเบียนสำเร็จ!';
        LabelStatus.Font.Color := clGreen;
        ClearForm;
      except
        on E: Exception do
        begin
          LabelStatus.Caption := 'เกิดข้อผิดพลาด: ' + E.Message;
          LabelStatus.Font.Color := clRed;
        end;
      end;
    end;
  end;
end;

procedure TRegistrationForm.ButtonClearClick(Sender: TObject);
begin
  if MessageDlg('ต้องการล้างข้อมูลทั้งหมดหรือไม่?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    ClearForm;
end;

procedure TRegistrationForm.ClearForm;
var
  i: Integer;
begin
  // Clear Edit controls
  for i := 0 to ComponentCount - 1 do
    if Components[i] is TEdit then
      TEdit(Components[i]).Text := '';
  
  MemoAddress.Clear;
  ComboCountry.ItemIndex := 0;
  DatePickerBirth.Date := EncodeDate(1990, 1, 1);
  RadioMale.Checked := True;
  CheckBoxTerms.Checked := False;
  ButtonRegister.Enabled := False;
  ClearErrors;
  EditFirstName.SetFocus;
end;

procedure TRegistrationForm.ButtonCancelClick(Sender: TObject);
begin
  if MessageDlg('ต้องการยกเลิกการลงทะเบียนหรือไม่?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    Close;
end;

procedure TRegistrationForm.EditEmailExit(Sender: TObject);
begin
  if (EditEmail.Text <> '') and not ValidateEmail(EditEmail.Text) then
  begin
    EditEmail.Color := $00CCFFFF;
    LabelStatus.Caption := 'รูปแบบอีเมลไม่ถูกต้อง';
  end
  else
    EditEmail.Color := clWhite;
end;

procedure TRegistrationForm.EditPasswordExit(Sender: TObject);
begin
  if (EditPassword.Text <> '') and not ValidatePassword(EditPassword.Text) then
  begin
    EditPassword.Color := $00CCFFFF;
    LabelStatus.Caption := 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
  end
  else
    EditPassword.Color := clWhite;
end;

procedure TRegistrationForm.CheckBoxTermsClick(Sender: TObject);
begin
  ButtonRegister.Enabled := CheckBoxTerms.Checked;
end;

procedure TRegistrationForm.LabelTermsLinkClick(Sender: TObject);
begin
  // แสดงเงื่อนไขการใช้งาน
  ShowMessage('เงื่อนไขการใช้งาน:' + #13#10 +
    '1. ผู้ใช้ต้องให้ข้อมูลที่ถูกต้องและเป็นความจริง' + #13#10 +
    '2. ห้ามใช้งานในทางผิดกฎหมาย' + #13#10 +
    '3. บริษัทขอสงวนสิทธิ์ในการยกเลิกบัญชีที่ละเมิดเงื่อนไข');
end;

end.
```

---

## 16.16 โปรแกรมตัวอย่าง: Survey Form

```pascal
unit SurveyForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls;

type
  TSurveyForm = class(TForm)
    PageControlSurvey: TPageControl;
    TabPage1: TTabSheet;
    TabPage2: TTabSheet;
    TabPage3: TTabSheet;
    TabPage4: TTabSheet;  // Summary
    
    // Page 1: Personal Info
    GroupBoxPersonal: TGroupBox;
    EditSurveyName: TEdit;
    EditSurveyAge: TEdit;
    ComboEducation: TComboBox;
    RadioSingle: TRadioButton;
    RadioMarried: TRadioButton;
    
    // Page 2: Product Experience
    GroupBoxProduct: TGroupBox;
    LabelQuestion1: TLabel;
    RadioGroup1: array[1..5] of TRadioButton;
    TrackBarSatisfaction: TTrackBar;
    LabelSatisfactionValue: TLabel;
    MemoFeedback: TMemo;
    
    // Page 3: Recommendations
    CheckBoxFeature1: TCheckBox;
    CheckBoxFeature2: TCheckBox;
    CheckBoxFeature3: TCheckBox;
    CheckBoxFeature4: TCheckBox;
    MemoSuggestion: TMemo;
    
    // Navigation
    ButtonPrev: TButton;
    ButtonNext: TButton;
    ButtonSubmit: TButton;
    ProgressBarSurvey: TProgressBar;
    LabelPageInfo: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure ButtonNextClick(Sender: TObject);
    procedure ButtonPrevClick(Sender: TObject);
    procedure ButtonSubmitClick(Sender: TObject);
    procedure TrackBarSatisfactionChange(Sender: TObject);
    
  private
    FCurrentPage: Integer;
    FTotalPages: Integer;
    
    procedure NavigateToPage(PageIndex: Integer);
    procedure UpdateNavigationButtons;
    procedure UpdateProgress;
    function ValidateCurrentPage: Boolean;
    procedure ShowSummary;
    function GetSatisfactionText(Value: Integer): string;
    procedure SaveSurveyData;
  end;

implementation

{$R *.lfm}

procedure TSurveyForm.FormCreate(Sender: TObject);
begin
  Caption := 'แบบสำรวจความพึงพอใจ';
  Width := 550;
  Height := 500;
  Position := poScreenCenter;
  
  FTotalPages := 3; // 3 หน้าถามคำถาม + 1 หน้าสรุป
  FCurrentPage := 0;
  
  // กำหนดชื่อ Tab
  TabPage1.Caption := 'ข้อมูลส่วนตัว';
  TabPage2.Caption := 'ประสบการณ์การใช้งาน';
  TabPage3.Caption := 'ข้อเสนอแนะ';
  TabPage4.Caption := 'สรุป';
  
  // ซ่อน Tab Header (นำทางด้วยปุ่มแทน)
  PageControlSurvey.ShowTabs := False;
  
  // Setup Progress Bar
  ProgressBarSurvey.Min := 0;
  ProgressBarSurvey.Max := FTotalPages;
  
  // Setup Education Combo
  ComboEducation.Items.AddStrings([
    'มัธยมศึกษา',
    'ปวช./ปวส.',
    'ปริญญาตรี',
    'ปริญญาโท',
    'ปริญญาเอก',
    'อื่นๆ'
  ]);
  ComboEducation.ItemIndex := 2;
  
  // Setup Satisfaction TrackBar
  TrackBarSatisfaction.Min := 1;
  TrackBarSatisfaction.Max := 10;
  TrackBarSatisfaction.Position := 5;
  TrackBarSatisfactionChange(nil);
  
  // Feature Checkboxes
  CheckBoxFeature1.Caption := 'ง่ายต่อการใช้งาน';
  CheckBoxFeature2.Caption := 'ราคาเหมาะสม';
  CheckBoxFeature3.Caption := 'รองรับภาษาไทย';
  CheckBoxFeature4.Caption := 'มีการ Update บ่อย';
  
  NavigateToPage(0);
end;

procedure TSurveyForm.NavigateToPage(PageIndex: Integer);
begin
  FCurrentPage := PageIndex;
  PageControlSurvey.ActivePageIndex := PageIndex;
  UpdateNavigationButtons;
  UpdateProgress;
  LabelPageInfo.Caption := 'หน้า ' + IntToStr(FCurrentPage + 1) + ' จาก ' + IntToStr(FTotalPages);
end;

procedure TSurveyForm.UpdateNavigationButtons;
begin
  ButtonPrev.Enabled := FCurrentPage > 0;
  ButtonNext.Visible := FCurrentPage < FTotalPages - 1;
  ButtonSubmit.Visible := FCurrentPage = FTotalPages - 1;
end;

procedure TSurveyForm.UpdateProgress;
begin
  ProgressBarSurvey.Position := FCurrentPage + 1;
end;

function TSurveyForm.ValidateCurrentPage: Boolean;
begin
  Result := True;
  case FCurrentPage of
    0: // Personal Info
    begin
      if Trim(EditSurveyName.Text) = '' then
      begin
        ShowMessage('กรุณากรอกชื่อ-นามสกุล');
        EditSurveyName.SetFocus;
        Result := False;
      end
      else if Trim(EditSurveyAge.Text) = '' then
      begin
        ShowMessage('กรุณากรอกอายุ');
        EditSurveyAge.SetFocus;
        Result := False;
      end;
    end;
  end;
end;

procedure TSurveyForm.ButtonNextClick(Sender: TObject);
begin
  if ValidateCurrentPage then
  begin
    if FCurrentPage < FTotalPages - 1 then
      NavigateToPage(FCurrentPage + 1)
    else
      ShowSummary;
  end;
end;

procedure TSurveyForm.ButtonPrevClick(Sender: TObject);
begin
  if FCurrentPage > 0 then
    NavigateToPage(FCurrentPage - 1);
end;

procedure TSurveyForm.ButtonSubmitClick(Sender: TObject);
begin
  if MessageDlg('ยืนยันการส่งแบบสำรวจ?', mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    SaveSurveyData;
    ShowMessage('ขอบคุณสำหรับการตอบแบบสำรวจ!');
    Close;
  end;
end;

procedure TSurveyForm.TrackBarSatisfactionChange(Sender: TObject);
begin
  LabelSatisfactionValue.Caption :=
    IntToStr(TrackBarSatisfaction.Position) + '/10 - ' +
    GetSatisfactionText(TrackBarSatisfaction.Position);
end;

function TSurveyForm.GetSatisfactionText(Value: Integer): string;
begin
  case Value of
    1..2: Result := 'ไม่พึงพอใจอย่างมาก';
    3..4: Result := 'ไม่พึงพอใจ';
    5..6: Result := 'พอใช้';
    7..8: Result := 'พึงพอใจ';
    9..10: Result := 'พึงพอใจอย่างมาก';
    else Result := '';
  end;
end;

procedure TSurveyForm.ShowSummary;
var
  features: TStringList;
begin
  // แสดงหน้าสรุป
  NavigateToPage(3); // TabPage4
  
  // อัปเดต Summary Page
  // (ในโปรแกรมจริงจะมี Labels ใน TabPage4)
  features := TStringList.Create;
  try
    if CheckBoxFeature1.Checked then features.Add(CheckBoxFeature1.Caption);
    if CheckBoxFeature2.Checked then features.Add(CheckBoxFeature2.Caption);
    if CheckBoxFeature3.Checked then features.Add(CheckBoxFeature3.Caption);
    if CheckBoxFeature4.Checked then features.Add(CheckBoxFeature4.Caption);
    
    ShowMessage(
      'สรุปการตอบแบบสำรวจ:' + #13#10 +
      'ชื่อ: ' + EditSurveyName.Text + #13#10 +
      'อายุ: ' + EditSurveyAge.Text + #13#10 +
      'ความพึงพอใจ: ' + IntToStr(TrackBarSatisfaction.Position) + '/10' + #13#10 +
      'ฟีเจอร์ที่ชอบ: ' + features.CommaText
    );
  finally
    features.Free;
  end;
end;

procedure TSurveyForm.SaveSurveyData;
var
  SurveyFile: TStringList;
begin
  SurveyFile := TStringList.Create;
  try
    SurveyFile.Add('=== แบบสำรวจความพึงพอใจ ===');
    SurveyFile.Add('วันที่: ' + FormatDateTime('dd/mm/yyyy hh:nn:ss', Now));
    SurveyFile.Add('ชื่อ: ' + EditSurveyName.Text);
    SurveyFile.Add('อายุ: ' + EditSurveyAge.Text);
    SurveyFile.Add('การศึกษา: ' + ComboEducation.Text);
    SurveyFile.Add('สถานภาพ: ' + IfThen(RadioMarried.Checked, 'สมรส', 'โสด'));
    SurveyFile.Add('ความพึงพอใจ: ' + IntToStr(TrackBarSatisfaction.Position) + '/10');
    SurveyFile.Add('ความคิดเห็น: ' + MemoFeedback.Text);
    SurveyFile.Add('ข้อเสนอแนะ: ' + MemoSuggestion.Text);
    
    SurveyFile.SaveToFile(
      ExtractFilePath(Application.ExeName) + 'survey_' +
      FormatDateTime('yyyymmdd_hhnnss', Now) + '.txt'
    );
  finally
    SurveyFile.Free;
  end;
end;

end.
```

---

## 16.17 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Form พื้นฐาน
สร้าง Form ที่มีลักษณะดังนี้:
- Title: "โปรแกรมคำนวณเงินเดือน"
- ขนาด 400x300 pixels
- ตำแหน่งกลางหน้าจอ
- BorderStyle: bsDialog
- มีปุ่ม "คำนวณ" และ "ล้างข้อมูล"

```pascal
// เฉลย
procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption := 'โปรแกรมคำนวณเงินเดือน';
  Width := 400;
  Height := 300;
  Position := poScreenCenter;
  BorderStyle := bsDialog;
  BorderIcons := [biSystemMenu];
end;
```

### แบบฝึกหัดที่ 2: TEdit Validation
สร้างฟอร์มรับข้อมูลบัตรประชาชน (13 หลัก) พร้อม Validation

```pascal
procedure TForm1.EditIDCardKeyPress(Sender: TObject; var Key: Char);
begin
  // รับเฉพาะตัวเลขและ Backspace
  if not (Key in ['0'..'9', #8]) then
    Key := #0;
  
  // จำกัด 13 หลัก
  if (Key <> #8) and (Length(EditIDCard.Text) >= 13) then
    Key := #0;
end;

procedure TForm1.EditIDCardExit(Sender: TObject);
begin
  if Length(EditIDCard.Text) <> 13 then
  begin
    EditIDCard.Color := clYellow;
    ShowMessage('บัตรประชาชนต้องมี 13 หลัก');
    EditIDCard.SetFocus;
  end
  else
    EditIDCard.Color := clWhite;
end;
```

### แบบฝึกหัดที่ 3: TListBox และ TComboBox
สร้างโปรแกรมจัดการรายการสินค้า:
- TEdit สำหรับกรอกชื่อสินค้า
- TButton เพิ่ม/ลบ/แก้ไข
- TListBox แสดงรายการ
- TComboBox เลือกหมวดหมู่

```pascal
procedure TForm1.ButtonAddClick(Sender: TObject);
begin
  if Trim(EditProductName.Text) = '' then
  begin
    ShowMessage('กรุณากรอกชื่อสินค้า');
    EditProductName.SetFocus;
    Exit;
  end;
  
  ListBoxProducts.Items.Add(
    '[' + ComboCategory.Text + '] ' + EditProductName.Text
  );
  EditProductName.Clear;
  EditProductName.SetFocus;
end;

procedure TForm1.ButtonDeleteClick(Sender: TObject);
begin
  if ListBoxProducts.ItemIndex < 0 then
  begin
    ShowMessage('กรุณาเลือกสินค้าที่ต้องการลบ');
    Exit;
  end;
  
  if MessageDlg('ลบ "' + ListBoxProducts.Items[ListBoxProducts.ItemIndex] + '" หรือไม่?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    ListBoxProducts.Items.Delete(ListBoxProducts.ItemIndex);
end;
```

### แบบฝึกหัดที่ 4: TProgressBar กับ Timer
สร้างโปรแกรม Countdown Timer พร้อม ProgressBar

```pascal
procedure TForm1.FormCreate(Sender: TObject);
begin
  ProgressBar1.Min := 0;
  ProgressBar1.Max := 60;
  ProgressBar1.Position := 60;
  Timer1.Interval := 1000; // 1 วินาที
  Timer1.Enabled := False;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  if ProgressBar1.Position > 0 then
  begin
    ProgressBar1.Position := ProgressBar1.Position - 1;
    LabelCountdown.Caption := IntToStr(ProgressBar1.Position) + ' วินาที';
  end
  else
  begin
    Timer1.Enabled := False;
    ShowMessage('หมดเวลา!');
  end;
end;
```

### แบบฝึกหัดที่ 5: TPageControl แบบ Wizard
สร้าง Wizard ติดตั้งโปรแกรม 4 ขั้นตอน:
- ขั้นตอนที่ 1: ยืนยันข้อตกลง
- ขั้นตอนที่ 2: เลือก Component
- ขั้นตอนที่ 3: เลือก Directory
- ขั้นตอนที่ 4: สรุปและติดตั้ง

```pascal
// แนวทางการพัฒนา
type
  TWizardStep = (wsLicense, wsComponents, wsDirectory, wsSummary);

procedure TInstallForm.GoToStep(Step: TWizardStep);
begin
  PageControl1.ActivePageIndex := Ord(Step);
  ButtonPrev.Enabled := Step > wsLicense;
  ButtonNext.Caption := IfThen(Step = wsSummary, 'ติดตั้ง', 'ถัดไป ›');
  LabelStep.Caption := 'ขั้นตอนที่ ' + IntToStr(Ord(Step) + 1) + ' จาก 4';
end;
```

### แบบฝึกหัดที่ 6: TRichEdit Text Formatter
สร้าง Mini Rich Text Editor ที่มีปุ่ม Bold, Italic, Underline, Color

```pascal
procedure TRichEditorForm.ToggleBold;
begin
  with RichEdit1.SelAttributes do
    Bold := not Bold;
end;

procedure TRichEditorForm.ChangeTextColor;
var
  ColorDialog: TColorDialog;
begin
  ColorDialog := TColorDialog.Create(Self);
  try
    ColorDialog.Color := RichEdit1.SelAttributes.Color;
    if ColorDialog.Execute then
      RichEdit1.SelAttributes.Color := ColorDialog.Color;
  finally
    ColorDialog.Free;
  end;
end;
```

### แบบฝึกหัดที่ 7: Anchors และ Layout
สร้างฟอร์มที่ปรับขนาดได้:
- Toolbar Panel ด้านบน (Anchor: Top, Left, Right)
- ListBox ด้านซ้าย (Anchor: Top, Left, Bottom)
- Memo หลัก (Anchor: ทุกด้าน)
- Status Panel ด้านล่าง (Anchor: Bottom, Left, Right)

```pascal
procedure TLayoutForm.FormCreate(Sender: TObject);
begin
  PanelToolbar.Align := alTop;
  PanelToolbar.Height := 35;
  
  ListBox1.Align := alLeft;
  ListBox1.Width := 150;
  
  Splitter1.Left := ListBox1.Width;
  Splitter1.Align := alLeft;
  
  Memo1.Align := alClient;
  
  PanelStatus.Align := alBottom;
  PanelStatus.Height := 24;
end;
```

### แบบฝึกหัดที่ 8: Dynamic Controls
สร้างโปรแกรมที่สร้าง Controls แบบ Dynamic ตามจำนวนที่กำหนด

```pascal
procedure TDynamicForm.CreateInputFields(Count: Integer);
var
  i: Integer;
  lbl: TLabel;
  edt: TEdit;
begin
  // ลบ Controls เก่า
  while ControlCount > 0 do
    Controls[0].Free;
    
  for i := 1 to Count do
  begin
    lbl := TLabel.Create(Self);
    lbl.Parent := Self;
    lbl.Caption := 'ข้อมูลที่ ' + IntToStr(i) + ':';
    lbl.Left := 20;
    lbl.Top := 20 + (i - 1) * 35;
    
    edt := TEdit.Create(Self);
    edt.Parent := Self;
    edt.Left := 120;
    edt.Top := 20 + (i - 1) * 35;
    edt.Width := 200;
    edt.Name := 'Edit' + IntToStr(i);
  end;
end;
```

### แบบฝึกหัดที่ 9: CheckBox Group
สร้างโปรแกรมเลือกวิชาเรียน โดยแสดงรายวิชาที่เลือกใน Memo

```pascal
procedure TSubjectForm.UpdateSelectedSubjects;
var
  i: Integer;
  selected: TStringList;
begin
  selected := TStringList.Create;
  try
    for i := 0 to GroupBoxSubjects.ControlCount - 1 do
      if (GroupBoxSubjects.Controls[i] is TCheckBox) and
         TCheckBox(GroupBoxSubjects.Controls[i]).Checked then
        selected.Add(TCheckBox(GroupBoxSubjects.Controls[i]).Caption);
    
    MemoSelected.Lines.Clear;
    MemoSelected.Lines.Add('วิชาที่เลือก (' + IntToStr(selected.Count) + ' วิชา):');
    MemoSelected.Lines.AddStrings(selected);
  finally
    selected.Free;
  end;
end;
```

### แบบฝึกหัดที่ 10: Form Validation ครบถ้วน
สร้างฟอร์มสมัครงานพร้อม Validation ครบทุก Field

```pascal
type
  TJobApplicationForm = class(TForm)
  private
    function ValidateAll: Boolean;
    procedure HighlightError(Control: TEdit; const Msg: string);
  end;

function TJobApplicationForm.ValidateAll: Boolean;
var
  hasError: Boolean;
begin
  hasError := False;
  
  // Reset colors
  EditName.Color := clWhite;
  EditEmail.Color := clWhite;
  EditPhone.Color := clWhite;
  
  if Trim(EditName.Text) = '' then
  begin
    HighlightError(EditName, 'กรุณากรอกชื่อ-นามสกุล');
    hasError := True;
  end;
  
  // ... ตรวจสอบ Field อื่นๆ
  
  Result := not hasError;
end;
```

### แบบฝึกหัดที่ 11: Custom Scroll
สร้างโปรแกรมแสดงรูปภาพขนาดใหญ่พร้อม ScrollBars แบบ Custom

```pascal
procedure TImageViewForm.SetupCustomScroll;
begin
  ScrollBarH.Min := 0;
  ScrollBarH.Max := Image1.Picture.Width - ScrollBox1.ClientWidth;
  ScrollBarH.LargeChange := ScrollBox1.ClientWidth div 2;
  
  ScrollBarV.Min := 0;
  ScrollBarV.Max := Image1.Picture.Height - ScrollBox1.ClientHeight;
  ScrollBarV.LargeChange := ScrollBox1.ClientHeight div 2;
end;

procedure TImageViewForm.ScrollBarHChange(Sender: TObject);
begin
  Image1.Left := -ScrollBarH.Position;
end;

procedure TImageViewForm.ScrollBarVChange(Sender: TObject);
begin
  Image1.Top := -ScrollBarV.Position;
end;
```

### แบบฝึกหัดที่ 12: TabControl แบบ Dynamic
สร้างโปรแกรม Multi-Document Interface อย่างง่าย

```pascal
procedure TTabEditorForm.NewDocument;
var
  newTab: TTabSheet;
  newMemo: TMemo;
begin
  newTab := TTabSheet.Create(PageControl1);
  newTab.PageControl := PageControl1;
  newTab.Caption := 'เอกสาร ' + IntToStr(FDocumentCount + 1);
  
  newMemo := TMemo.Create(newTab);
  newMemo.Parent := newTab;
  newMemo.Align := alClient;
  newMemo.ScrollBars := ssBoth;
  newMemo.Font.Name := 'Courier New';
  newMemo.Font.Size := 11;
  
  PageControl1.ActivePage := newTab;
  Inc(FDocumentCount);
  newMemo.SetFocus;
end;

procedure TTabEditorForm.CloseCurrentTab;
var
  currentTab: TTabSheet;
begin
  if PageControl1.PageCount = 0 then Exit;
  
  currentTab := PageControl1.ActivePage;
  if MessageDlg('ต้องการปิดเอกสาร "' + currentTab.Caption + '"?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    currentTab.Free;
  end;
end;
```

### แบบฝึกหัดที่ 13: Form Communication
สร้าง 2 Forms ที่สามารถส่งข้อมูลหากันได้

```pascal
// Form1
procedure TMainForm.ButtonOpenForm2Click(Sender: TObject);
begin
  Form2 := TInputForm.Create(Self);
  try
    Form2.SetInitialData(Edit1.Text);
    if Form2.ShowModal = mrOk then
    begin
      Edit1.Text := Form2.GetResult;
      Memo1.Lines.Add('ได้รับข้อมูล: ' + Form2.GetResult);
    end;
  finally
    Form2.Free;
  end;
end;

// Form2
type
  TInputForm = class(TForm)
  private
    FResult: string;
  public
    procedure SetInitialData(const Data: string);
    function GetResult: string;
  end;

procedure TInputForm.SetInitialData(const Data: string);
begin
  EditInput.Text := Data;
end;

function TInputForm.GetResult: string;
begin
  Result := FResult;
end;

procedure TInputForm.ButtonOKClick(Sender: TObject);
begin
  FResult := EditInput.Text;
  ModalResult := mrOk;
end;
```

### แบบฝึกหัดที่ 14: Controls ที่ซับซ้อน
สร้าง Color Picker อย่างง่ายโดยใช้ TrackBar สำหรับ R, G, B

```pascal
procedure TColorPickerForm.UpdateColor;
var
  r, g, b: Byte;
begin
  r := TrackBarR.Position;
  g := TrackBarG.Position;
  b := TrackBarB.Position;
  
  PanelPreview.Color := RGB(r, g, b);
  
  LabelRValue.Caption := IntToStr(r);
  LabelGValue.Caption := IntToStr(g);
  LabelBValue.Caption := IntToStr(b);
  
  LabelHexColor.Caption := '#' +
    IntToHex(r, 2) + IntToHex(g, 2) + IntToHex(b, 2);
end;

procedure TColorPickerForm.TrackBarChange(Sender: TObject);
begin
  UpdateColor;
end;
```

### แบบฝึกหัดที่ 15: สร้าง Form Calculator
สร้างเครื่องคิดเลขที่มี:
- Display แสดงผล
- ปุ่ม 0-9, +, -, *, /
- ปุ่ม C (Clear), CE (Clear Entry), =

```pascal
unit Calculator;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls;

type
  TCalcForm = class(TForm)
    EditDisplay: TEdit;
    PanelButtons: TPanel;
    procedure FormCreate(Sender: TObject);
    procedure ButtonClick(Sender: TObject);
  private
    FFirstNumber: Double;
    FOperator: Char;
    FNewInput: Boolean;
    
    procedure SetupButtons;
    procedure AppendDigit(const Digit: string);
    procedure SetOperator(Op: Char);
    procedure Calculate;
    procedure Clear;
  end;

implementation

{$R *.lfm}

procedure TCalcForm.FormCreate(Sender: TObject);
begin
  Caption := 'เครื่องคิดเลข';
  Width := 240;
  Height := 320;
  Position := poScreenCenter;
  
  EditDisplay.ReadOnly := True;
  EditDisplay.Alignment := taRightJustify;
  EditDisplay.Font.Size := 16;
  EditDisplay.Text := '0';
  
  FFirstNumber := 0;
  FOperator := #0;
  FNewInput := True;
  
  SetupButtons;
end;

procedure TCalcForm.SetupButtons;
const
  ButtonCaptions: array[0..18] of string = (
    'C', 'CE', '±', '÷',
    '7', '8', '9', '×',
    '4', '5', '6', '-',
    '1', '2', '3', '+',
    '0', '.', '='
  );
var
  i, col, row: Integer;
  btn: TButton;
  btnWidth, btnHeight: Integer;
begin
  btnWidth := 50;
  btnHeight := 45;
  
  for i := 0 to High(ButtonCaptions) do
  begin
    btn := TButton.Create(Self);
    btn.Parent := PanelButtons;
    btn.Caption := ButtonCaptions[i];
    btn.Tag := i;
    btn.OnClick := ButtonClick;
    
    if i < 16 then
    begin
      col := i mod 4;
      row := i div 4;
      btn.Left := col * btnWidth + 5;
      btn.Top := row * btnHeight + 5;
      btn.Width := btnWidth;
      btn.Height := btnHeight;
    end
    else
    begin
      // ปุ่ม 0 กว้าง 2 ช่อง, . และ = ปกติ
      case i of
        16: begin // 0
          btn.Left := 5;
          btn.Top := 4 * btnHeight + 5;
          btn.Width := btnWidth * 2;
          btn.Height := btnHeight;
        end;
        17: begin // .
          btn.Left := 2 * btnWidth + 5;
          btn.Top := 4 * btnHeight + 5;
          btn.Width := btnWidth;
          btn.Height := btnHeight;
        end;
        18: begin // =
          btn.Left := 3 * btnWidth + 5;
          btn.Top := 4 * btnHeight + 5;
          btn.Width := btnWidth;
          btn.Height := btnHeight;
          btn.Font.Bold := True;
          btn.Color := clLime;
        end;
      end;
    end;
  end;
end;

procedure TCalcForm.AppendDigit(const Digit: string);
begin
  if FNewInput then
  begin
    EditDisplay.Text := Digit;
    FNewInput := False;
  end
  else
  begin
    if (EditDisplay.Text = '0') and (Digit <> '.') then
      EditDisplay.Text := Digit
    else if (Digit = '.') and (Pos('.', EditDisplay.Text) > 0) then
      // ไม่เพิ่มจุดถ้ามีอยู่แล้ว
    else
      EditDisplay.Text := EditDisplay.Text + Digit;
  end;
end;

procedure TCalcForm.SetOperator(Op: Char);
begin
  FFirstNumber := StrToFloatDef(EditDisplay.Text, 0);
  FOperator := Op;
  FNewInput := True;
end;

procedure TCalcForm.Calculate;
var
  secondNumber, result: Double;
begin
  secondNumber := StrToFloatDef(EditDisplay.Text, 0);
  result := 0;
  
  case FOperator of
    '+': result := FFirstNumber + secondNumber;
    '-': result := FFirstNumber - secondNumber;
    '*': result := FFirstNumber * secondNumber;
    '/': 
      if secondNumber <> 0 then
        result := FFirstNumber / secondNumber
      else
      begin
        ShowMessage('ไม่สามารถหารด้วย 0 ได้');
        Clear;
        Exit;
      end;
  else
    result := secondNumber;
  end;
  
  // แสดงผลลัพธ์
  if Frac(result) = 0 then
    EditDisplay.Text := IntToStr(Round(result))
  else
    EditDisplay.Text := FloatToStrF(result, ffFixed, 15, 8);
    
  FFirstNumber := result;
  FNewInput := True;
end;

procedure TCalcForm.Clear;
begin
  EditDisplay.Text := '0';
  FFirstNumber := 0;
  FOperator := #0;
  FNewInput := True;
end;

procedure TCalcForm.ButtonClick(Sender: TObject);
var
  btn: TButton;
  caption: string;
begin
  btn := TButton(Sender);
  caption := btn.Caption;
  
  case caption[1] of
    '0'..'9': AppendDigit(caption);
    '.': AppendDigit('.');
    '+': SetOperator('+');
    '-': SetOperator('-');
    '×': SetOperator('*');
    '÷': SetOperator('/');
    '=': Calculate;
    'C': Clear;
    'E': // CE - Clear Entry
    begin
      EditDisplay.Text := '0';
      FNewInput := True;
    end;
    '±': // Toggle sign
    begin
      if EditDisplay.Text[1] = '-' then
        EditDisplay.Text := Copy(EditDisplay.Text, 2, MaxInt)
      else
        EditDisplay.Text := '-' + EditDisplay.Text;
    end;
  end;
end;

end.
```

---

## สรุปบทที่ 16

ในบทนี้เราได้เรียนรู้:

1. **TForm** - Form หลักของ Application รวมถึง Properties, Methods และ Events
2. **TButton, TBitBtn, TSpeedButton** - ปุ่มต่างๆ สำหรับการกระทำ
3. **TLabel, TStaticText** - การแสดงข้อความ
4. **TEdit, TMemo, TRichEdit** - การรับและแสดงข้อความ
5. **TCheckBox, TRadioButton, TGroupBox** - การเลือกตัวเลือก
6. **TListBox, TComboBox** - การเลือกจากรายการ
7. **TScrollBar, TTrackBar, TProgressBar** - Controls สำหรับค่าตัวเลข
8. **TPanel, TScrollBox** - Container Controls
9. **TImage** - การแสดงรูปภาพ
10. **TPageControl** - การแบ่งหน้าด้วย Tabs
11. **Anchors และ Constraints** - การจัดวาง Controls
12. **Z-Order** - ลำดับการซ้อนทับ
13. **Focus Management** - การจัดการ Focus
14. **Validation** - การตรวจสอบข้อมูล

บทถัดไปจะเรียนรู้เกี่ยวกับ **Events และ Event Handling** อย่างละเอียด
