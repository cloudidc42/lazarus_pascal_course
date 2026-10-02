# Part 18 - Menus และ Dialogs

## บทนำ

Menus และ Dialogs เป็นส่วนสำคัญของ GUI Application ที่ช่วยให้ผู้ใช้สามารถเข้าถึงฟังก์ชันต่างๆ ได้อย่างมีระเบียบ ในบทนี้เราจะเรียนรู้การสร้างและใช้งาน Menus และ Dialogs ประเภทต่างๆ

---

## 18.1 TMainMenu

### 18.1.1 การสร้าง MainMenu

```pascal
// TMainMenu เป็น Menu Bar ที่ด้านบนของ Form
// สร้างผ่าน Object Inspector หรือ Code

unit MainMenuExample;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Menus, Dialogs, StdCtrls;

type
  TMainMenuForm = class(TForm)
    MainMenu1: TMainMenu;
    
    // File Menu
    MenuFile: TMenuItem;
    MenuFileNew: TMenuItem;
    MenuFileOpen: TMenuItem;
    MenuFileSave: TMenuItem;
    MenuFileSaveAs: TMenuItem;
    MenuFileSep1: TMenuItem;  // Separator
    MenuFileRecentFiles: TMenuItem;
    MenuFileSep2: TMenuItem;
    MenuFilePrint: TMenuItem;
    MenuFileSep3: TMenuItem;
    MenuFileExit: TMenuItem;
    
    // Edit Menu
    MenuEdit: TMenuItem;
    MenuEditUndo: TMenuItem;
    MenuEditRedo: TMenuItem;
    MenuEditSep1: TMenuItem;
    MenuEditCut: TMenuItem;
    MenuEditCopy: TMenuItem;
    MenuEditPaste: TMenuItem;
    MenuEditDelete: TMenuItem;
    MenuEditSep2: TMenuItem;
    MenuEditSelectAll: TMenuItem;
    MenuEditFind: TMenuItem;
    MenuEditReplace: TMenuItem;
    
    // View Menu
    MenuView: TMenuItem;
    MenuViewToolbar: TMenuItem;
    MenuViewStatusBar: TMenuItem;
    MenuViewSep1: TMenuItem;
    MenuViewZoomIn: TMenuItem;
    MenuViewZoomOut: TMenuItem;
    MenuViewZoomReset: TMenuItem;
    
    // Help Menu
    MenuHelp: TMenuItem;
    MenuHelpContents: TMenuItem;
    MenuHelpAbout: TMenuItem;
    
    procedure FormCreate(Sender: TObject);
    procedure MenuFileNewClick(Sender: TObject);
    procedure MenuFileOpenClick(Sender: TObject);
    procedure MenuFileSaveClick(Sender: TObject);
    procedure MenuFileExitClick(Sender: TObject);
    procedure MenuEditUndoClick(Sender: TObject);
    procedure MenuViewToolbarClick(Sender: TObject);
    
  private
    procedure SetupMenus;
    procedure SetupFileMenu;
    procedure SetupEditMenu;
    procedure SetupViewMenu;
    procedure SetupHelpMenu;
    procedure UpdateMenuState;
  end;

implementation

procedure TMainMenuForm.SetupFileMenu;
begin
  // File Menu
  MenuFile.Caption := '&ไฟล์';
  
  MenuFileNew.Caption := '&ใหม่';
  MenuFileNew.ShortCut := TextToShortCut('Ctrl+N');
  MenuFileNew.ImageIndex := 0;
  
  MenuFileOpen.Caption := '&เปิด...';
  MenuFileOpen.ShortCut := TextToShortCut('Ctrl+O');
  MenuFileOpen.ImageIndex := 1;
  
  MenuFileSave.Caption := '&บันทึก';
  MenuFileSave.ShortCut := TextToShortCut('Ctrl+S');
  MenuFileSave.ImageIndex := 2;
  MenuFileSave.Enabled := False; // Disabled ตอนเริ่มต้น
  
  MenuFileSaveAs.Caption := 'บันทึก&เป็น...';
  MenuFileSaveAs.ShortCut := TextToShortCut('Ctrl+Shift+S');
  
  MenuFileSep1.Caption := '-'; // Separator
  
  MenuFileRecentFiles.Caption := 'ไฟล์&ล่าสุด';
  // จะเพิ่ม Submenu แบบ Dynamic
  
  MenuFileSep2.Caption := '-';
  
  MenuFilePrint.Caption := '&พิมพ์...';
  MenuFilePrint.ShortCut := TextToShortCut('Ctrl+P');
  
  MenuFileSep3.Caption := '-';
  
  MenuFileExit.Caption := 'ออก&จากโปรแกรม';
  MenuFileExit.ShortCut := TextToShortCut('Alt+F4');
end;

procedure TMainMenuForm.SetupEditMenu;
begin
  MenuEdit.Caption := '&แก้ไข';
  
  MenuEditUndo.Caption := '&ยกเลิก';
  MenuEditUndo.ShortCut := TextToShortCut('Ctrl+Z');
  MenuEditUndo.Enabled := False;
  
  MenuEditRedo.Caption := '&ทำซ้ำ';
  MenuEditRedo.ShortCut := TextToShortCut('Ctrl+Y');
  MenuEditRedo.Enabled := False;
  
  MenuEditSep1.Caption := '-';
  
  MenuEditCut.Caption := '&ตัด';
  MenuEditCut.ShortCut := TextToShortCut('Ctrl+X');
  
  MenuEditCopy.Caption := '&คัดลอก';
  MenuEditCopy.ShortCut := TextToShortCut('Ctrl+C');
  
  MenuEditPaste.Caption := '&วาง';
  MenuEditPaste.ShortCut := TextToShortCut('Ctrl+V');
  
  MenuEditDelete.Caption := '&ลบ';
  MenuEditDelete.ShortCut := TextToShortCut('Del');
  
  MenuEditSep2.Caption := '-';
  
  MenuEditSelectAll.Caption := 'เลือก&ทั้งหมด';
  MenuEditSelectAll.ShortCut := TextToShortCut('Ctrl+A');
  
  MenuEditFind.Caption := '&ค้นหา...';
  MenuEditFind.ShortCut := TextToShortCut('Ctrl+F');
  
  MenuEditReplace.Caption := 'ค้นหา&และแทนที่...';
  MenuEditReplace.ShortCut := TextToShortCut('Ctrl+H');
end;

procedure TMainMenuForm.UpdateMenuState;
var
  hasText: Boolean;
  hasSelection: Boolean;
begin
  hasText := (ActiveControl is TMemo) and 
             (TMemo(ActiveControl).Lines.Count > 0);
  hasSelection := (ActiveControl is TMemo) and 
                  (TMemo(ActiveControl).SelLength > 0);
  
  MenuFileSave.Enabled := hasText;
  MenuEditCut.Enabled := hasSelection;
  MenuEditCopy.Enabled := hasSelection;
  MenuEditDelete.Enabled := hasSelection;
end;
```

### 18.1.2 การสร้าง Menu ด้วย Code

```pascal
// สร้าง Menu Items ด้วย Code (Dynamic)
procedure TForm1.CreateMenuDynamically;
var
  fileMenu, editMenu: TMenuItem;
  newItem, openItem, saveItem, sep, exitItem: TMenuItem;
begin
  // สร้าง Main Menu
  MainMenu1 := TMainMenu.Create(Self);
  Self.Menu := MainMenu1;
  
  // สร้าง File Menu
  fileMenu := TMenuItem.Create(MainMenu1);
  fileMenu.Caption := '&ไฟล์';
  MainMenu1.Items.Add(fileMenu);
  
  // เพิ่มรายการใน File Menu
  newItem := TMenuItem.Create(fileMenu);
  newItem.Caption := '&ใหม่';
  newItem.ShortCut := TextToShortCut('Ctrl+N');
  newItem.OnClick := MenuNewClick;
  fileMenu.Add(newItem);
  
  openItem := TMenuItem.Create(fileMenu);
  openItem.Caption := '&เปิด...';
  openItem.ShortCut := TextToShortCut('Ctrl+O');
  openItem.OnClick := MenuOpenClick;
  fileMenu.Add(openItem);
  
  // Separator
  sep := TMenuItem.Create(fileMenu);
  sep.Caption := '-';
  fileMenu.Add(sep);
  
  exitItem := TMenuItem.Create(fileMenu);
  exitItem.Caption := 'ออกจากโปรแกรม';
  exitItem.OnClick := MenuExitClick;
  fileMenu.Add(exitItem);
  
  // Edit Menu
  editMenu := TMenuItem.Create(MainMenu1);
  editMenu.Caption := '&แก้ไข';
  MainMenu1.Items.Add(editMenu);
end;
```

---

## 18.2 TPopupMenu

### 18.2.1 การสร้าง PopupMenu

```pascal
// PopupMenu แสดงเมื่อคลิกขวา
procedure TForm1.SetupPopupMenu;
var
  popMenu: TPopupMenu;
  item: TMenuItem;
begin
  popMenu := TPopupMenu.Create(Self);
  
  // เพิ่มรายการ
  item := TMenuItem.Create(popMenu);
  item.Caption := 'คัดลอก';
  item.OnClick := CopyClick;
  popMenu.Items.Add(item);
  
  item := TMenuItem.Create(popMenu);
  item.Caption := 'วาง';
  item.OnClick := PasteClick;
  popMenu.Items.Add(item);
  
  item := TMenuItem.Create(popMenu);
  item.Caption := '-'; // Separator
  popMenu.Items.Add(item);
  
  item := TMenuItem.Create(popMenu);
  item.Caption := 'คุณสมบัติ...';
  item.OnClick := PropertiesClick;
  popMenu.Items.Add(item);
  
  // กำหนดให้ Control ใช้ PopupMenu นี้
  Memo1.PopupMenu := popMenu;
  ListBox1.PopupMenu := popMenu;
end;

// แสดง PopupMenu ด้วย Code
procedure TForm1.ShowContextMenu(X, Y: Integer);
begin
  PopupMenu1.Popup(X, Y);
end;

// Event OnPopup - เกิดก่อน PopupMenu แสดง
procedure TForm1.PopupMenu1Popup(Sender: TObject);
begin
  // อัปเดต State ของ Menu Items ก่อนแสดง
  MenuItemCopy.Enabled := (ActiveControl is TMemo) and 
                           (TMemo(ActiveControl).SelLength > 0);
  MenuItemPaste.Enabled := Clipboard.HasFormat(CF_TEXT);
  MenuItemDelete.Enabled := MenuItemCopy.Enabled;
end;
```

### 18.2.2 Context Menu ตาม Context

```pascal
// สร้าง Context Menu ที่แตกต่างกันตาม Context
procedure TForm1.ListBox1MouseDown(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  if Button = mbRight then
  begin
    // ตรวจสอบว่าคลิกที่รายการไหน
    ListBox1.ItemIndex := ListBox1.ItemAtPos(Point(X, Y), True);
    
    if ListBox1.ItemIndex >= 0 then
    begin
      // มีรายการถูกเลือก - แสดง Menu สำหรับรายการ
      PopupMenuItem.Popup(
        ListBox1.ClientToScreen(Point(X, Y)).X,
        ListBox1.ClientToScreen(Point(X, Y)).Y
      );
    end
    else
    begin
      // ไม่มีรายการถูกเลือก - แสดง Menu ทั่วไป
      PopupMenuGeneral.Popup(
        ListBox1.ClientToScreen(Point(X, Y)).X,
        ListBox1.ClientToScreen(Point(X, Y)).Y
      );
    end;
  end;
end;
```

---

## 18.3 Menu Items: Properties, Shortcuts, Icons

### 18.3.1 Properties ของ TMenuItem

```pascal
// Properties สำคัญของ TMenuItem
procedure TForm1.SetupMenuItem;
var
  item: TMenuItem;
begin
  item := TMenuItem.Create(Self);
  
  // Text
  item.Caption := '&บันทึก';    // & นำหน้าตัวอักษร = Underline + Alt Shortcut
  
  // Shortcut
  item.ShortCut := TextToShortCut('Ctrl+S');
  
  // State
  item.Enabled := True;         // เปิด/ปิดใช้งาน
  item.Visible := True;         // แสดง/ซ่อน
  item.Checked := False;        // แสดง Checkmark (สำหรับ Toggle)
  
  // RadioItem - เหมือน RadioButton ใน Menu Group
  item.RadioItem := False;
  item.GroupIndex := 0;         // กลุ่มของ RadioItems
  
  // ImageIndex - Index รูปภาพจาก ImageList
  item.ImageIndex := 2;         // กำหนด ImageList ใน Menu ก่อน
  
  // Tag - สำหรับเก็บข้อมูลเพิ่มเติม
  item.Tag := 100;
  
  // Break - แบ่ง Menu เป็นคอลัมน์
  item.Break := mbNone;         // mbNone, mbBarBreak, mbBreak
  
  // Default - แสดงตัวหนา (เป็น Default Action)
  item.Default := False;
  
  // Hint - Tooltip
  item.Hint := 'บันทึกไฟล์ (Ctrl+S)';
  
  // SubMenu - มี Sub Menu ไหม
  // ถ้าเพิ่ม Items ลงไป จะกลายเป็น Sub Menu อัตโนมัติ
end;
```

### 18.3.2 Icons ใน Menu

```pascal
// การใช้ Icons ใน Menu ต้องใช้ TImageList
procedure TForm1.SetupMenuIcons;
begin
  // กำหนด ImageList ให้ MainMenu
  MainMenu1.Images := ImageList1;
  
  // กำหนด ImageIndex ให้แต่ละ MenuItem
  MenuFileNew.ImageIndex := 0;    // Icon แรกใน ImageList
  MenuFileOpen.ImageIndex := 1;
  MenuFileSave.ImageIndex := 2;
  MenuEditCut.ImageIndex := 3;
  MenuEditCopy.ImageIndex := 4;
  MenuEditPaste.ImageIndex := 5;
end;

// โหลด Icons เข้า ImageList
procedure TForm1.LoadMenuIcons;
var
  bmp: TBitmap;
begin
  ImageList1.Width := 16;
  ImageList1.Height := 16;
  
  bmp := TBitmap.Create;
  try
    bmp.LoadFromFile('icons/new.bmp');
    ImageList1.Add(bmp, nil);
    
    bmp.LoadFromFile('icons/open.bmp');
    ImageList1.Add(bmp, nil);
    
    bmp.LoadFromFile('icons/save.bmp');
    ImageList1.Add(bmp, nil);
  finally
    bmp.Free;
  end;
end;
```

---

## 18.4 Dynamic Menus

### 18.4.1 เพิ่ม/ลบ Menu Items แบบ Dynamic

```pascal
// สร้าง Recent Files Menu แบบ Dynamic
procedure TForm1.UpdateRecentFilesMenu;
var
  i: Integer;
  item: TMenuItem;
begin
  // ลบรายการเก่าออก
  MenuFileRecentFiles.Clear;
  
  if FRecentFiles.Count = 0 then
  begin
    item := TMenuItem.Create(MenuFileRecentFiles);
    item.Caption := '(ไม่มีไฟล์ล่าสุด)';
    item.Enabled := False;
    MenuFileRecentFiles.Add(item);
  end
  else
  begin
    for i := 0 to Min(FRecentFiles.Count - 1, 9) do
    begin
      item := TMenuItem.Create(MenuFileRecentFiles);
      item.Caption := '&' + IntToStr(i + 1) + ' ' + 
                      ExtractFileName(FRecentFiles[i]);
      item.Hint := FRecentFiles[i];
      item.Tag := i;
      item.OnClick := RecentFileClick;
      MenuFileRecentFiles.Add(item);
    end;
    
    // Separator + Clear
    item := TMenuItem.Create(MenuFileRecentFiles);
    item.Caption := '-';
    MenuFileRecentFiles.Add(item);
    
    item := TMenuItem.Create(MenuFileRecentFiles);
    item.Caption := 'ล้างรายการ';
    item.OnClick := ClearRecentFilesClick;
    MenuFileRecentFiles.Add(item);
  end;
end;

procedure TForm1.RecentFileClick(Sender: TObject);
var
  fileName: string;
begin
  fileName := FRecentFiles[TMenuItem(Sender).Tag];
  if FileExists(fileName) then
    OpenFile(fileName)
  else
  begin
    ShowMessage('ไม่พบไฟล์: ' + fileName);
    FRecentFiles.Delete(TMenuItem(Sender).Tag);
    UpdateRecentFilesMenu;
  end;
end;
```

### 18.4.2 Menu ที่ Toggle State

```pascal
// Menu Item แบบ Toggle (Checkmark)
procedure TForm1.MenuViewToolbarClick(Sender: TObject);
begin
  MenuViewToolbar.Checked := not MenuViewToolbar.Checked;
  PanelToolbar.Visible := MenuViewToolbar.Checked;
end;

// Radio-style Menu Items
procedure TForm1.SetupViewModeMenu;
begin
  MenuViewModeNormal.GroupIndex := 1;
  MenuViewModeCompact.GroupIndex := 1;
  MenuViewModeExpanded.GroupIndex := 1;
  
  MenuViewModeNormal.RadioItem := True;
  MenuViewModeCompact.RadioItem := True;
  MenuViewModeExpanded.RadioItem := True;
  
  MenuViewModeNormal.Checked := True; // เลือก Normal ตอนเริ่ม
end;

procedure TForm1.MenuViewModeClick(Sender: TObject);
begin
  // ไม่ต้อง Set Checked เอง เพราะ RadioItem จัดการให้
  if Sender = MenuViewModeNormal then
    SetViewMode(vmNormal)
  else if Sender = MenuViewModeCompact then
    SetViewMode(vmCompact)
  else if Sender = MenuViewModeExpanded then
    SetViewMode(vmExpanded);
end;
```

---

## 18.5 Standard Dialogs

### 18.5.1 TOpenDialog

```pascal
// TOpenDialog - เปิดไฟล์
procedure TForm1.OpenFileDialog;
var
  OpenDlg: TOpenDialog;
begin
  OpenDlg := TOpenDialog.Create(Self);
  try
    // กำหนด Filter
    OpenDlg.Filter := 
      'Text Files|*.txt|' +
      'Pascal Files|*.pas;*.pp|' +
      'All Files|*.*';
    OpenDlg.FilterIndex := 1;  // Filter เริ่มต้น (เริ่มที่ 1)
    
    // กำหนด Title
    OpenDlg.Title := 'เปิดไฟล์';
    
    // Initial Directory
    OpenDlg.InitialDir := ExtractFilePath(Application.ExeName);
    
    // Default Extension
    OpenDlg.DefaultExt := 'txt';
    
    // Options
    OpenDlg.Options := [ofFileMustExist, ofPathMustExist];
    // ofAllowMultiSelect - เลือกหลายไฟล์
    // ofFileMustExist    - ไฟล์ต้องมีอยู่
    // ofPathMustExist    - Path ต้องมีอยู่
    // ofNoChangeDir      - ไม่เปลี่ยน Working Directory
    
    if OpenDlg.Execute then
    begin
      // เปิดไฟล์
      OpenFile(OpenDlg.FileName);
    end;
  finally
    OpenDlg.Free;
  end;
end;

// เปิดหลายไฟล์พร้อมกัน
procedure TForm1.OpenMultipleFiles;
var
  OpenDlg: TOpenDialog;
  i: Integer;
begin
  OpenDlg := TOpenDialog.Create(Self);
  try
    OpenDlg.Options := [ofAllowMultiSelect, ofFileMustExist];
    OpenDlg.Filter := 'รูปภาพ|*.jpg;*.png;*.bmp|ทั้งหมด|*.*';
    
    if OpenDlg.Execute then
    begin
      for i := 0 to OpenDlg.Files.Count - 1 do
        ProcessFile(OpenDlg.Files[i]);
    end;
  finally
    OpenDlg.Free;
  end;
end;
```

### 18.5.2 TSaveDialog

```pascal
// TSaveDialog - บันทึกไฟล์
procedure TForm1.SaveFileDialog;
var
  SaveDlg: TSaveDialog;
begin
  SaveDlg := TSaveDialog.Create(Self);
  try
    SaveDlg.Title := 'บันทึกไฟล์';
    SaveDlg.Filter := 
      'Text Files|*.txt|' +
      'Rich Text|*.rtf|' +
      'All Files|*.*';
    SaveDlg.DefaultExt := 'txt';
    SaveDlg.FileName := FCurrentFileName; // ชื่อไฟล์ปัจจุบัน
    
    // Options
    SaveDlg.Options := [ofOverwritePrompt]; // ถามก่อน Overwrite
    
    if SaveDlg.Execute then
    begin
      SaveFile(SaveDlg.FileName);
      FCurrentFileName := SaveDlg.FileName;
      Caption := ExtractFileName(FCurrentFileName) + ' - Text Editor';
    end;
  finally
    SaveDlg.Free;
  end;
end;
```

### 18.5.3 TFontDialog

```pascal
// TFontDialog - เลือก Font
procedure TForm1.ChooseFont;
var
  FontDlg: TFontDialog;
begin
  FontDlg := TFontDialog.Create(Self);
  try
    // กำหนด Font เริ่มต้น
    FontDlg.Font := Memo1.Font;
    
    // Options
    FontDlg.Options := [fdEffects, fdShowHelp];
    // fdEffects  - แสดง Strikeout, Underline
    // fdShowHelp - แสดงปุ่ม Help
    // fdFixedPitchOnly - เฉพาะ Fixed Pitch Fonts
    // fdTrueTypeOnly - เฉพาะ TrueType Fonts
    
    if FontDlg.Execute then
    begin
      // Apply font ที่เลือก
      if RichEdit1.SelLength > 0 then
        RichEdit1.SelAttributes.Assign(FontDlg.Font)
      else
        Memo1.Font.Assign(FontDlg.Font);
    end;
  finally
    FontDlg.Free;
  end;
end;
```

### 18.5.4 TColorDialog

```pascal
// TColorDialog - เลือกสี
procedure TForm1.ChooseColor;
var
  ColorDlg: TColorDialog;
begin
  ColorDlg := TColorDialog.Create(Self);
  try
    ColorDlg.Color := FCurrentColor;
    
    // Options
    ColorDlg.Options := [cdFullOpen, cdAnyColor];
    // cdFullOpen   - แสดง Custom Color Panel ทันที
    // cdAnyColor   - อนุญาตทุกสี
    // cdSolidColor - เฉพาะสีแบบ Solid
    
    if ColorDlg.Execute then
    begin
      FCurrentColor := ColorDlg.Color;
      PanelColorPreview.Color := FCurrentColor;
      
      // Apply สีที่เลือก
      if RichEdit1.SelLength > 0 then
        RichEdit1.SelAttributes.Color := FCurrentColor
      else
        Memo1.Font.Color := FCurrentColor;
    end;
  finally
    ColorDlg.Free;
  end;
end;
```

### 18.5.5 TPrintDialog

```pascal
// TPrintDialog - ตั้งค่าการพิมพ์
procedure TForm1.PrintDocument;
var
  PrintDlg: TPrintDialog;
begin
  PrintDlg := TPrintDialog.Create(Self);
  try
    PrintDlg.Options := [poPageNums, poSelection, poPrintToFile];
    PrintDlg.Copies := 1;
    PrintDlg.MinPage := 1;
    PrintDlg.MaxPage := GetPageCount;
    PrintDlg.FromPage := 1;
    PrintDlg.ToPage := GetPageCount;
    
    if PrintDlg.Execute then
    begin
      // เริ่มการพิมพ์
      Printer.BeginDoc;
      try
        DoPrint(PrintDlg.FromPage, PrintDlg.ToPage, PrintDlg.Copies);
      finally
        Printer.EndDoc;
      end;
    end;
  finally
    PrintDlg.Free;
  end;
end;
```

---

## 18.6 MessageDlg, InputBox, InputQuery

### 18.6.1 MessageDlg

```pascal
// MessageDlg - แสดงกล่องข้อความ
procedure TForm1.ShowMessages;
var
  result: Integer;
begin
  // ข้อความทั่วไป
  ShowMessage('ดำเนินการเสร็จสิ้น!');
  
  // MessageDlg พื้นฐาน
  result := MessageDlg('ต้องการออกจากโปรแกรมหรือไม่?',
                        mtConfirmation,
                        [mbYes, mbNo],
                        0);
  if result = mrYes then
    Close;
  
  // ประเภท Dialog (mtXxx):
  // mtWarning      - คำเตือน (มีไอคอนอัศเจรีย์)
  // mtError        - ข้อผิดพลาด (มีไอคอน X)
  // mtInformation  - ข้อมูล (มีไอคอน i)
  // mtConfirmation - ยืนยัน (มีไอคอน ?)
  // mtCustom       - Custom ไม่มีไอคอน
  
  // ปุ่มที่ใช้ได้:
  // mbYes, mbNo, mbOK, mbCancel, mbAbort, mbRetry,
  // mbIgnore, mbAll, mbNoToAll, mbYesToAll, mbHelp
  
  // ผลลัพธ์:
  // mrYes, mrNo, mrOK, mrCancel, mrAbort, mrRetry,
  // mrIgnore, mrAll, mrNoToAll, mrYesToAll
  
  // ตัวอย่างขั้นสูง
  case MessageDlg(
    'ไฟล์ถูกแก้ไข ต้องการบันทึกก่อนปิดหรือไม่?',
    mtConfirmation,
    [mbYes, mbNo, mbCancel],
    0) of
    mrYes:
    begin
      SaveFile;
      CloseFile;
    end;
    mrNo:
      CloseFile;
    mrCancel:
      ; // ไม่ทำอะไร
  end;
end;

// แสดงข้อผิดพลาด
procedure TForm1.ShowError(const Msg: string);
begin
  MessageDlg(Msg, mtError, [mbOK], 0);
end;

// แสดงคำเตือน
procedure TForm1.ShowWarning(const Msg: string);
begin
  MessageDlg(Msg, mtWarning, [mbOK], 0);
end;

// ยืนยันก่อนลบ
function TForm1.ConfirmDelete(const ItemName: string): Boolean;
begin
  Result := MessageDlg(
    'ต้องการลบ "' + ItemName + '" หรือไม่?' + #13#10 +
    'การดำเนินการนี้ไม่สามารถยกเลิกได้',
    mtWarning,
    [mbYes, mbNo],
    0
  ) = mrYes;
end;
```

### 18.6.2 InputBox และ InputQuery

```pascal
// InputBox - กล่องรับข้อมูลง่ายๆ
procedure TForm1.GetUserInput;
var
  userName: string;
begin
  userName := InputBox(
    'ใส่ชื่อ',       // Title
    'กรุณากรอกชื่อของคุณ:', // Message/Prompt
    'ผู้ใช้'          // Default Value
  );
  
  if userName <> '' then
    ShowMessage('สวัสดี, ' + userName);
end;

// InputQuery - เหมือน InputBox แต่คืนค่า Boolean (ผู้ใช้กด OK หรือ Cancel)
procedure TForm1.GetSearchTerm;
var
  searchTerm: string;
begin
  searchTerm := '';
  if InputQuery('ค้นหา', 'กรอกคำที่ต้องการค้นหา:', searchTerm) then
  begin
    if searchTerm <> '' then
      PerformSearch(searchTerm)
    else
      ShowMessage('กรุณากรอกคำค้นหา');
  end;
  // ถ้ากด Cancel จะ Return False และ searchTerm ไม่เปลี่ยน
end;

// ตัวอย่างการใช้ InputQuery สำหรับ Rename
procedure TForm1.RenameItem;
var
  newName: string;
begin
  newName := CurrentItemName;
  if InputQuery('เปลี่ยนชื่อ', 
                'ชื่อใหม่:', 
                newName) then
  begin
    if Trim(newName) = '' then
      ShowMessage('ชื่อต้องไม่ว่าง')
    else if newName <> CurrentItemName then
      DoRename(CurrentItemName, newName);
  end;
end;
```

---

## 18.7 Custom Dialogs

### 18.7.1 Dialog Form พื้นฐาน

```pascal
// สร้าง Custom Dialog Form
unit CustomDialog;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls;

type
  TCustomDialogForm = class(TForm)
    PanelContent: TPanel;
    LabelMessage: TLabel;
    EditInput: TEdit;
    PanelButtons: TPanel;
    ButtonOK: TButton;
    ButtonCancel: TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure ButtonOKClick(Sender: TObject);
    procedure ButtonCancelClick(Sender: TObject);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    
  private
    FInputValue: string;
    
  public
    class function ShowDialog(const ATitle, AMessage, ADefaultValue: string;
      out AResult: string): Boolean;
    property InputValue: string read FInputValue;
  end;

implementation

{$R *.lfm}

uses LCLType;

class function TCustomDialogForm.ShowDialog(
  const ATitle, AMessage, ADefaultValue: string;
  out AResult: string): Boolean;
var
  dlg: TCustomDialogForm;
begin
  dlg := TCustomDialogForm.Create(nil);
  try
    dlg.Caption := ATitle;
    dlg.LabelMessage.Caption := AMessage;
    dlg.EditInput.Text := ADefaultValue;
    
    Result := dlg.ShowModal = mrOk;
    if Result then
      AResult := dlg.FInputValue;
  finally
    dlg.Free;
  end;
end;

procedure TCustomDialogForm.FormCreate(Sender: TObject);
begin
  BorderStyle := bsDialog;
  Position := poOwnerFormCenter;
  Width := 400;
  Height := 180;
  KeyPreview := True;
  
  ButtonOK.ModalResult := mrNone; // จัดการเอง
  ButtonCancel.ModalResult := mrCancel;
  ButtonOK.Default := True;
  ButtonCancel.Cancel := True;
end;

procedure TCustomDialogForm.ButtonOKClick(Sender: TObject);
begin
  if Trim(EditInput.Text) = '' then
  begin
    ShowMessage('กรุณากรอกข้อมูล');
    EditInput.SetFocus;
    Exit;
  end;
  
  FInputValue := EditInput.Text;
  ModalResult := mrOk;
end;

procedure TCustomDialogForm.ButtonCancelClick(Sender: TObject);
begin
  ModalResult := mrCancel;
end;

procedure TCustomDialogForm.FormKeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  if Key = VK_ESCAPE then
    ModalResult := mrCancel;
end;

// การใช้งาน
procedure TMainForm.RenameButtonClick(Sender: TObject);
var
  newName: string;
begin
  if TCustomDialogForm.ShowDialog('เปลี่ยนชื่อ', 
                                  'กรอกชื่อใหม่:', 
                                  CurrentName,
                                  newName) then
  begin
    DoRename(newName);
  end;
end;
```

### 18.7.2 About Dialog

```pascal
unit AboutDialog;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, StdCtrls, ExtCtrls;

type
  TAboutDialog = class(TForm)
    ImageLogo: TImage;
    LabelAppName: TLabel;
    LabelVersion: TLabel;
    LabelCopyright: TLabel;
    LabelDescription: TMemo;
    Bevel1: TBevel;
    LabelWebsite: TLabel;
    ButtonOK: TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure LabelWebsiteClick(Sender: TObject);
    procedure ButtonOKClick(Sender: TObject);
  end;

implementation

{$R *.lfm}

procedure TAboutDialog.FormCreate(Sender: TObject);
begin
  Caption := 'เกี่ยวกับโปรแกรม';
  BorderStyle := bsDialog;
  Position := poOwnerFormCenter;
  Width := 450;
  Height := 350;
  
  LabelAppName.Caption := 'Text Editor Pro';
  LabelAppName.Font.Size := 18;
  LabelAppName.Font.Bold := True;
  
  LabelVersion.Caption := 'เวอร์ชัน 1.0.0';
  
  LabelCopyright.Caption := '© 2024 บริษัท ตัวอย่าง จำกัด';
  
  LabelDescription.Text := 
    'Text Editor Pro เป็นโปรแกรมแก้ไขข้อความที่มีประสิทธิภาพ' + #13#10 +
    'พัฒนาด้วย Lazarus/Free Pascal' + #13#10 +
    'รองรับไฟล์หลายรูปแบบ: TXT, RTF, HTML';
  LabelDescription.ReadOnly := True;
  LabelDescription.BorderStyle := bsNone;
  LabelDescription.Color := Color;
  
  LabelWebsite.Caption := 'www.example.com';
  LabelWebsite.Font.Color := clBlue;
  LabelWebsite.Font.Style := [fsUnderline];
  LabelWebsite.Cursor := crHandPoint;
  
  ButtonOK.Caption := 'ตกลง';
  ButtonOK.ModalResult := mrOk;
end;

procedure TAboutDialog.LabelWebsiteClick(Sender: TObject);
begin
  OpenURL('https://www.example.com');
end;

procedure TAboutDialog.ButtonOKClick(Sender: TObject);
begin
  ModalResult := mrOk;
end;
```

### 18.7.3 Progress Dialog

```pascal
unit ProgressDialog;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ComCtrls, ExtCtrls;

type
  TProgressCallback = procedure(Progress: Integer; const Status: string) of object;

  TProgressDialog = class(TForm)
    LabelTitle: TLabel;
    LabelStatus: TLabel;
    ProgressBar: TProgressBar;
    LabelPercent: TLabel;
    ButtonCancel: TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure ButtonCancelClick(Sender: TObject);
    
  private
    FCancelled: Boolean;
    
  public
    procedure SetProgress(Value: Integer; const Status: string);
    property Cancelled: Boolean read FCancelled;
    
    class function ShowProgress(const ATitle: string; 
      ATask: TProgressCallback): Boolean;
  end;

implementation

{$R *.lfm}

procedure TProgressDialog.FormCreate(Sender: TObject);
begin
  Caption := 'กำลังดำเนินการ...';
  BorderStyle := bsDialog;
  Position := poOwnerFormCenter;
  Width := 450;
  Height := 170;
  
  ProgressBar.Min := 0;
  ProgressBar.Max := 100;
  ProgressBar.Position := 0;
  
  FCancelled := False;
  ButtonCancel.Caption := 'ยกเลิก';
end;

procedure TProgressDialog.SetProgress(Value: Integer; const Status: string);
begin
  ProgressBar.Position := Value;
  LabelPercent.Caption := IntToStr(Value) + '%';
  LabelStatus.Caption := Status;
  Application.ProcessMessages; // อัปเดต UI
end;

procedure TProgressDialog.ButtonCancelClick(Sender: TObject);
begin
  if MessageDlg('ต้องการยกเลิกการดำเนินการหรือไม่?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    FCancelled := True;
    ButtonCancel.Enabled := False;
    ButtonCancel.Caption := 'กำลังยกเลิก...';
  end;
end;
```

---

## 18.8 Modal vs Non-Modal Dialogs

### 18.8.1 Modal Dialog

```pascal
// Modal Dialog - ผู้ใช้ต้องปิด Dialog ก่อนใช้งาน Form หลัก
procedure TMainForm.ShowModalDialog;
var
  dlg: TSettingsForm;
begin
  dlg := TSettingsForm.Create(Self);
  try
    // กำหนดค่าเริ่มต้น
    dlg.LoadSettings;
    
    // ShowModal รอจนกว่า Dialog ปิด
    if dlg.ShowModal = mrOk then
    begin
      // Apply Settings ที่เปลี่ยน
      ApplySettings(dlg.GetSettings);
    end;
  finally
    dlg.Free; // ต้อง Free เอง
  end;
end;

// Dialog ที่ Return ผลลัพธ์
function TMainForm.AskUserName: string;
var
  dlg: TInputForm;
begin
  Result := '';
  dlg := TInputForm.Create(Self);
  try
    dlg.Caption := 'ชื่อผู้ใช้';
    if dlg.ShowModal = mrOk then
      Result := dlg.InputText;
  finally
    dlg.Free;
  end;
end;
```

### 18.8.2 Non-Modal Dialog

```pascal
// Non-Modal Dialog - ผู้ใช้สามารถใช้งาน Form อื่นได้ขณะที่ Dialog เปิดอยู่
procedure TMainForm.ShowFindDialog;
begin
  if not Assigned(FFindDialog) then
  begin
    FFindDialog := TFindReplaceForm.Create(Self);
    FFindDialog.OnFind := HandleFind;
    FFindDialog.OnReplace := HandleReplace;
  end;
  
  FFindDialog.Show; // ไม่ใช่ ShowModal
  FFindDialog.SetFocus;
end;

// Non-Modal Dialog ต้องการการสื่อสารผ่าน Events/Callbacks
procedure TMainForm.HandleFind(const SearchText: string; Options: TFindOptions);
begin
  // ค้นหาใน Active Editor
  if Assigned(ActiveEditor) then
    ActiveEditor.FindText(SearchText, Options);
end;

// ทำลาย Non-Modal Dialog เมื่อปิด
procedure TFindReplaceForm.FormClose(Sender: TObject; var CloseAction: TCloseAction);
begin
  CloseAction := caFree; // Free เมื่อปิด
  // หรือ caHide เพื่อซ่อนและนำกลับมาใช้ได้อีก
end;
```

---

## 18.9 โปรแกรมตัวอย่าง: Notepad Clone

```pascal
unit NotepadClone;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  Menus, StdCtrls, ExtCtrls, ComCtrls, Clipbrd, LCLType,
  FindReplaceDialog;

type
  TNotepadForm = class(TForm)
    MainMenu1: TMainMenu;
    Memo1: TMemo;
    StatusBar1: TStatusBar;
    
    // File Menu
    MenuFile: TMenuItem;
    MenuNew: TMenuItem;
    MenuOpen: TMenuItem;
    MenuSave: TMenuItem;
    MenuSaveAs: TMenuItem;
    MenuSep1: TMenuItem;
    MenuPageSetup: TMenuItem;
    MenuPrint: TMenuItem;
    MenuSep2: TMenuItem;
    MenuExit: TMenuItem;
    
    // Edit Menu
    MenuEdit: TMenuItem;
    MenuUndo: TMenuItem;
    MenuSepEdit1: TMenuItem;
    MenuCut: TMenuItem;
    MenuCopy: TMenuItem;
    MenuPaste: TMenuItem;
    MenuDelete: TMenuItem;
    MenuSepEdit2: TMenuItem;
    MenuFind: TMenuItem;
    MenuFindNext: TMenuItem;
    MenuReplace: TMenuItem;
    MenuGoTo: TMenuItem;
    MenuSepEdit3: TMenuItem;
    MenuSelectAll: TMenuItem;
    MenuTimeDate: TMenuItem;
    
    // Format Menu
    MenuFormat: TMenuItem;
    MenuWordWrap: TMenuItem;
    MenuFont: TMenuItem;
    
    // View Menu
    MenuView: TMenuItem;
    MenuStatusBar: TMenuItem;
    
    // Help Menu
    MenuHelp: TMenuItem;
    MenuViewHelp: TMenuItem;
    MenuSepHelp: TMenuItem;
    MenuAbout: TMenuItem;
    
    procedure FormCreate(Sender: TObject);
    procedure FormCloseQuery(Sender: TObject; var CanClose: Boolean);
    procedure FormDestroy(Sender: TObject);
    
    // File Menu Handlers
    procedure MenuNewClick(Sender: TObject);
    procedure MenuOpenClick(Sender: TObject);
    procedure MenuSaveClick(Sender: TObject);
    procedure MenuSaveAsClick(Sender: TObject);
    procedure MenuPrintClick(Sender: TObject);
    procedure MenuExitClick(Sender: TObject);
    
    // Edit Menu Handlers
    procedure MenuUndoClick(Sender: TObject);
    procedure MenuCutClick(Sender: TObject);
    procedure MenuCopyClick(Sender: TObject);
    procedure MenuPasteClick(Sender: TObject);
    procedure MenuDeleteClick(Sender: TObject);
    procedure MenuFindClick(Sender: TObject);
    procedure MenuReplaceClick(Sender: TObject);
    procedure MenuSelectAllClick(Sender: TObject);
    procedure MenuTimeDateClick(Sender: TObject);
    
    // Format Menu Handlers
    procedure MenuWordWrapClick(Sender: TObject);
    procedure MenuFontClick(Sender: TObject);
    
    // View Menu Handlers
    procedure MenuStatusBarClick(Sender: TObject);
    
    // Help Menu Handlers
    procedure MenuAboutClick(Sender: TObject);
    
    // Memo Events
    procedure Memo1Change(Sender: TObject);
    procedure Memo1KeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure Memo1Click(Sender: TObject);
    
    // Edit Menu Update
    procedure MenuEditClick(Sender: TObject);
    
  private
    FFileName: string;
    FModified: Boolean;
    FSearchText: string;
    FFindOptions: TFindOptions;
    
    procedure UpdateTitle;
    procedure UpdateStatusBar;
    procedure UpdateEditMenu;
    procedure SetModified(Value: Boolean);
    function SaveCurrentFile: Boolean;
    function AskSaveChanges: Integer;  // mrYes, mrNo, mrCancel
    procedure LoadFile(const FileName: string);
    procedure SaveFile(const FileName: string);
    procedure AddToRecentFiles(const FileName: string);
    function FindNext(const SearchText: string; Options: TFindOptions): Boolean;
    procedure ShowFindDialog;
    procedure ShowReplaceDialog;
    
    property Modified: Boolean read FModified write SetModified;
  end;

var
  NotepadForm: TNotepadForm;

implementation

{$R *.lfm}

uses
  Printers;

const
  APP_TITLE = 'Notepad Clone';
  MAX_RECENT_FILES = 10;

procedure TNotepadForm.FormCreate(Sender: TObject);
begin
  // กำหนด Properties ของ Form
  Caption := APP_TITLE;
  Width := 800;
  Height := 600;
  Position := poScreenCenter;
  KeyPreview := True;
  
  // กำหนด Memo
  Memo1.Align := alClient;
  Memo1.ScrollBars := ssBoth;
  Memo1.WordWrap := False;
  Memo1.Font.Name := 'Courier New';
  Memo1.Font.Size := 11;
  
  // Initialize State
  FFileName := '';
  FModified := False;
  FSearchText := '';
  
  // Update UI
  UpdateTitle;
  UpdateStatusBar;
  
  // Menu Checkmarks
  MenuWordWrap.Checked := Memo1.WordWrap;
  MenuStatusBar.Checked := StatusBar1.Visible;
  
  // Status Bar
  StatusBar1.Panels.Add;
  StatusBar1.Panels.Add;
  StatusBar1.Panels[0].Width := 200;
  StatusBar1.Panels[0].Text := 'บรรทัดที่ 1, คอลัมน์ที่ 1';
  StatusBar1.Panels[1].Width := 200;
  StatusBar1.Panels[1].Text := 'UTF-8';
end;

procedure TNotepadForm.FormCloseQuery(Sender: TObject; var CanClose: Boolean);
begin
  CanClose := True;
  if FModified then
  begin
    case AskSaveChanges of
      mrYes:
        if not SaveCurrentFile then
          CanClose := False;
      mrCancel:
        CanClose := False;
    end;
  end;
end;

procedure TNotepadForm.FormDestroy(Sender: TObject);
begin
  // Cleanup
end;

procedure TNotepadForm.UpdateTitle;
begin
  if FFileName = '' then
    Caption := 'ไม่มีชื่อ - ' + APP_TITLE
  else
    Caption := ExtractFileName(FFileName) + ' - ' + APP_TITLE;
    
  if FModified then
    Caption := '*' + Caption;
end;

procedure TNotepadForm.UpdateStatusBar;
var
  line, col: Integer;
  selStart: Integer;
  lineText: string;
begin
  if not StatusBar1.Visible then Exit;
  
  // คำนวณตำแหน่ง Cursor
  selStart := Memo1.SelStart;
  lineText := Copy(Memo1.Text, 1, selStart);
  line := 1;
  col := selStart + 1;
  
  for var i := 1 to Length(lineText) do
  begin
    if lineText[i] = #13 then
    begin
      Inc(line);
      col := selStart - i + 1;
    end;
  end;
  
  StatusBar1.Panels[0].Text := 
    'บรรทัดที่ ' + IntToStr(line) + ', คอลัมน์ที่ ' + IntToStr(col);
end;

procedure TNotepadForm.SetModified(Value: Boolean);
begin
  if FModified <> Value then
  begin
    FModified := Value;
    UpdateTitle;
  end;
end;

function TNotepadForm.AskSaveChanges: Integer;
begin
  Result := MessageDlg(
    'ต้องการบันทึกการเปลี่ยนแปลงใน "' + 
    IfThen(FFileName = '', 'ไม่มีชื่อ', ExtractFileName(FFileName)) + 
    '" หรือไม่?',
    mtConfirmation,
    [mbYes, mbNo, mbCancel],
    0
  );
end;

procedure TNotepadForm.LoadFile(const FileName: string);
begin
  try
    Memo1.Lines.LoadFromFile(FileName);
    FFileName := FileName;
    Modified := False;
    AddToRecentFiles(FileName);
    StatusBar1.Panels[0].Text := 'โหลด: ' + ExtractFileName(FileName);
  except
    on E: Exception do
      ShowMessage('ไม่สามารถเปิดไฟล์: ' + E.Message);
  end;
end;

procedure TNotepadForm.SaveFile(const FileName: string);
begin
  try
    Memo1.Lines.SaveToFile(FileName);
    FFileName := FileName;
    Modified := False;
    StatusBar1.Panels[0].Text := 'บันทึกแล้ว: ' + ExtractFileName(FileName);
  except
    on E: Exception do
      ShowMessage('ไม่สามารถบันทึกไฟล์: ' + E.Message);
  end;
end;

function TNotepadForm.SaveCurrentFile: Boolean;
var
  SaveDlg: TSaveDialog;
begin
  Result := True;
  
  if FFileName = '' then
  begin
    SaveDlg := TSaveDialog.Create(Self);
    try
      SaveDlg.Title := 'บันทึกไฟล์';
      SaveDlg.Filter := 'Text Files|*.txt|All Files|*.*';
      SaveDlg.DefaultExt := 'txt';
      
      if SaveDlg.Execute then
        SaveFile(SaveDlg.FileName)
      else
        Result := False;
    finally
      SaveDlg.Free;
    end;
  end
  else
    SaveFile(FFileName);
end;

procedure TNotepadForm.AddToRecentFiles(const FileName: string);
begin
  // TODO: Implement recent files list
end;

procedure TNotepadForm.MenuNewClick(Sender: TObject);
begin
  if FModified then
    case AskSaveChanges of
      mrYes:
        if not SaveCurrentFile then Exit;
      mrCancel:
        Exit;
    end;
    
  Memo1.Clear;
  FFileName := '';
  Modified := False;
  UpdateTitle;
end;

procedure TNotepadForm.MenuOpenClick(Sender: TObject);
var
  OpenDlg: TOpenDialog;
begin
  if FModified then
    case AskSaveChanges of
      mrYes:
        if not SaveCurrentFile then Exit;
      mrCancel:
        Exit;
    end;
    
  OpenDlg := TOpenDialog.Create(Self);
  try
    OpenDlg.Title := 'เปิดไฟล์';
    OpenDlg.Filter := 'Text Files|*.txt|All Files|*.*';
    OpenDlg.Options := [ofFileMustExist];
    
    if OpenDlg.Execute then
      LoadFile(OpenDlg.FileName);
  finally
    OpenDlg.Free;
  end;
end;

procedure TNotepadForm.MenuSaveClick(Sender: TObject);
begin
  SaveCurrentFile;
end;

procedure TNotepadForm.MenuSaveAsClick(Sender: TObject);
var
  SaveDlg: TSaveDialog;
begin
  SaveDlg := TSaveDialog.Create(Self);
  try
    SaveDlg.Title := 'บันทึกไฟล์เป็น';
    SaveDlg.Filter := 'Text Files|*.txt|All Files|*.*';
    SaveDlg.DefaultExt := 'txt';
    SaveDlg.FileName := ExtractFileName(FFileName);
    SaveDlg.Options := [ofOverwritePrompt];
    
    if SaveDlg.Execute then
      SaveFile(SaveDlg.FileName);
  finally
    SaveDlg.Free;
  end;
end;

procedure TNotepadForm.MenuPrintClick(Sender: TObject);
var
  PrintDlg: TPrintDialog;
begin
  PrintDlg := TPrintDialog.Create(Self);
  try
    if PrintDlg.Execute then
    begin
      Printer.BeginDoc;
      try
        Printer.Canvas.TextOut(100, 100, 'กำลังพิมพ์...');
        // TODO: Print Memo content properly
      finally
        Printer.EndDoc;
      end;
    end;
  finally
    PrintDlg.Free;
  end;
end;

procedure TNotepadForm.MenuExitClick(Sender: TObject);
begin
  Close;
end;

procedure TNotepadForm.MenuUndoClick(Sender: TObject);
begin
  Memo1.Undo;
end;

procedure TNotepadForm.MenuCutClick(Sender: TObject);
begin
  Memo1.CutToClipboard;
end;

procedure TNotepadForm.MenuCopyClick(Sender: TObject);
begin
  Memo1.CopyToClipboard;
end;

procedure TNotepadForm.MenuPasteClick(Sender: TObject);
begin
  Memo1.PasteFromClipboard;
end;

procedure TNotepadForm.MenuDeleteClick(Sender: TObject);
begin
  Memo1.ClearSelection;
end;

procedure TNotepadForm.MenuFindClick(Sender: TObject);
begin
  ShowFindDialog;
end;

procedure TNotepadForm.ShowFindDialog;
var
  searchTerm: string;
begin
  searchTerm := FSearchText;
  if InputQuery('ค้นหา', 'ค้นหา:', searchTerm) then
  begin
    if searchTerm <> '' then
    begin
      FSearchText := searchTerm;
      if not FindNext(FSearchText, FFindOptions) then
        ShowMessage('"' + FSearchText + '" ไม่พบ');
    end;
  end;
end;

function TNotepadForm.FindNext(const SearchText: string; 
  Options: TFindOptions): Boolean;
var
  foundPos: Integer;
  searchIn: string;
  startPos: Integer;
begin
  Result := False;
  if SearchText = '' then Exit;
  
  startPos := Memo1.SelStart + Memo1.SelLength;
  
  if frMatchCase in Options then
    searchIn := Copy(Memo1.Text, startPos + 1, MaxInt)
  else
    searchIn := LowerCase(Copy(Memo1.Text, startPos + 1, MaxInt));
    
  if frMatchCase in Options then
    foundPos := Pos(SearchText, searchIn)
  else
    foundPos := Pos(LowerCase(SearchText), searchIn);
  
  if foundPos > 0 then
  begin
    Memo1.SelStart := startPos + foundPos - 1;
    Memo1.SelLength := Length(SearchText);
    Result := True;
  end
  else if startPos > 0 then
  begin
    // ค้นหาจากต้น
    if frMatchCase in Options then
      foundPos := Pos(SearchText, Memo1.Text)
    else
      foundPos := Pos(LowerCase(SearchText), LowerCase(Memo1.Text));
      
    if foundPos > 0 then
    begin
      Memo1.SelStart := foundPos - 1;
      Memo1.SelLength := Length(SearchText);
      Result := True;
    end;
  end;
  
  if Result then
    Memo1.SetFocus;
end;

procedure TNotepadForm.ShowReplaceDialog;
begin
  // TODO: Implement Replace Dialog
end;

procedure TNotepadForm.MenuReplaceClick(Sender: TObject);
begin
  ShowReplaceDialog;
end;

procedure TNotepadForm.MenuSelectAllClick(Sender: TObject);
begin
  Memo1.SelectAll;
end;

procedure TNotepadForm.MenuTimeDateClick(Sender: TObject);
begin
  Memo1.SelText := FormatDateTime('hh:nn dd/mm/yyyy', Now);
end;

procedure TNotepadForm.MenuWordWrapClick(Sender: TObject);
begin
  MenuWordWrap.Checked := not MenuWordWrap.Checked;
  Memo1.WordWrap := MenuWordWrap.Checked;
  if Memo1.WordWrap then
    Memo1.ScrollBars := ssVertical
  else
    Memo1.ScrollBars := ssBoth;
end;

procedure TNotepadForm.MenuFontClick(Sender: TObject);
var
  FontDlg: TFontDialog;
begin
  FontDlg := TFontDialog.Create(Self);
  try
    FontDlg.Font := Memo1.Font;
    if FontDlg.Execute then
      Memo1.Font.Assign(FontDlg.Font);
  finally
    FontDlg.Free;
  end;
end;

procedure TNotepadForm.MenuStatusBarClick(Sender: TObject);
begin
  MenuStatusBar.Checked := not MenuStatusBar.Checked;
  StatusBar1.Visible := MenuStatusBar.Checked;
end;

procedure TNotepadForm.MenuAboutClick(Sender: TObject);
begin
  ShowMessage(
    APP_TITLE + #13#10 +
    'เวอร์ชัน 1.0' + #13#10 +
    'พัฒนาด้วย Lazarus/Free Pascal' + #13#10 +
    #13#10 +
    'โปรแกรม Notepad Clone สำหรับการศึกษา'
  );
end;

procedure TNotepadForm.Memo1Change(Sender: TObject);
begin
  Modified := True;
  UpdateStatusBar;
end;

procedure TNotepadForm.Memo1KeyUp(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  UpdateStatusBar;
end;

procedure TNotepadForm.Memo1Click(Sender: TObject);
begin
  UpdateStatusBar;
end;

procedure TNotepadForm.MenuEditClick(Sender: TObject);
begin
  UpdateEditMenu;
end;

procedure TNotepadForm.UpdateEditMenu;
begin
  MenuUndo.Enabled := Memo1.CanUndo;
  MenuCut.Enabled := Memo1.SelLength > 0;
  MenuCopy.Enabled := Memo1.SelLength > 0;
  MenuPaste.Enabled := Clipboard.HasFormat(CF_TEXT);
  MenuDelete.Enabled := Memo1.SelLength > 0;
end;

end.
```

---

## 18.10 โปรแกรมตัวอย่าง: Color Picker App

```pascal
unit ColorPickerApp;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls, Spin, Menus;

type
  TColorPickerForm = class(TForm)
    // Panels
    PanelLeft: TPanel;
    PanelRight: TPanel;
    PanelPreview: TPanel;
    
    // Color Wheel (ใช้ TImage เพื่อง่าย)
    ImageColorWheel: TImage;
    
    // RGB Sliders
    GroupBoxRGB: TGroupBox;
    LabelR: TLabel;
    TrackBarR: TTrackBar;
    SpinEditR: TSpinEdit;
    LabelG: TLabel;
    TrackBarG: TTrackBar;
    SpinEditG: TSpinEdit;
    LabelB: TLabel;
    TrackBarB: TTrackBar;
    SpinEditB: TSpinEdit;
    
    // HSL Sliders
    GroupBoxHSL: TGroupBox;
    LabelH: TLabel;
    TrackBarH: TTrackBar;
    SpinEditH: TSpinEdit;
    LabelS: TLabel;
    TrackBarS: TTrackBar;
    SpinEditS: TSpinEdit;
    LabelL: TLabel;
    TrackBarL: TTrackBar;
    SpinEditL: TSpinEdit;
    
    // Hex Input
    LabelHex: TLabel;
    EditHex: TEdit;
    ButtonApplyHex: TButton;
    
    // Color History
    GroupBoxHistory: TGroupBox;
    FlowPanelHistory: TFlowPanel;
    
    // Buttons
    ButtonOK: TButton;
    ButtonCancel: TButton;
    ButtonAdd: TButton;
    ButtonCopyHex: TButton;
    
    // Menus
    PopupMenuHistory: TPopupMenu;
    
    procedure FormCreate(Sender: TObject);
    procedure TrackBarRGBChange(Sender: TObject);
    procedure SpinEditRGBChange(Sender: TObject);
    procedure TrackBarHSLChange(Sender: TObject);
    procedure EditHexChange(Sender: TObject);
    procedure ButtonApplyHexClick(Sender: TObject);
    procedure ButtonAddClick(Sender: TObject);
    procedure ButtonCopyHexClick(Sender: TObject);
    procedure ButtonOKClick(Sender: TObject);
    
  private
    FCurrentColor: TColor;
    FUpdating: Boolean;
    FColorHistory: array[0..19] of TColor;
    FHistoryCount: Integer;
    
    procedure SetColor(AColor: TColor);
    procedure UpdateRGBFromColor;
    procedure UpdateHSLFromColor;
    procedure UpdateHexFromColor;
    procedure UpdatePreview;
    procedure RGBToHSL(R, G, B: Byte; out H, S, L: Double);
    procedure HSLToRGB(H, S, L: Double; out R, G, B: Byte);
    function HexToColor(const HexStr: string): TColor;
    function ColorToHex(AColor: TColor): string;
    procedure AddColorToHistory(AColor: TColor);
    procedure DrawHistory;
    procedure ColorHistoryClick(Sender: TObject);
    
  public
    property SelectedColor: TColor read FCurrentColor;
  end;

implementation

{$R *.lfm}

uses
  Clipbrd, Math;

procedure TColorPickerForm.FormCreate(Sender: TObject);
begin
  Caption := 'เลือกสี';
  Width := 600;
  Height := 500;
  Position := poScreenCenter;
  
  // Setup Sliders
  TrackBarR.Min := 0; TrackBarR.Max := 255;
  TrackBarG.Min := 0; TrackBarG.Max := 255;
  TrackBarB.Min := 0; TrackBarB.Max := 255;
  
  SpinEditR.MinValue := 0; SpinEditR.MaxValue := 255;
  SpinEditG.MinValue := 0; SpinEditG.MaxValue := 255;
  SpinEditB.MinValue := 0; SpinEditB.MaxValue := 255;
  
  TrackBarH.Min := 0; TrackBarH.Max := 360;
  TrackBarS.Min := 0; TrackBarS.Max := 100;
  TrackBarL.Min := 0; TrackBarL.Max := 100;
  
  // เชื่อม Events
  TrackBarR.OnChange := TrackBarRGBChange;
  TrackBarG.OnChange := TrackBarRGBChange;
  TrackBarB.OnChange := TrackBarRGBChange;
  
  SpinEditR.OnChange := SpinEditRGBChange;
  SpinEditG.OnChange := SpinEditRGBChange;
  SpinEditB.OnChange := SpinEditRGBChange;
  
  // Initialize Color
  FUpdating := False;
  FHistoryCount := 0;
  SetColor(clBlue);
  
  // Initialize History with default colors
  AddColorToHistory(clBlack);
  AddColorToHistory(clWhite);
  AddColorToHistory(clRed);
  AddColorToHistory(clGreen);
  AddColorToHistory(clBlue);
  AddColorToHistory(clYellow);
  AddColorToHistory(clCyan);
  AddColorToHistory(clMagenta);
  DrawHistory;
end;

procedure TColorPickerForm.SetColor(AColor: TColor);
begin
  if FCurrentColor = AColor then Exit;
  
  FCurrentColor := AColor;
  FUpdating := True;
  try
    UpdateRGBFromColor;
    UpdateHSLFromColor;
    UpdateHexFromColor;
    UpdatePreview;
  finally
    FUpdating := False;
  end;
end;

procedure TColorPickerForm.UpdateRGBFromColor;
var
  r, g, b: Byte;
begin
  r := GetRValue(FCurrentColor);
  g := GetGValue(FCurrentColor);
  b := GetBValue(FCurrentColor);
  
  TrackBarR.Position := r;
  TrackBarG.Position := g;
  TrackBarB.Position := b;
  
  SpinEditR.Value := r;
  SpinEditG.Value := g;
  SpinEditB.Value := b;
end;

procedure TColorPickerForm.UpdateHexFromColor;
begin
  EditHex.Text := ColorToHex(FCurrentColor);
end;

procedure TColorPickerForm.UpdatePreview;
begin
  PanelPreview.Color := FCurrentColor;
  PanelPreview.Caption := ColorToHex(FCurrentColor);
  PanelPreview.Font.Color := 
    IfThen(
      (GetRValue(FCurrentColor) * 299 + GetGValue(FCurrentColor) * 587 + 
       GetBValue(FCurrentColor) * 114) div 1000 > 128,
      clBlack, 
      clWhite
    );
end;

function TColorPickerForm.ColorToHex(AColor: TColor): string;
begin
  Result := '#' + 
    IntToHex(GetRValue(AColor), 2) + 
    IntToHex(GetGValue(AColor), 2) + 
    IntToHex(GetBValue(AColor), 2);
end;

function TColorPickerForm.HexToColor(const HexStr: string): TColor;
var
  s: string;
  r, g, b: Integer;
begin
  s := Trim(HexStr);
  if (Length(s) > 0) and (s[1] = '#') then
    s := Copy(s, 2, MaxInt);
    
  if Length(s) = 6 then
  begin
    r := StrToIntDef('$' + Copy(s, 1, 2), 0);
    g := StrToIntDef('$' + Copy(s, 3, 2), 0);
    b := StrToIntDef('$' + Copy(s, 5, 2), 0);
    Result := RGB(r, g, b);
  end
  else
    Result := clBlack;
end;

procedure TColorPickerForm.TrackBarRGBChange(Sender: TObject);
begin
  if FUpdating then Exit;
  
  SetColor(RGB(TrackBarR.Position, TrackBarG.Position, TrackBarB.Position));
end;

procedure TColorPickerForm.SpinEditRGBChange(Sender: TObject);
begin
  if FUpdating then Exit;
  
  SetColor(RGB(SpinEditR.Value, SpinEditG.Value, SpinEditB.Value));
end;

procedure TColorPickerForm.TrackBarHSLChange(Sender: TObject);
var
  r, g, b: Byte;
begin
  if FUpdating then Exit;
  
  HSLToRGB(
    TrackBarH.Position, 
    TrackBarS.Position / 100, 
    TrackBarL.Position / 100, 
    r, g, b
  );
  SetColor(RGB(r, g, b));
end;

procedure TColorPickerForm.EditHexChange(Sender: TObject);
begin
  // ไม่ต้อง Update ทันที รอจนกด Apply
end;

procedure TColorPickerForm.ButtonApplyHexClick(Sender: TObject);
begin
  SetColor(HexToColor(EditHex.Text));
end;

procedure TColorPickerForm.ButtonAddClick(Sender: TObject);
begin
  AddColorToHistory(FCurrentColor);
  DrawHistory;
end;

procedure TColorPickerForm.ButtonCopyHexClick(Sender: TObject);
begin
  Clipboard.AsText := ColorToHex(FCurrentColor);
  ShowMessage('คัดลอก ' + ColorToHex(FCurrentColor) + ' แล้ว');
end;

procedure TColorPickerForm.ButtonOKClick(Sender: TObject);
begin
  ModalResult := mrOk;
end;

procedure TColorPickerForm.AddColorToHistory(AColor: TColor);
var
  i: Integer;
begin
  // ตรวจว่ามีอยู่แล้วหรือไม่
  for i := 0 to FHistoryCount - 1 do
    if FColorHistory[i] = AColor then Exit;
  
  if FHistoryCount < 20 then
  begin
    FColorHistory[FHistoryCount] := AColor;
    Inc(FHistoryCount);
  end
  else
  begin
    // เลื่อนออกสีเก่าสุด
    for i := 0 to 18 do
      FColorHistory[i] := FColorHistory[i + 1];
    FColorHistory[19] := AColor;
  end;
end;

procedure TColorPickerForm.DrawHistory;
var
  i: Integer;
  panel: TPanel;
begin
  // ลบ Panels เก่า
  FlowPanelHistory.DestroyComponents;
  
  // สร้าง Panel ใหม่สำหรับแต่ละสีใน History
  for i := 0 to FHistoryCount - 1 do
  begin
    panel := TPanel.Create(FlowPanelHistory);
    panel.Parent := FlowPanelHistory;
    panel.Width := 25;
    panel.Height := 25;
    panel.Color := FColorHistory[i];
    panel.BevelOuter := bvRaised;
    panel.Caption := '';
    panel.Tag := i;
    panel.Hint := ColorToHex(FColorHistory[i]);
    panel.ShowHint := True;
    panel.Cursor := crHandPoint;
    panel.OnClick := ColorHistoryClick;
  end;
end;

procedure TColorPickerForm.ColorHistoryClick(Sender: TObject);
begin
  SetColor(TPanel(Sender).Color);
end;

procedure TColorPickerForm.RGBToHSL(R, G, B: Byte; out H, S, L: Double);
var
  maxC, minC, delta: Double;
  rf, gf, bf: Double;
begin
  rf := R / 255;
  gf := G / 255;
  bf := B / 255;
  
  maxC := Max(Max(rf, gf), bf);
  minC := Min(Min(rf, gf), bf);
  delta := maxC - minC;
  
  L := (maxC + minC) / 2;
  
  if delta = 0 then
  begin
    H := 0;
    S := 0;
  end
  else
  begin
    if L < 0.5 then
      S := delta / (maxC + minC)
    else
      S := delta / (2 - maxC - minC);
    
    if rf = maxC then
      H := (gf - bf) / delta
    else if gf = maxC then
      H := 2 + (bf - rf) / delta
    else
      H := 4 + (rf - gf) / delta;
    
    H := H * 60;
    if H < 0 then H := H + 360;
  end;
end;

procedure TColorPickerForm.HSLToRGB(H, S, L: Double; out R, G, B: Byte);
  function HueToRGB(p, q, t: Double): Double;
  begin
    if t < 0 then t := t + 1;
    if t > 1 then t := t - 1;
    if t < 1/6 then
      Result := p + (q - p) * 6 * t
    else if t < 1/2 then
      Result := q
    else if t < 2/3 then
      Result := p + (q - p) * (2/3 - t) * 6
    else
      Result := p;
  end;

var
  q, p: Double;
begin
  if S = 0 then
  begin
    R := Round(L * 255);
    G := R;
    B := R;
  end
  else
  begin
    if L < 0.5 then
      q := L * (1 + S)
    else
      q := L + S - L * S;
    p := 2 * L - q;
    
    R := Round(HueToRGB(p, q, H/360 + 1/3) * 255);
    G := Round(HueToRGB(p, q, H/360) * 255);
    B := Round(HueToRGB(p, q, H/360 - 1/3) * 255);
  end;
end;

procedure TColorPickerForm.UpdateHSLFromColor;
var
  h, s, l: Double;
begin
  RGBToHSL(
    GetRValue(FCurrentColor),
    GetGValue(FCurrentColor),
    GetBValue(FCurrentColor),
    h, s, l
  );
  
  TrackBarH.Position := Round(h);
  TrackBarS.Position := Round(s * 100);
  TrackBarL.Position := Round(l * 100);
  
  SpinEditH.Value := Round(h);
  SpinEditS.Value := Round(s * 100);
  SpinEditL.Value := Round(l * 100);
end;

end.
```

---

## 18.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Main Menu พื้นฐาน
สร้าง Menu Bar สำหรับ Application ที่มี File, Edit, View, Help

```pascal
procedure TSimpleMenuForm.SetupMenus;
begin
  MainMenu1 := TMainMenu.Create(Self);
  
  // สร้าง File Menu
  // ... (code ตามที่เรียนมา)
end;
```

### แบบฝึกหัดที่ 2: Context Menu
สร้าง Context Menu สำหรับ ListBox ที่มีรายการสินค้า

```pascal
procedure TProductForm.ListBoxProductMouseDown(Sender: TObject;
  Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  if Button = mbRight then
  begin
    ListBoxProduct.ItemIndex := ListBoxProduct.ItemAtPos(Point(X, Y), True);
    
    MenuItemEdit.Enabled := ListBoxProduct.ItemIndex >= 0;
    MenuItemDelete.Enabled := ListBoxProduct.ItemIndex >= 0;
    
    PopupMenuProduct.Popup(
      ListBoxProduct.ClientToScreen(Point(X, Y)).X,
      ListBoxProduct.ClientToScreen(Point(X, Y)).Y
    );
  end;
end;
```

### แบบฝึกหัดที่ 3: Recent Files
สร้าง Recent Files Menu ที่บันทึกและโหลดจาก INI File

```pascal
procedure TRecentFilesForm.SaveRecentFiles;
var
  ini: TIniFile;
  i: Integer;
begin
  ini := TIniFile.Create(GetAppDataPath + 'settings.ini');
  try
    ini.EraseSection('RecentFiles');
    for i := 0 to FRecentFiles.Count - 1 do
      ini.WriteString('RecentFiles', 'File' + IntToStr(i), FRecentFiles[i]);
  finally
    ini.Free;
  end;
end;
```

### แบบฝึกหัดที่ 4-10: เพิ่มเติม

- แบบฝึกหัดที่ 4: TOpenDialog ที่รองรับหลาย Format
- แบบฝึกหัดที่ 5: Custom Font Dialog ที่มี Preview
- แบบฝึกหัดที่ 6: Custom Color Palette Dialog
- แบบฝึกหัดที่ 7: Custom Progress Dialog สำหรับ Long Operation
- แบบฝึกหัดที่ 8: About Dialog ที่มีรูปและลิงก์
- แบบฝึกหัดที่ 9: Settings Dialog พร้อม Save/Load
- แบบฝึกหัดที่ 10: สร้าง Notepad Clone ที่มีครบทุก Features

---

## สรุปบทที่ 18

ในบทนี้เราได้เรียนรู้:

1. **TMainMenu** - Menu Bar หลักของ Application
2. **TPopupMenu** - Context Menu เมื่อคลิกขวา
3. **Menu Items** - Properties, Shortcuts, Icons, Dynamic Menus
4. **TOpenDialog, TSaveDialog** - Standard File Dialogs
5. **TFontDialog, TColorDialog** - ตัวเลือก Font และสี
6. **MessageDlg, InputBox** - Dialog ง่ายๆ
7. **Custom Dialogs** - Dialog ที่ออกแบบเอง
8. **Modal vs Non-Modal** - ความแตกต่างและการใช้งาน

บทถัดไปจะเรียนรู้เกี่ยวกับ **Layout Management** อย่างละเอียด
