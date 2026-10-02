# Part 20 - โปรเจคพื้นฐาน: Text Editor

## บทนำ

ในบทสุดท้ายนี้เราจะสร้าง **Text Editor** ที่สมบูรณ์ รวบรวมทุกทักษะที่ได้เรียนมาตั้งแต่ Part 16-19 โปรแกรมจะมีฟีเจอร์ครบถ้วนเทียบได้กับ Notepad ของ Windows

---

## 20.1 การออกแบบ Text Editor

### 20.1.1 Feature List

```
Features ที่จะพัฒนา:
━━━━━━━━━━━━━━━━━━━━

File Operations:
  ✓ New Document (Ctrl+N)
  ✓ Open File (Ctrl+O)
  ✓ Save (Ctrl+S)
  ✓ Save As (Ctrl+Shift+S)
  ✓ Recent Files (เก็บ 10 ไฟล์ล่าสุด)
  ✓ Print (Ctrl+P)
  ✓ Exit (Alt+F4)

Edit Operations:
  ✓ Undo (Ctrl+Z)
  ✓ Redo (Ctrl+Y)
  ✓ Cut (Ctrl+X)
  ✓ Copy (Ctrl+C)
  ✓ Paste (Ctrl+V)
  ✓ Select All (Ctrl+A)
  ✓ Find (Ctrl+F)
  ✓ Find Next (F3)
  ✓ Replace (Ctrl+H)
  ✓ Go To Line (Ctrl+G)

Format:
  ✓ Font
  ✓ Word Wrap Toggle

View:
  ✓ Toolbar Toggle
  ✓ Status Bar Toggle
  ✓ Zoom In/Out (Ctrl+=/-)

Extra:
  ✓ Line/Column indicator
  ✓ Character count
  ✓ Encoding display
  ✓ Tab to Spaces
  ✓ Auto-indent
```

### 20.1.2 Project Structure

```
TextEditor/
├── TextEditor.lpr          (Project file)
├── MainForm.pas            (Main editor form)
├── MainForm.lfm            (Form layout)
├── FindReplaceForm.pas     (Find/Replace dialog)
├── FindReplaceForm.lfm
├── GoToLineForm.pas        (Go To Line dialog)
├── GoToLineForm.lfm
├── AboutForm.pas           (About dialog)
├── AboutForm.lfm
├── EditorSettings.pas      (Settings management)
└── AppUtils.pas            (Utility functions)
```

---

## 20.2 Main Form - ไฟล์หลัก

### MainForm.pas - ส่วนที่ 1: Declaration

```pascal
unit MainForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls, Menus, Clipbrd, LCLType,
  LCLIntf, IniFiles, Printers, FindReplaceForm, GoToLineForm,
  AboutForm, EditorSettings;

const
  APP_NAME = 'Text Editor';
  APP_VERSION = '1.0.0';
  MAX_RECENT_FILES = 10;
  MAX_UNDO_LEVELS = 100;

type
  TEditorEncoding = (encANSI, encUTF8, encUTF16LE, encUTF16BE);

  TMainForm = class(TForm)
    // ===== Main Menu =====
    MainMenu1: TMainMenu;
    
    // File Menu
    MenuFile: TMenuItem;
    MenuFileNew: TMenuItem;
    MenuFileOpen: TMenuItem;
    MenuFileSave: TMenuItem;
    MenuFileSaveAs: TMenuItem;
    MenuFileSep1: TMenuItem;
    MenuFileRecentFiles: TMenuItem;
    MenuFileSep2: TMenuItem;
    MenuFilePrint: TMenuItem;
    MenuFilePrintPreview: TMenuItem;
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
    MenuEditSep3: TMenuItem;
    MenuEditFind: TMenuItem;
    MenuEditFindNext: TMenuItem;
    MenuEditReplace: TMenuItem;
    MenuEditGoTo: TMenuItem;
    MenuEditSep4: TMenuItem;
    MenuEditInsertDate: TMenuItem;
    
    // Format Menu
    MenuFormat: TMenuItem;
    MenuFormatWordWrap: TMenuItem;
    MenuFormatFont: TMenuItem;
    MenuFormatSep1: TMenuItem;
    MenuFormatTabWidth: TMenuItem;
    
    // View Menu
    MenuView: TMenuItem;
    MenuViewToolbar: TMenuItem;
    MenuViewStatusBar: TMenuItem;
    MenuViewLineNumbers: TMenuItem;
    MenuViewSep1: TMenuItem;
    MenuViewZoomIn: TMenuItem;
    MenuViewZoomOut: TMenuItem;
    MenuViewZoomReset: TMenuItem;
    
    // Help Menu
    MenuHelp: TMenuItem;
    MenuHelpContents: TMenuItem;
    MenuHelpSep1: TMenuItem;
    MenuHelpAbout: TMenuItem;
    
    // ===== Toolbar =====
    ToolBar1: TToolBar;
    ToolButtonNew: TToolButton;
    ToolButtonOpen: TToolButton;
    ToolButtonSave: TToolButton;
    ToolBarSep1: TToolButton;
    ToolButtonCut: TToolButton;
    ToolButtonCopy: TToolButton;
    ToolButtonPaste: TToolButton;
    ToolBarSep2: TToolButton;
    ToolButtonUndo: TToolButton;
    ToolButtonRedo: TToolButton;
    ToolBarSep3: TToolButton;
    ToolButtonFind: TToolButton;
    
    // ===== Main Editor =====
    Memo1: TMemo;
    
    // ===== Status Bar =====
    StatusBar1: TStatusBar;
    
    // ===== Popup Menu =====
    PopupMenuEditor: TPopupMenu;
    PopupUndo: TMenuItem;
    PopupSep1: TMenuItem;
    PopupCut: TMenuItem;
    PopupCopy: TMenuItem;
    PopupPaste: TMenuItem;
    PopupDelete: TMenuItem;
    PopupSep2: TMenuItem;
    PopupSelectAll: TMenuItem;
    
    // ===== Image Lists =====
    ImageList16: TImageList;
    
    // ===== Events =====
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormCloseQuery(Sender: TObject; var CanClose: Boolean);
    procedure FormShow(Sender: TObject);
    procedure FormResize(Sender: TObject);
    procedure FormDropFiles(Sender: TObject; const FileNames: array of String);
    
    // File Menu Events
    procedure MenuFileNewClick(Sender: TObject);
    procedure MenuFileOpenClick(Sender: TObject);
    procedure MenuFileSaveClick(Sender: TObject);
    procedure MenuFileSaveAsClick(Sender: TObject);
    procedure MenuFilePrintClick(Sender: TObject);
    procedure MenuFileExitClick(Sender: TObject);
    procedure MenuFileClick(Sender: TObject);   // Update before show
    
    // Edit Menu Events
    procedure MenuEditUndoClick(Sender: TObject);
    procedure MenuEditRedoClick(Sender: TObject);
    procedure MenuEditCutClick(Sender: TObject);
    procedure MenuEditCopyClick(Sender: TObject);
    procedure MenuEditPasteClick(Sender: TObject);
    procedure MenuEditDeleteClick(Sender: TObject);
    procedure MenuEditSelectAllClick(Sender: TObject);
    procedure MenuEditFindClick(Sender: TObject);
    procedure MenuEditFindNextClick(Sender: TObject);
    procedure MenuEditReplaceClick(Sender: TObject);
    procedure MenuEditGoToClick(Sender: TObject);
    procedure MenuEditInsertDateClick(Sender: TObject);
    procedure MenuEditClick(Sender: TObject);   // Update before show
    
    // Format Events
    procedure MenuFormatWordWrapClick(Sender: TObject);
    procedure MenuFormatFontClick(Sender: TObject);
    
    // View Events
    procedure MenuViewToolbarClick(Sender: TObject);
    procedure MenuViewStatusBarClick(Sender: TObject);
    procedure MenuViewZoomInClick(Sender: TObject);
    procedure MenuViewZoomOutClick(Sender: TObject);
    procedure MenuViewZoomResetClick(Sender: TObject);
    
    // Help Events
    procedure MenuHelpAboutClick(Sender: TObject);
    
    // Memo Events
    procedure Memo1Change(Sender: TObject);
    procedure Memo1KeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure Memo1KeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure Memo1Click(Sender: TObject);
    
    // Popup Events
    procedure PopupMenuEditorPopup(Sender: TObject);
    
  private
    // State
    FFileName: string;
    FModified: Boolean;
    FRecentFiles: TStringList;
    FFontSize: Integer;
    FEncoding: TEditorEncoding;
    FSettings: TEditorSettings;
    
    // Find/Replace State
    FFindText: string;
    FReplaceText: string;
    FFindMatchCase: Boolean;
    FFindWholeWord: Boolean;
    FFindForward: Boolean;
    
    // Form References
    FFindForm: TFindReplaceForm;
    
    // Private Methods
    procedure InitializeApp;
    procedure SetupMenus;
    procedure SetupToolbar;
    procedure SetupEditor;
    procedure SetupStatusBar;
    procedure SetupPopupMenu;
    procedure LoadSettings;
    procedure SaveSettings;
    
    // File Operations
    function NewDocument: Boolean;
    function OpenFile(const FileName: string = ''): Boolean;
    function SaveFile: Boolean;
    function SaveFileAs: Boolean;
    function AskSaveChanges: TModalResult;
    procedure LoadFileContent(const FileName: string);
    procedure SaveFileContent(const FileName: string);
    
    // Recent Files
    procedure AddRecentFile(const FileName: string);
    procedure RemoveRecentFile(const FileName: string);
    procedure UpdateRecentFilesMenu;
    procedure RecentFileClick(Sender: TObject);
    procedure LoadRecentFiles;
    procedure SaveRecentFiles;
    
    // Edit Operations
    function FindText(const SearchText: string; Forward: Boolean;
      MatchCase, WholeWord: Boolean): Boolean;
    procedure ReplaceText(const SearchText, ReplaceWith: string;
      ReplaceAll, MatchCase, WholeWord: Boolean);
    
    // State Management
    procedure SetModified(Value: Boolean);
    procedure UpdateTitle;
    procedure UpdateStatusBar;
    procedure UpdateMenuState;
    procedure SetZoom(Delta: Integer);
    function GetCurrentZoom: Integer;
    
    // Utility
    function GetLineAndColumn: TPoint;
    function GetSelectionInfo: string;
    function GetEncodingName: string;
    
    property Modified: Boolean read FModified write SetModified;
    
  public
    procedure OpenFileFromCommandLine(const FileName: string);
    procedure FindNext(const SearchText: string);
    procedure ReplaceAndFind(const SearchText, ReplaceWith: string;
      MatchCase, WholeWord: Boolean);
  end;

var
  MainForm: TMainForm;

implementation

{$R *.lfm}

uses
  FileUtil, LazFileUtils, StrUtils, Math;
```

### MainForm.pas - ส่วนที่ 2: Implementation

```pascal
// ===== Initialization =====

procedure TMainForm.FormCreate(Sender: TObject);
begin
  FSettings := TEditorSettings.Create;
  FRecentFiles := TStringList.Create;
  FFontSize := 11;
  FModified := False;
  FFileName := '';
  FEncoding := encUTF8;
  
  InitializeApp;
  LoadSettings;
  LoadRecentFiles;
  UpdateRecentFilesMenu;
  UpdateTitle;
  UpdateStatusBar;
  
  // Allow Drag & Drop Files
  AllowDropFiles := True;
end;

procedure TMainForm.InitializeApp;
begin
  Caption := APP_NAME;
  Width := 900;
  Height := 650;
  Position := poScreenCenter;
  KeyPreview := True;
  
  SetupMenus;
  SetupToolbar;
  SetupEditor;
  SetupStatusBar;
  SetupPopupMenu;
end;

procedure TMainForm.SetupEditor;
begin
  Memo1.Align := alClient;
  Memo1.ScrollBars := ssBoth;
  Memo1.WordWrap := False;
  Memo1.Font.Name := 'Courier New';
  Memo1.Font.Size := FFontSize;
  Memo1.Font.Color := clBlack;
  Memo1.Color := clWhite;
  Memo1.OnChange := Memo1Change;
  Memo1.OnKeyDown := Memo1KeyDown;
  Memo1.OnKeyUp := Memo1KeyUp;
  Memo1.OnClick := Memo1Click;
  Memo1.PopupMenu := PopupMenuEditor;
  Memo1.TabOrder := 0;
  
  // แสดงให้ชัดเจน
  Memo1.WantTabs := True; // รับ Tab Key
end;

procedure TMainForm.SetupStatusBar;
begin
  StatusBar1.Align := alBottom;
  StatusBar1.SimplePanel := False;
  
  with StatusBar1.Panels.Add do
  begin
    Width := 200;
    Text := 'บรรทัดที่ 1, คอลัมน์ที่ 1';
  end;
  
  with StatusBar1.Panels.Add do
  begin
    Width := 150;
    Text := '0 ตัวอักษร';
  end;
  
  with StatusBar1.Panels.Add do
  begin
    Width := 100;
    Text := 'UTF-8';
  end;
  
  with StatusBar1.Panels.Add do
  begin
    Width := 120;
    Alignment := taRightJustify;
    Text := 'พร้อมใช้งาน';
  end;
end;

// ===== Menu Setup =====

procedure TMainForm.SetupMenus;
begin
  // File Menu
  MenuFile.Caption := '&ไฟล์';
  SetMenuItem(MenuFileNew, '&ใหม่', 'Ctrl+N', MenuFileNewClick);
  SetMenuItem(MenuFileOpen, '&เปิด...', 'Ctrl+O', MenuFileOpenClick);
  SetMenuItem(MenuFileSave, '&บันทึก', 'Ctrl+S', MenuFileSaveClick);
  SetMenuItem(MenuFileSaveAs, 'บันทึก&เป็น...', 'Ctrl+Shift+S', MenuFileSaveAsClick);
  MenuFileSep1.Caption := '-';
  MenuFileRecentFiles.Caption := 'ไฟล์&ล่าสุด';
  MenuFileSep2.Caption := '-';
  SetMenuItem(MenuFilePrint, '&พิมพ์...', 'Ctrl+P', MenuFilePrintClick);
  MenuFileSep3.Caption := '-';
  SetMenuItem(MenuFileExit, 'อ&อก', 'Alt+F4', MenuFileExitClick);
  MenuFile.OnClick := MenuFileClick;
  
  // Edit Menu
  MenuEdit.Caption := '&แก้ไข';
  SetMenuItem(MenuEditUndo, '&ยกเลิก', 'Ctrl+Z', MenuEditUndoClick);
  SetMenuItem(MenuEditRedo, '&ทำซ้ำ', 'Ctrl+Y', MenuEditRedoClick);
  MenuEditSep1.Caption := '-';
  SetMenuItem(MenuEditCut, '&ตัด', 'Ctrl+X', MenuEditCutClick);
  SetMenuItem(MenuEditCopy, '&คัดลอก', 'Ctrl+C', MenuEditCopyClick);
  SetMenuItem(MenuEditPaste, '&วาง', 'Ctrl+V', MenuEditPasteClick);
  SetMenuItem(MenuEditDelete, '&ลบ', 'Del', MenuEditDeleteClick);
  MenuEditSep2.Caption := '-';
  SetMenuItem(MenuEditSelectAll, 'เลือก&ทั้งหมด', 'Ctrl+A', MenuEditSelectAllClick);
  MenuEditSep3.Caption := '-';
  SetMenuItem(MenuEditFind, '&ค้นหา...', 'Ctrl+F', MenuEditFindClick);
  SetMenuItem(MenuEditFindNext, 'ค้นหา&ถัดไป', 'F3', MenuEditFindNextClick);
  SetMenuItem(MenuEditReplace, 'ค้นหาและ&แทนที่...', 'Ctrl+H', MenuEditReplaceClick);
  SetMenuItem(MenuEditGoTo, 'ไปที่&บรรทัด...', 'Ctrl+G', MenuEditGoToClick);
  MenuEditSep4.Caption := '-';
  SetMenuItem(MenuEditInsertDate, 'แทรกวัน/เวลา', 'F5', MenuEditInsertDateClick);
  MenuEdit.OnClick := MenuEditClick;
  
  // Format Menu
  MenuFormat.Caption := '&รูปแบบ';
  MenuFormatWordWrap.Caption := '&ตัดบรรทัด';
  MenuFormatWordWrap.OnClick := MenuFormatWordWrapClick;
  SetMenuItem(MenuFormatFont, '&ตัวอักษร...', '', MenuFormatFontClick);
  
  // View Menu
  MenuView.Caption := '&มุมมอง';
  MenuViewToolbar.Caption := '&แถบเครื่องมือ';
  MenuViewToolbar.Checked := True;
  MenuViewToolbar.OnClick := MenuViewToolbarClick;
  MenuViewStatusBar.Caption := '&แถบสถานะ';
  MenuViewStatusBar.Checked := True;
  MenuViewStatusBar.OnClick := MenuViewStatusBarClick;
  MenuViewSep1.Caption := '-';
  SetMenuItem(MenuViewZoomIn, 'ขยาย', 'Ctrl+=', MenuViewZoomInClick);
  SetMenuItem(MenuViewZoomOut, 'ย่อ', 'Ctrl+-', MenuViewZoomOutClick);
  SetMenuItem(MenuViewZoomReset, 'ขนาดปกติ', 'Ctrl+0', MenuViewZoomResetClick);
  
  // Help Menu
  MenuHelp.Caption := '&ช่วยเหลือ';
  SetMenuItem(MenuHelpAbout, 'เกี่ยว&กับ...', '', MenuHelpAboutClick);
end;

procedure TMainForm.SetMenuItem(Item: TMenuItem; const ACaption, AShortcut: string;
  AHandler: TNotifyEvent);
begin
  Item.Caption := ACaption;
  if AShortcut <> '' then
    Item.ShortCut := TextToShortCut(AShortcut);
  Item.OnClick := AHandler;
end;

// ===== File Operations =====

function TMainForm.NewDocument: Boolean;
begin
  Result := True;
  
  if FModified then
  begin
    case AskSaveChanges of
      mrYes: if not SaveFile then begin Result := False; Exit; end;
      mrCancel: begin Result := False; Exit; end;
    end;
  end;
  
  Memo1.Clear;
  FFileName := '';
  Modified := False;
  FFontSize := 11;
  Memo1.Font.Size := FFontSize;
  UpdateTitle;
  UpdateStatusBar;
end;

function TMainForm.OpenFile(const FileName: string = ''): Boolean;
var
  OpenDlg: TOpenDialog;
  fileToOpen: string;
begin
  Result := False;
  
  if FModified then
  begin
    case AskSaveChanges of
      mrYes: if not SaveFile then Exit;
      mrCancel: Exit;
    end;
  end;
  
  if FileName = '' then
  begin
    OpenDlg := TOpenDialog.Create(Self);
    try
      OpenDlg.Title := 'เปิดไฟล์';
      OpenDlg.Filter := 
        'ไฟล์ข้อความ|*.txt;*.text|' +
        'Pascal Source|*.pas;*.pp;*.inc|' +
        'HTML Files|*.html;*.htm|' +
        'XML Files|*.xml|' +
        'CSV Files|*.csv|' +
        'ทุกไฟล์|*.*';
      OpenDlg.Options := [ofFileMustExist, ofPathMustExist];
      
      if not OpenDlg.Execute then Exit;
      fileToOpen := OpenDlg.FileName;
    finally
      OpenDlg.Free;
    end;
  end
  else
    fileToOpen := FileName;
  
  try
    LoadFileContent(fileToOpen);
    FFileName := fileToOpen;
    Modified := False;
    AddRecentFile(fileToOpen);
    UpdateTitle;
    UpdateStatusBar;
    StatusBar1.Panels[3].Text := 'เปิด: ' + ExtractFileName(fileToOpen);
    Result := True;
  except
    on E: Exception do
    begin
      ShowMessage('ไม่สามารถเปิดไฟล์:' + #13#10 + E.Message);
      RemoveRecentFile(fileToOpen);
    end;
  end;
end;

procedure TMainForm.LoadFileContent(const FileName: string);
var
  stream: TFileStream;
  bom: array[0..2] of Byte;
  bytesRead: Integer;
begin
  stream := TFileStream.Create(FileName, fmOpenRead or fmShareDenyNone);
  try
    // ตรวจสอบ BOM
    bytesRead := stream.Read(bom, 3);
    stream.Position := 0;
    
    if (bytesRead >= 3) and (bom[0] = $EF) and (bom[1] = $BB) and (bom[2] = $BF) then
    begin
      FEncoding := encUTF8;
      stream.Position := 3; // ข้าม BOM
    end
    else if (bytesRead >= 2) and (bom[0] = $FF) and (bom[1] = $FE) then
    begin
      FEncoding := encUTF16LE;
      stream.Position := 2;
    end
    else if (bytesRead >= 2) and (bom[0] = $FE) and (bom[1] = $FF) then
    begin
      FEncoding := encUTF16BE;
      stream.Position := 2;
    end
    else
    begin
      FEncoding := encUTF8; // สมมติเป็น UTF-8
      stream.Position := 0;
    end;
    
    Memo1.Lines.LoadFromStream(stream);
  finally
    stream.Free;
  end;
  
  StatusBar1.Panels[2].Text := GetEncodingName;
end;

function TMainForm.SaveFile: Boolean;
begin
  if FFileName = '' then
    Result := SaveFileAs
  else
  begin
    try
      SaveFileContent(FFileName);
      Modified := False;
      StatusBar1.Panels[3].Text := 'บันทึกแล้ว';
      Result := True;
    except
      on E: Exception do
      begin
        ShowMessage('ไม่สามารถบันทึกไฟล์:' + #13#10 + E.Message);
        Result := False;
      end;
    end;
  end;
end;

function TMainForm.SaveFileAs: Boolean;
var
  SaveDlg: TSaveDialog;
begin
  Result := False;
  
  SaveDlg := TSaveDialog.Create(Self);
  try
    SaveDlg.Title := 'บันทึกไฟล์เป็น';
    SaveDlg.Filter := 
      'ไฟล์ข้อความ|*.txt|' +
      'Pascal Source|*.pas|' +
      'ทุกไฟล์|*.*';
    SaveDlg.DefaultExt := 'txt';
    if FFileName <> '' then
    begin
      SaveDlg.InitialDir := ExtractFilePath(FFileName);
      SaveDlg.FileName := ExtractFileName(FFileName);
    end;
    SaveDlg.Options := [ofOverwritePrompt];
    
    if SaveDlg.Execute then
    begin
      try
        SaveFileContent(SaveDlg.FileName);
        FFileName := SaveDlg.FileName;
        Modified := False;
        AddRecentFile(FFileName);
        UpdateTitle;
        StatusBar1.Panels[3].Text := 'บันทึกแล้ว';
        Result := True;
      except
        on E: Exception do
          ShowMessage('ไม่สามารถบันทึก:' + #13#10 + E.Message);
      end;
    end;
  finally
    SaveDlg.Free;
  end;
end;

procedure TMainForm.SaveFileContent(const FileName: string);
var
  stream: TFileStream;
  bom: array[0..2] of Byte;
begin
  stream := TFileStream.Create(FileName, fmCreate);
  try
    // เขียน BOM สำหรับ UTF-8
    if FEncoding = encUTF8 then
    begin
      bom[0] := $EF;
      bom[1] := $BB;
      bom[2] := $BF;
      stream.Write(bom, 3);
    end;
    
    Memo1.Lines.SaveToStream(stream);
  finally
    stream.Free;
  end;
end;

function TMainForm.AskSaveChanges: TModalResult;
var
  fileDesc: string;
begin
  if FFileName = '' then
    fileDesc := 'ไม่มีชื่อ'
  else
    fileDesc := ExtractFileName(FFileName);
    
  Result := MessageDlg(
    'ไฟล์ "' + fileDesc + '" ถูกแก้ไข' + #13#10 +
    'ต้องการบันทึกการเปลี่ยนแปลงหรือไม่?',
    mtConfirmation,
    [mbYes, mbNo, mbCancel],
    0
  );
end;

// ===== Recent Files =====

procedure TMainForm.AddRecentFile(const FileName: string);
var
  idx: Integer;
begin
  // ลบถ้ามีอยู่แล้ว (จะเพิ่มที่ต้นรายการ)
  idx := FRecentFiles.IndexOf(FileName);
  if idx >= 0 then
    FRecentFiles.Delete(idx);
  
  // เพิ่มที่ต้น
  FRecentFiles.Insert(0, FileName);
  
  // จำกัดจำนวน
  while FRecentFiles.Count > MAX_RECENT_FILES do
    FRecentFiles.Delete(FRecentFiles.Count - 1);
  
  UpdateRecentFilesMenu;
  SaveRecentFiles;
end;

procedure TMainForm.RemoveRecentFile(const FileName: string);
var
  idx: Integer;
begin
  idx := FRecentFiles.IndexOf(FileName);
  if idx >= 0 then
  begin
    FRecentFiles.Delete(idx);
    UpdateRecentFilesMenu;
    SaveRecentFiles;
  end;
end;

procedure TMainForm.UpdateRecentFilesMenu;
var
  i: Integer;
  item: TMenuItem;
begin
  MenuFileRecentFiles.Clear;
  
  if FRecentFiles.Count = 0 then
  begin
    item := TMenuItem.Create(MenuFileRecentFiles);
    item.Caption := '(ไม่มีไฟล์ล่าสุด)';
    item.Enabled := False;
    MenuFileRecentFiles.Add(item);
    Exit;
  end;
  
  for i := 0 to FRecentFiles.Count - 1 do
  begin
    item := TMenuItem.Create(MenuFileRecentFiles);
    item.Caption := '&' + IntToStr(i + 1) + ' ' + 
                    ShortenPath(FRecentFiles[i], 50);
    item.Hint := FRecentFiles[i];
    item.Tag := i;
    item.OnClick := RecentFileClick;
    MenuFileRecentFiles.Add(item);
  end;
  
  // Separator และ Clear
  item := TMenuItem.Create(MenuFileRecentFiles);
  item.Caption := '-';
  MenuFileRecentFiles.Add(item);
  
  item := TMenuItem.Create(MenuFileRecentFiles);
  item.Caption := 'ล้างรายการ';
  item.OnClick := procedure(Sender: TObject)
  begin
    if MessageDlg('ล้างรายการไฟล์ล่าสุดหรือไม่?',
                  mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    begin
      FRecentFiles.Clear;
      UpdateRecentFilesMenu;
      SaveRecentFiles;
    end;
  end;
  MenuFileRecentFiles.Add(item);
end;

procedure TMainForm.RecentFileClick(Sender: TObject);
var
  fileName: string;
begin
  fileName := FRecentFiles[TMenuItem(Sender).Tag];
  
  if not FileExists(fileName) then
  begin
    if MessageDlg('ไม่พบไฟล์: ' + fileName + #13#10 +
                  'ต้องการลบออกจากรายการหรือไม่?',
                  mtConfirmation, [mbYes, mbNo], 0) = mrYes then
      RemoveRecentFile(fileName);
    Exit;
  end;
  
  OpenFile(fileName);
end;

// ===== Edit Operations =====

function TMainForm.FindText(const SearchText: string; Forward: Boolean;
  MatchCase, WholeWord: Boolean): Boolean;
var
  searchIn: string;
  searchFor: string;
  startPos: Integer;
  foundPos: Integer;
begin
  Result := False;
  if SearchText = '' then Exit;
  
  if MatchCase then
  begin
    searchIn := Memo1.Text;
    searchFor := SearchText;
  end
  else
  begin
    searchIn := LowerCase(Memo1.Text);
    searchFor := LowerCase(SearchText);
  end;
  
  if Forward then
    startPos := Memo1.SelStart + Memo1.SelLength
  else
    startPos := Memo1.SelStart - 1;
  
  if Forward then
  begin
    foundPos := PosEx(searchFor, searchIn, startPos + 1);
    if foundPos = 0 then
    begin
      // Wrap around
      foundPos := Pos(searchFor, searchIn);
      if foundPos > 0 then
        StatusBar1.Panels[3].Text := 'กลับไปต้นเอกสาร';
    end;
  end
  else
  begin
    // Find backwards
    foundPos := 0;
    var p := 1;
    repeat
      var fp := PosEx(searchFor, searchIn, p);
      if (fp > 0) and (fp < startPos + 1) then
      begin
        foundPos := fp;
        p := fp + 1;
      end
      else
        Break;
    until False;
  end;
  
  if foundPos > 0 then
  begin
    Memo1.SelStart := foundPos - 1;
    Memo1.SelLength := Length(SearchText);
    Memo1.SetFocus;
    
    // Scroll ให้เห็น Selection
    SendMessage(Memo1.Handle, EM_SCROLLCARET, 0, 0);
    
    Result := True;
  end;
end;

procedure TMainForm.ReplaceText(const SearchText, ReplaceWith: string;
  ReplaceAll, MatchCase, WholeWord: Boolean);
var
  count: Integer;
  searchIn, searchFor: string;
  pos, offset: Integer;
  newText: string;
begin
  count := 0;
  
  if ReplaceAll then
  begin
    if MatchCase then
    begin
      searchIn := Memo1.Text;
      searchFor := SearchText;
    end
    else
    begin
      searchIn := LowerCase(Memo1.Text);
      searchFor := LowerCase(SearchText);
    end;
    
    newText := '';
    offset := 1;
    
    repeat
      pos := PosEx(searchFor, searchIn, offset);
      if pos > 0 then
      begin
        newText := newText + Copy(Memo1.Text, offset, pos - offset) + ReplaceWith;
        offset := pos + Length(SearchText);
        Inc(count);
      end
      else
      begin
        newText := newText + Copy(Memo1.Text, offset, MaxInt);
        Break;
      end;
    until False;
    
    if count > 0 then
    begin
      Memo1.Text := newText;
      Modified := True;
      ShowMessage('แทนที่แล้ว ' + IntToStr(count) + ' ครั้ง');
    end
    else
      ShowMessage('ไม่พบ "' + SearchText + '"');
  end
  else
  begin
    // Replace Current Selection
    if SameText(Memo1.SelText, SearchText) or
       (not MatchCase and (LowerCase(Memo1.SelText) = LowerCase(SearchText))) then
    begin
      Memo1.SelText := ReplaceWith;
      Modified := True;
    end;
    
    // Find Next
    FindText(SearchText, FFindForward, MatchCase, WholeWord);
  end;
end;

// ===== State Management =====

procedure TMainForm.SetModified(Value: Boolean);
begin
  if FModified <> Value then
  begin
    FModified := Value;
    UpdateTitle;
  end;
end;

procedure TMainForm.UpdateTitle;
var
  fileDesc: string;
begin
  if FFileName = '' then
    fileDesc := 'ไม่มีชื่อ'
  else
    fileDesc := ExtractFileName(FFileName);
    
  if FModified then
    Caption := '* ' + fileDesc + ' - ' + APP_NAME
  else
    Caption := fileDesc + ' - ' + APP_NAME;
end;

procedure TMainForm.UpdateStatusBar;
var
  lineCol: TPoint;
  charCount: Integer;
begin
  lineCol := GetLineAndColumn;
  
  StatusBar1.Panels[0].Text := 
    'บรรทัดที่ ' + IntToStr(lineCol.Y) + 
    ', คอลัมน์ที่ ' + IntToStr(lineCol.X);
  
  charCount := Length(Memo1.Text);
  if Memo1.SelLength > 0 then
    StatusBar1.Panels[1].Text := 
      IntToStr(Memo1.SelLength) + '/' + IntToStr(charCount) + ' ตัวอักษร'
  else
    StatusBar1.Panels[1].Text := IntToStr(charCount) + ' ตัวอักษร';
  
  StatusBar1.Panels[2].Text := GetEncodingName;
end;

function TMainForm.GetLineAndColumn: TPoint;
var
  selStart: Integer;
  text: string;
  i, line, col: Integer;
begin
  selStart := Memo1.SelStart;
  text := Memo1.Text;
  
  line := 1;
  col := 1;
  
  for i := 1 to Min(selStart, Length(text)) do
  begin
    if text[i] = #10 then
    begin
      Inc(line);
      col := 1;
    end
    else if text[i] <> #13 then
      Inc(col);
  end;
  
  Result := Point(col, line);
end;

function TMainForm.GetEncodingName: string;
begin
  case FEncoding of
    encANSI:   Result := 'ANSI';
    encUTF8:   Result := 'UTF-8';
    encUTF16LE: Result := 'UTF-16 LE';
    encUTF16BE: Result := 'UTF-16 BE';
    else Result := 'Unknown';
  end;
end;

procedure TMainForm.SetZoom(Delta: Integer);
begin
  FFontSize := Max(6, Min(72, FFontSize + Delta));
  Memo1.Font.Size := FFontSize;
  StatusBar1.Panels[3].Text := 'ขนาด: ' + IntToStr(FFontSize) + 'pt';
end;

// ===== Settings =====

procedure TMainForm.LoadSettings;
var
  ini: TIniFile;
  settingsFile: string;
begin
  settingsFile := GetAppConfigDir(False) + 'texteditor.ini';
  
  if not FileExists(settingsFile) then Exit;
  
  ini := TIniFile.Create(settingsFile);
  try
    // Window Position
    Left := ini.ReadInteger('Window', 'Left', Left);
    Top := ini.ReadInteger('Window', 'Top', Top);
    Width := ini.ReadInteger('Window', 'Width', Width);
    Height := ini.ReadInteger('Window', 'Height', Height);
    
    // Editor Settings
    FFontSize := ini.ReadInteger('Editor', 'FontSize', 11);
    Memo1.Font.Name := ini.ReadString('Editor', 'FontName', 'Courier New');
    Memo1.Font.Size := FFontSize;
    
    var wordWrap := ini.ReadBool('Editor', 'WordWrap', False);
    Memo1.WordWrap := wordWrap;
    MenuFormatWordWrap.Checked := wordWrap;
    if wordWrap then
      Memo1.ScrollBars := ssVertical
    else
      Memo1.ScrollBars := ssBoth;
    
    // View Settings
    var showToolbar := ini.ReadBool('View', 'Toolbar', True);
    ToolBar1.Visible := showToolbar;
    MenuViewToolbar.Checked := showToolbar;
    
    var showStatus := ini.ReadBool('View', 'StatusBar', True);
    StatusBar1.Visible := showStatus;
    MenuViewStatusBar.Checked := showStatus;
  finally
    ini.Free;
  end;
end;

procedure TMainForm.SaveSettings;
var
  ini: TIniFile;
  settingsFile: string;
  settingsDir: string;
begin
  settingsDir := GetAppConfigDir(False);
  if not DirectoryExists(settingsDir) then
    CreateDir(settingsDir);
    
  settingsFile := settingsDir + 'texteditor.ini';
  
  ini := TIniFile.Create(settingsFile);
  try
    ini.WriteInteger('Window', 'Left', Left);
    ini.WriteInteger('Window', 'Top', Top);
    ini.WriteInteger('Window', 'Width', Width);
    ini.WriteInteger('Window', 'Height', Height);
    
    ini.WriteInteger('Editor', 'FontSize', FFontSize);
    ini.WriteString('Editor', 'FontName', Memo1.Font.Name);
    ini.WriteBool('Editor', 'WordWrap', Memo1.WordWrap);
    
    ini.WriteBool('View', 'Toolbar', ToolBar1.Visible);
    ini.WriteBool('View', 'StatusBar', StatusBar1.Visible);
  finally
    ini.Free;
  end;
end;

procedure TMainForm.LoadRecentFiles;
var
  ini: TIniFile;
  settingsFile: string;
  i: Integer;
  fileName: string;
begin
  settingsFile := GetAppConfigDir(False) + 'texteditor.ini';
  if not FileExists(settingsFile) then Exit;
  
  ini := TIniFile.Create(settingsFile);
  try
    FRecentFiles.Clear;
    for i := 0 to MAX_RECENT_FILES - 1 do
    begin
      fileName := ini.ReadString('RecentFiles', 'File' + IntToStr(i), '');
      if (fileName <> '') and FileExists(fileName) then
        FRecentFiles.Add(fileName);
    end;
  finally
    ini.Free;
  end;
end;

procedure TMainForm.SaveRecentFiles;
var
  ini: TIniFile;
  settingsFile: string;
  i: Integer;
begin
  settingsFile := GetAppConfigDir(False) + 'texteditor.ini';
  ini := TIniFile.Create(settingsFile);
  try
    ini.EraseSection('RecentFiles');
    for i := 0 to FRecentFiles.Count - 1 do
      ini.WriteString('RecentFiles', 'File' + IntToStr(i), FRecentFiles[i]);
  finally
    ini.Free;
  end;
end;

// ===== Event Handlers =====

procedure TMainForm.FormCloseQuery(Sender: TObject; var CanClose: Boolean);
begin
  CanClose := True;
  if FModified then
  begin
    case AskSaveChanges of
      mrYes: if not SaveFile then CanClose := False;
      mrCancel: CanClose := False;
    end;
  end;
  
  if CanClose then
  begin
    SaveSettings;
    SaveRecentFiles;
  end;
end;

procedure TMainForm.FormDestroy(Sender: TObject);
begin
  FRecentFiles.Free;
  FSettings.Free;
  if Assigned(FFindForm) then
    FFindForm.Free;
end;

procedure TMainForm.FormShow(Sender: TObject);
begin
  Memo1.SetFocus;
  
  // ตรวจสอบ Command Line Arguments
  if ParamCount > 0 then
    OpenFile(ParamStr(1));
end;

procedure TMainForm.FormResize(Sender: TObject);
begin
  // ไม่ต้องทำอะไรพิเศษ เพราะใช้ Align
end;

procedure TMainForm.FormDropFiles(Sender: TObject; const FileNames: array of String);
begin
  if Length(FileNames) > 0 then
    OpenFile(FileNames[0]);
end;

// File Menu
procedure TMainForm.MenuFileNewClick(Sender: TObject);
begin
  NewDocument;
end;

procedure TMainForm.MenuFileOpenClick(Sender: TObject);
begin
  OpenFile;
end;

procedure TMainForm.MenuFileSaveClick(Sender: TObject);
begin
  SaveFile;
end;

procedure TMainForm.MenuFileSaveAsClick(Sender: TObject);
begin
  SaveFileAs;
end;

procedure TMainForm.MenuFilePrintClick(Sender: TObject);
var
  printDlg: TPrintDialog;
begin
  printDlg := TPrintDialog.Create(Self);
  try
    if printDlg.Execute then
    begin
      Printer.BeginDoc;
      try
        var y := 100;
        var i: Integer;
        for i := 0 to Memo1.Lines.Count - 1 do
        begin
          Printer.Canvas.TextOut(100, y, Memo1.Lines[i]);
          Inc(y, Printer.Canvas.TextHeight(Memo1.Lines[i]) + 5);
          
          // ขึ้นหน้าใหม่ถ้าจำเป็น
          if y > Printer.PageHeight - 100 then
          begin
            Printer.NewPage;
            y := 100;
          end;
        end;
      finally
        Printer.EndDoc;
      end;
    end;
  finally
    printDlg.Free;
  end;
end;

procedure TMainForm.MenuFileExitClick(Sender: TObject);
begin
  Close;
end;

procedure TMainForm.MenuFileClick(Sender: TObject);
begin
  MenuFileSave.Enabled := FModified;
end;

// Edit Menu
procedure TMainForm.MenuEditUndoClick(Sender: TObject);
begin
  Memo1.Undo;
end;

procedure TMainForm.MenuEditRedoClick(Sender: TObject);
begin
  // Memo ใน Lazarus อาจไม่รองรับ Redo โดยตรง
  // ใช้ Clipboard หรือ History แทน
end;

procedure TMainForm.MenuEditCutClick(Sender: TObject);
begin
  Memo1.CutToClipboard;
end;

procedure TMainForm.MenuEditCopyClick(Sender: TObject);
begin
  Memo1.CopyToClipboard;
end;

procedure TMainForm.MenuEditPasteClick(Sender: TObject);
begin
  Memo1.PasteFromClipboard;
end;

procedure TMainForm.MenuEditDeleteClick(Sender: TObject);
begin
  Memo1.ClearSelection;
end;

procedure TMainForm.MenuEditSelectAllClick(Sender: TObject);
begin
  Memo1.SelectAll;
end;

procedure TMainForm.MenuEditFindClick(Sender: TObject);
begin
  if not Assigned(FFindForm) then
  begin
    FFindForm := TFindReplaceForm.Create(Self);
    FFindForm.ShowReplaceControls := False;
    FFindForm.OnFind := procedure(SearchText: string; 
      Forward, MatchCase, WholeWord: Boolean)
    begin
      FFindText := SearchText;
      FFindForward := Forward;
      FFindMatchCase := MatchCase;
      FFindWholeWord := WholeWord;
      
      if not FindText(SearchText, Forward, MatchCase, WholeWord) then
        ShowMessage('"' + SearchText + '" ไม่พบ');
    end;
  end;
  
  FFindForm.SetSearchText(Memo1.SelText);
  FFindForm.Show;
  FFindForm.SetFocus;
end;

procedure TMainForm.MenuEditFindNextClick(Sender: TObject);
begin
  if FFindText <> '' then
  begin
    if not FindText(FFindText, True, FFindMatchCase, FFindWholeWord) then
      ShowMessage('"' + FFindText + '" ไม่พบอีกแล้ว');
  end
  else
    MenuEditFindClick(nil);
end;

procedure TMainForm.MenuEditReplaceClick(Sender: TObject);
begin
  if not Assigned(FFindForm) then
    FFindForm := TFindReplaceForm.Create(Self);
    
  FFindForm.ShowReplaceControls := True;
  FFindForm.OnFind := procedure(SearchText: string;
    Forward, MatchCase, WholeWord: Boolean)
  begin
    FindText(SearchText, Forward, MatchCase, WholeWord);
  end;
  FFindForm.OnReplace := procedure(SearchText, ReplaceWith: string;
    ReplaceAll, MatchCase, WholeWord: Boolean)
  begin
    ReplaceText(SearchText, ReplaceWith, ReplaceAll, MatchCase, WholeWord);
  end;
  
  FFindForm.Show;
end;

procedure TMainForm.MenuEditGoToClick(Sender: TObject);
var
  lineNum: string;
  targetLine: Integer;
  i: Integer;
  pos: Integer;
begin
  lineNum := '';
  if InputQuery('ไปที่บรรทัด', 
                'บรรทัดที่ (1-' + IntToStr(Memo1.Lines.Count) + '):', 
                lineNum) then
  begin
    if TryStrToInt(lineNum, targetLine) then
    begin
      if (targetLine >= 1) and (targetLine <= Memo1.Lines.Count) then
      begin
        // คำนวณ Position ของบรรทัดที่ต้องการ
        pos := 0;
        for i := 0 to targetLine - 2 do
          pos := pos + Length(Memo1.Lines[i]) + Length(LineEnding);
          
        Memo1.SelStart := pos;
        Memo1.SelLength := 0;
        Memo1.SetFocus;
        UpdateStatusBar;
      end
      else
        ShowMessage('บรรทัดที่ ' + lineNum + ' ไม่มีในเอกสาร');
    end
    else
      ShowMessage('กรุณากรอกตัวเลข');
  end;
end;

procedure TMainForm.MenuEditInsertDateClick(Sender: TObject);
begin
  Memo1.SelText := FormatDateTime('hh:nn dd/mm/yyyy', Now);
  Modified := True;
end;

procedure TMainForm.MenuEditClick(Sender: TObject);
begin
  MenuEditUndo.Enabled := Memo1.CanUndo;
  MenuEditCut.Enabled := Memo1.SelLength > 0;
  MenuEditCopy.Enabled := Memo1.SelLength > 0;
  MenuEditPaste.Enabled := Clipboard.HasFormat(CF_TEXT);
  MenuEditDelete.Enabled := Memo1.SelLength > 0;
end;

// Format Menu
procedure TMainForm.MenuFormatWordWrapClick(Sender: TObject);
begin
  MenuFormatWordWrap.Checked := not MenuFormatWordWrap.Checked;
  Memo1.WordWrap := MenuFormatWordWrap.Checked;
  
  if Memo1.WordWrap then
    Memo1.ScrollBars := ssVertical
  else
    Memo1.ScrollBars := ssBoth;
end;

procedure TMainForm.MenuFormatFontClick(Sender: TObject);
var
  fontDlg: TFontDialog;
begin
  fontDlg := TFontDialog.Create(Self);
  try
    fontDlg.Font.Assign(Memo1.Font);
    fontDlg.Options := [fdEffects];
    
    if fontDlg.Execute then
    begin
      Memo1.Font.Assign(fontDlg.Font);
      FFontSize := Memo1.Font.Size;
    end;
  finally
    fontDlg.Free;
  end;
end;

// View Menu
procedure TMainForm.MenuViewToolbarClick(Sender: TObject);
begin
  MenuViewToolbar.Checked := not MenuViewToolbar.Checked;
  ToolBar1.Visible := MenuViewToolbar.Checked;
end;

procedure TMainForm.MenuViewStatusBarClick(Sender: TObject);
begin
  MenuViewStatusBar.Checked := not MenuViewStatusBar.Checked;
  StatusBar1.Visible := MenuViewStatusBar.Checked;
end;

procedure TMainForm.MenuViewZoomInClick(Sender: TObject);
begin
  SetZoom(2);
end;

procedure TMainForm.MenuViewZoomOutClick(Sender: TObject);
begin
  SetZoom(-2);
end;

procedure TMainForm.MenuViewZoomResetClick(Sender: TObject);
begin
  FFontSize := 11;
  Memo1.Font.Size := FFontSize;
  StatusBar1.Panels[3].Text := 'ขนาดปกติ';
end;

// Help Menu
procedure TMainForm.MenuHelpAboutClick(Sender: TObject);
begin
  with TAboutForm.Create(Self) do
  try
    ShowModal;
  finally
    Free;
  end;
end;

// Memo Events
procedure TMainForm.Memo1Change(Sender: TObject);
begin
  Modified := True;
  UpdateStatusBar;
end;

procedure TMainForm.Memo1KeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  // Global Shortcuts ที่ไม่อยู่ใน Menu
  if ssCtrl in Shift then
  begin
    case Key of
      Ord('='): begin SetZoom(2); Key := 0; end;
      Ord('-'): begin SetZoom(-2); Key := 0; end;
      Ord('0'): begin
        FFontSize := 11;
        Memo1.Font.Size := FFontSize;
        Key := 0;
      end;
    end;
  end;
  
  // Tab Key - แทรก Spaces แทน Tab
  if Key = VK_TAB then
  begin
    Memo1.SelText := '    '; // 4 Spaces
    Key := 0;
  end;
end;

procedure TMainForm.Memo1KeyUp(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  UpdateStatusBar;
end;

procedure TMainForm.Memo1Click(Sender: TObject);
begin
  UpdateStatusBar;
end;

// Popup Menu
procedure TMainForm.PopupMenuEditorPopup(Sender: TObject);
begin
  PopupUndo.Enabled := Memo1.CanUndo;
  PopupCut.Enabled := Memo1.SelLength > 0;
  PopupCopy.Enabled := Memo1.SelLength > 0;
  PopupPaste.Enabled := Clipboard.HasFormat(CF_TEXT);
  PopupDelete.Enabled := Memo1.SelLength > 0;
end;

// Public
procedure TMainForm.OpenFileFromCommandLine(const FileName: string);
begin
  if FileExists(FileName) then
    OpenFile(FileName);
end;

procedure TMainForm.FindNext(const SearchText: string);
begin
  FFindText := SearchText;
  if not FindText(SearchText, True, FFindMatchCase, FFindWholeWord) then
    ShowMessage('"' + SearchText + '" ไม่พบ');
end;

procedure TMainForm.ReplaceAndFind(const SearchText, ReplaceWith: string;
  MatchCase, WholeWord: Boolean);
begin
  ReplaceText(SearchText, ReplaceWith, False, MatchCase, WholeWord);
end;

end.
```

---

## 20.3 Find/Replace Form

```pascal
unit FindReplaceForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls;

type
  TFindEvent = procedure(SearchText: string; 
    Forward, MatchCase, WholeWord: Boolean) of object;
  TReplaceEvent = procedure(SearchText, ReplaceWith: string;
    ReplaceAll, MatchCase, WholeWord: Boolean) of object;

  TFindReplaceForm = class(TForm)
    LabelFind: TLabel;
    EditFind: TEdit;
    LabelReplace: TLabel;
    EditReplace: TEdit;
    
    GroupBoxOptions: TGroupBox;
    CheckMatchCase: TCheckBox;
    CheckWholeWord: TCheckBox;
    CheckSearchUp: TCheckBox;
    
    ButtonFindNext: TButton;
    ButtonReplace: TButton;
    ButtonReplaceAll: TButton;
    ButtonClose: TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure ButtonFindNextClick(Sender: TObject);
    procedure ButtonReplaceClick(Sender: TObject);
    procedure ButtonReplaceAllClick(Sender: TObject);
    procedure ButtonCloseClick(Sender: TObject);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    
  private
    FShowReplace: Boolean;
    FOnFind: TFindEvent;
    FOnReplace: TReplaceEvent;
    
    procedure SetShowReplaceControls(Value: Boolean);
    
  public
    procedure SetSearchText(const Text: string);
    property ShowReplaceControls: Boolean read FShowReplace 
      write SetShowReplaceControls;
    property OnFind: TFindEvent read FOnFind write FOnFind;
    property OnReplace: TReplaceEvent read FOnReplace write FOnReplace;
  end;

implementation

{$R *.lfm}

uses LCLType;

procedure TFindReplaceForm.FormCreate(Sender: TObject);
begin
  Caption := 'ค้นหา';
  BorderStyle := bsToolWindow;
  Width := 400;
  Height := 220;
  KeyPreview := True;
  
  SetShowReplaceControls(False);
  
  ButtonFindNext.Caption := 'ค้นหาถัดไป';
  ButtonFindNext.Default := True;
  ButtonReplace.Caption := 'แทนที่';
  ButtonReplaceAll.Caption := 'แทนที่ทั้งหมด';
  ButtonClose.Caption := 'ปิด';
  ButtonClose.Cancel := True;
  
  CheckMatchCase.Caption := 'ตรงตัวพิมพ์ใหญ่-เล็ก';
  CheckWholeWord.Caption := 'คำทั้งคำ';
  CheckSearchUp.Caption := 'ค้นหาขึ้น';
end;

procedure TFindReplaceForm.SetShowReplaceControls(Value: Boolean);
begin
  FShowReplace := Value;
  
  LabelReplace.Visible := Value;
  EditReplace.Visible := Value;
  ButtonReplace.Visible := Value;
  ButtonReplaceAll.Visible := Value;
  
  if Value then
  begin
    Caption := 'ค้นหาและแทนที่';
    Height := 250;
  end
  else
  begin
    Caption := 'ค้นหา';
    Height := 220;
  end;
end;

procedure TFindReplaceForm.SetSearchText(const Text: string);
begin
  if Text <> '' then
  begin
    EditFind.Text := Text;
    EditFind.SelectAll;
  end;
end;

procedure TFindReplaceForm.ButtonFindNextClick(Sender: TObject);
begin
  if EditFind.Text <> '' then
  begin
    if Assigned(FOnFind) then
      FOnFind(
        EditFind.Text,
        not CheckSearchUp.Checked,
        CheckMatchCase.Checked,
        CheckWholeWord.Checked
      );
  end
  else
    EditFind.SetFocus;
end;

procedure TFindReplaceForm.ButtonReplaceClick(Sender: TObject);
begin
  if EditFind.Text <> '' then
  begin
    if Assigned(FOnReplace) then
      FOnReplace(
        EditFind.Text,
        EditReplace.Text,
        False,
        CheckMatchCase.Checked,
        CheckWholeWord.Checked
      );
  end;
end;

procedure TFindReplaceForm.ButtonReplaceAllClick(Sender: TObject);
begin
  if EditFind.Text <> '' then
  begin
    if Assigned(FOnReplace) then
      FOnReplace(
        EditFind.Text,
        EditReplace.Text,
        True,
        CheckMatchCase.Checked,
        CheckWholeWord.Checked
      );
  end;
end;

procedure TFindReplaceForm.ButtonCloseClick(Sender: TObject);
begin
  Hide;
end;

procedure TFindReplaceForm.FormKeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  if Key = VK_ESCAPE then
  begin
    Hide;
    Key := 0;
  end;
end;

end.
```

---

## 20.4 About Form

```pascal
unit AboutForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, StdCtrls, ExtCtrls;

type
  TAboutForm = class(TForm)
    PanelTop: TPanel;
    ImageLogo: TImage;
    LabelAppName: TLabel;
    LabelVersion: TLabel;
    Bevel1: TBevel;
    LabelDescription: TLabel;
    LabelCopyright: TLabel;
    LabelBuiltWith: TLabel;
    ButtonOK: TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure ButtonOKClick(Sender: TObject);
  end;

implementation

{$R *.lfm}

procedure TAboutForm.FormCreate(Sender: TObject);
begin
  Caption := 'เกี่ยวกับ Text Editor';
  BorderStyle := bsDialog;
  Position := poOwnerFormCenter;
  Width := 400;
  Height := 300;
  
  PanelTop.Color := $00336699;
  PanelTop.Height := 80;
  PanelTop.Align := alTop;
  
  LabelAppName.Caption := 'Text Editor';
  LabelAppName.Font.Size := 20;
  LabelAppName.Font.Bold := True;
  LabelAppName.Font.Color := clWhite;
  LabelAppName.Parent := PanelTop;
  LabelAppName.Align := alLeft;
  LabelAppName.Layout := tlCenter;
  LabelAppName.Width := 200;
  LabelAppName.Left := 10;
  
  LabelVersion.Caption := 'Version 1.0.0';
  LabelVersion.Font.Color := clWhite;
  LabelVersion.Parent := PanelTop;
  LabelVersion.Left := 10;
  LabelVersion.Top := 55;
  
  LabelDescription.Caption := 
    'Text Editor สำหรับการแก้ไขไฟล์ข้อความ' + #13#10 +
    'รองรับ Unicode, UTF-8, Find & Replace';
  LabelDescription.Top := 100;
  LabelDescription.Left := 20;
  LabelDescription.AutoSize := True;
  
  LabelCopyright.Caption := '© 2024 - สร้างด้วย Lazarus/Free Pascal';
  LabelCopyright.Top := 160;
  LabelCopyright.Left := 20;
  LabelCopyright.Font.Color := $00666666;
  LabelCopyright.AutoSize := True;
  
  ButtonOK.Caption := 'ตกลง';
  ButtonOK.ModalResult := mrOk;
  ButtonOK.Default := True;
  ButtonOK.Left := Width - ButtonOK.Width - 20;
  ButtonOK.Top := Height - ButtonOK.Height - 50;
  ButtonOK.Anchors := [akRight, akBottom];
end;

procedure TAboutForm.ButtonOKClick(Sender: TObject);
begin
  ModalResult := mrOk;
end;

end.
```

---

## 20.5 EditorSettings Unit

```pascal
unit EditorSettings;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, IniFiles, Graphics;

type
  TEditorSettings = class
  private
    FSettingsFile: string;
    
    // Editor
    FFontName: string;
    FFontSize: Integer;
    FFontColor: TColor;
    FBackgroundColor: TColor;
    FWordWrap: Boolean;
    FTabWidth: Integer;
    FTabToSpaces: Boolean;
    FAutoIndent: Boolean;
    FShowLineNumbers: Boolean;
    
    // Window
    FWindowLeft: Integer;
    FWindowTop: Integer;
    FWindowWidth: Integer;
    FWindowHeight: Integer;
    FWindowMaximized: Boolean;
    
    // View
    FShowToolbar: Boolean;
    FShowStatusBar: Boolean;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Load;
    procedure Save;
    procedure SetDefaults;
    
    // Editor Properties
    property FontName: string read FFontName write FFontName;
    property FontSize: Integer read FFontSize write FFontSize;
    property FontColor: TColor read FFontColor write FFontColor;
    property BackgroundColor: TColor read FBackgroundColor write FBackgroundColor;
    property WordWrap: Boolean read FWordWrap write FWordWrap;
    property TabWidth: Integer read FTabWidth write FTabWidth;
    property TabToSpaces: Boolean read FTabToSpaces write FTabToSpaces;
    property AutoIndent: Boolean read FAutoIndent write FAutoIndent;
    property ShowLineNumbers: Boolean read FShowLineNumbers write FShowLineNumbers;
    
    // Window Properties
    property WindowLeft: Integer read FWindowLeft write FWindowLeft;
    property WindowTop: Integer read FWindowTop write FWindowTop;
    property WindowWidth: Integer read FWindowWidth write FWindowWidth;
    property WindowHeight: Integer read FWindowHeight write FWindowHeight;
    property WindowMaximized: Boolean read FWindowMaximized write FWindowMaximized;
    
    // View Properties
    property ShowToolbar: Boolean read FShowToolbar write FShowToolbar;
    property ShowStatusBar: Boolean read FShowStatusBar write FShowStatusBar;
  end;

implementation

constructor TEditorSettings.Create;
begin
  inherited Create;
  FSettingsFile := GetAppConfigDir(False) + 'texteditor.ini';
  SetDefaults;
end;

destructor TEditorSettings.Destroy;
begin
  inherited Destroy;
end;

procedure TEditorSettings.SetDefaults;
begin
  FFontName := 'Courier New';
  FFontSize := 11;
  FFontColor := clBlack;
  FBackgroundColor := clWhite;
  FWordWrap := False;
  FTabWidth := 4;
  FTabToSpaces := True;
  FAutoIndent := True;
  FShowLineNumbers := False;
  
  FWindowLeft := 100;
  FWindowTop := 100;
  FWindowWidth := 900;
  FWindowHeight := 650;
  FWindowMaximized := False;
  
  FShowToolbar := True;
  FShowStatusBar := True;
end;

procedure TEditorSettings.Load;
var
  ini: TIniFile;
begin
  if not FileExists(FSettingsFile) then Exit;
  
  ini := TIniFile.Create(FSettingsFile);
  try
    // Editor
    FFontName := ini.ReadString('Editor', 'FontName', FFontName);
    FFontSize := ini.ReadInteger('Editor', 'FontSize', FFontSize);
    FWordWrap := ini.ReadBool('Editor', 'WordWrap', FWordWrap);
    FTabWidth := ini.ReadInteger('Editor', 'TabWidth', FTabWidth);
    FTabToSpaces := ini.ReadBool('Editor', 'TabToSpaces', FTabToSpaces);
    FAutoIndent := ini.ReadBool('Editor', 'AutoIndent', FAutoIndent);
    FShowLineNumbers := ini.ReadBool('Editor', 'ShowLineNumbers', FShowLineNumbers);
    
    // Window
    FWindowLeft := ini.ReadInteger('Window', 'Left', FWindowLeft);
    FWindowTop := ini.ReadInteger('Window', 'Top', FWindowTop);
    FWindowWidth := ini.ReadInteger('Window', 'Width', FWindowWidth);
    FWindowHeight := ini.ReadInteger('Window', 'Height', FWindowHeight);
    FWindowMaximized := ini.ReadBool('Window', 'Maximized', FWindowMaximized);
    
    // View
    FShowToolbar := ini.ReadBool('View', 'Toolbar', FShowToolbar);
    FShowStatusBar := ini.ReadBool('View', 'StatusBar', FShowStatusBar);
  finally
    ini.Free;
  end;
end;

procedure TEditorSettings.Save;
var
  ini: TIniFile;
  dir: string;
begin
  dir := ExtractFilePath(FSettingsFile);
  if not DirectoryExists(dir) then
    CreateDir(dir);
  
  ini := TIniFile.Create(FSettingsFile);
  try
    ini.WriteString('Editor', 'FontName', FFontName);
    ini.WriteInteger('Editor', 'FontSize', FFontSize);
    ini.WriteBool('Editor', 'WordWrap', FWordWrap);
    ini.WriteInteger('Editor', 'TabWidth', FTabWidth);
    ini.WriteBool('Editor', 'TabToSpaces', FTabToSpaces);
    ini.WriteBool('Editor', 'AutoIndent', FAutoIndent);
    ini.WriteBool('Editor', 'ShowLineNumbers', FShowLineNumbers);
    
    ini.WriteInteger('Window', 'Left', FWindowLeft);
    ini.WriteInteger('Window', 'Top', FWindowTop);
    ini.WriteInteger('Window', 'Width', FWindowWidth);
    ini.WriteInteger('Window', 'Height', FWindowHeight);
    ini.WriteBool('Window', 'Maximized', FWindowMaximized);
    
    ini.WriteBool('View', 'Toolbar', FShowToolbar);
    ini.WriteBool('View', 'StatusBar', FShowStatusBar);
  finally
    ini.Free;
  end;
end;

end.
```

---

## 20.6 Project File

```pascal
// TextEditor.lpr - Project File
program TextEditor;

{$mode objfpc}{$H+}

uses
  {$IFDEF UNIX}
  cthreads,
  {$ENDIF}
  {$IFDEF HASAMIGA}
  athreads,
  {$ENDIF}
  Interfaces, // นำเข้า LCL Widget Set
  Forms,
  MainForm,
  FindReplaceForm,
  GoToLineForm,
  AboutForm,
  EditorSettings;

{$R *.res}

begin
  RequireDerivedFormResource := True;
  Application.Scaled := True; // รองรับ High DPI
  Application.Initialize;
  Application.CreateForm(TMainForm, MainForm);
  Application.Run;
end.
```

---

## 20.7 การทดสอบโปรแกรม

### 20.7.1 Test Cases

```pascal
// Test 1: New Document
procedure TestNewDocument;
begin
  // กด Ctrl+N
  // คาดหวัง: Memo ถูก Clear, Title = "ไม่มีชื่อ - Text Editor"
end;

// Test 2: Open File
procedure TestOpenFile;
begin
  // กด Ctrl+O, เลือกไฟล์
  // คาดหวัง: ไฟล์ถูกโหลดใน Memo
  // คาดหวัง: Title แสดงชื่อไฟล์
  // คาดหวัง: Recent Files มีไฟล์นี้
end;

// Test 3: Save File
procedure TestSaveFile;
begin
  // แก้ไข Memo, กด Ctrl+S
  // ถ้าใหม่: แสดง Save Dialog
  // คาดหวัง: ไฟล์ถูกบันทึก, Title ไม่มี *
end;

// Test 4: Find Text
procedure TestFindText;
begin
  // กด Ctrl+F, พิมพ์คำค้นหา
  // คาดหวัง: คำที่พบถูก Highlight
  // กด F3 = ค้นหาถัดไป
end;

// Test 5: Replace Text
procedure TestReplaceText;
begin
  // กด Ctrl+H, พิมพ์คำค้นหาและแทนที่
  // คาดหวัง: คำถูกแทนที่
end;

// Test 6: Word Wrap
procedure TestWordWrap;
begin
  // กด Format > Word Wrap
  // คาดหวัง: Memo Toggle Word Wrap
  // คาดหวัง: Check Mark ใน Menu ถูก Toggle
end;

// Test 7: Font Change
procedure TestFontChange;
begin
  // กด Format > Font, เลือก Font
  // คาดหวัง: Font ใน Memo เปลี่ยน
end;

// Test 8: Zoom
procedure TestZoom;
begin
  // กด Ctrl+= = Zoom In
  // กด Ctrl+- = Zoom Out
  // กด Ctrl+0 = Reset Zoom
end;
```

---

## 20.8 การ Package และ Distribution

### 20.8.1 การ Compile

```bash
# Compile ด้วย lazbuild
lazbuild --build-mode=Release TextEditor.lpi

# Cross-compile สำหรับ Windows บน Linux
lazbuild --os=win64 --cpu=x86_64 TextEditor.lpi
```

### 20.8.2 การสร้าง Installer (Windows)

```
[Script สำหรับ Inno Setup]
[Setup]
AppName=Text Editor
AppVersion=1.0.0
DefaultDirName={pf}\TextEditor
DefaultGroupName=Text Editor
OutputBaseFilename=TextEditorSetup

[Files]
Source: "TextEditor.exe"; DestDir: "{app}"
Source: "README.txt"; DestDir: "{app}"

[Icons]
Name: "{group}\Text Editor"; Filename: "{app}\TextEditor.exe"
Name: "{commondesktop}\Text Editor"; Filename: "{app}\TextEditor.exe"

[Registry]
; Register .txt file association
Root: HKCR; Subkey: ".txt\OpenWithProgids"; ValueType: string; 
  ValueName: "TextEditor.txt"; ValueData: ""
Root: HKCR; Subkey: "TextEditor.txt"; ValueType: string; ValueData: "Text File"
Root: HKCR; Subkey: "TextEditor.txt\shell\open\command"; ValueType: string; 
  ValueData: """{app}\TextEditor.exe"" ""%1"""
```

---

## 20.9 บทส่งท้าย - สรุปบทที่ 20 และ Course ทั้งหมด

### สิ่งที่ได้เรียนรู้ใน Part 16-20:

1. **Part 16 - Forms และ Controls**: TForm, TButton, TEdit, TMemo, TListBox ฯลฯ
2. **Part 17 - Events**: Mouse, Keyboard, Focus, Custom Events
3. **Part 18 - Menus และ Dialogs**: TMainMenu, TPopupMenu, Standard Dialogs
4. **Part 19 - Layout Management**: Anchors, Align, Splitters, Responsive Design
5. **Part 20 - Text Editor Project**: นำทุกอย่างมารวมกัน

### ขั้นตอนต่อไป:

```
1. เพิ่มฟีเจอร์ Syntax Highlighting ด้วย SynEdit
2. เพิ่มการรองรับ Plugins
3. เพิ่ม File Encoding Detection ที่ดีกว่า
4. สร้าง Settings Dialog ที่สมบูรณ์
5. เพิ่ม Print Preview
6. รองรับ Multiple Documents (MDI)
```

### แหล่งเรียนรู้เพิ่มเติม:

- [Lazarus Wiki](https://wiki.lazarus.freepascal.org/)
- [Free Pascal Documentation](https://www.freepascal.org/docs.html)
- [Lazarus Forum](https://forum.lazarus.freepascal.org/)

---

## แบบฝึกหัดโปรเจค

### โปรเจคที่ 1: Text Editor Plus
ขยายจาก Text Editor ที่สร้างมาพร้อมฟีเจอร์เพิ่ม:
- Line Numbers ด้านซ้าย
- Syntax Highlighting ง่ายๆ (Keywords สี่แดง)
- Auto-complete ง่ายๆ
- Multiple Tabs

### โปรเจคที่ 2: File Manager
สร้าง File Manager อย่างง่ายที่มี:
- TreeView แสดง Directory
- ListView แสดงไฟล์
- ตัวอย่าง Copy, Move, Delete
- Preview ไฟล์ข้อความ

```pascal
unit SimpleFileManager;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, ComCtrls, ExtCtrls,
  StdCtrls, Menus, FileUtil;

type
  TFileManagerForm = class(TForm)
    TreeViewDirs: TTreeView;
    ListViewFiles: TListView;
    SplitterMain: TSplitter;
    StatusBar: TStatusBar;
    
    procedure FormCreate(Sender: TObject);
    procedure TreeViewDirsChange(Sender: TObject; Node: TTreeNode);
    procedure ListViewFilesDblClick(Sender: TObject);
    
  private
    procedure LoadDrives;
    procedure LoadDirectory(const Path: string; ParentNode: TTreeNode);
    procedure DisplayFiles(const Path: string);
    procedure OpenFile(const FileName: string);
  end;

implementation

procedure TFileManagerForm.FormCreate(Sender: TObject);
begin
  Caption := 'File Manager';
  Width := 1000;
  Height := 700;
  
  // Setup TreeView
  TreeViewDirs.Align := alLeft;
  TreeViewDirs.Width := 300;
  TreeViewDirs.OnChange := TreeViewDirsChange;
  
  // Setup Splitter
  SplitterMain.Align := alLeft;
  SplitterMain.Width := 4;
  
  // Setup ListView
  ListViewFiles.Align := alClient;
  ListViewFiles.ViewStyle := vsReport;
  ListViewFiles.OnDblClick := ListViewFilesDblClick;
  
  with ListViewFiles.Columns.Add do
  begin
    Caption := 'ชื่อ';
    Width := 300;
  end;
  with ListViewFiles.Columns.Add do
  begin
    Caption := 'ขนาด';
    Width := 100;
    Alignment := taRightJustify;
  end;
  with ListViewFiles.Columns.Add do
  begin
    Caption := 'วันที่แก้ไข';
    Width := 150;
  end;
  with ListViewFiles.Columns.Add do
  begin
    Caption := 'ประเภท';
    Width := 100;
  end;
  
  LoadDrives;
end;

procedure TFileManagerForm.LoadDrives;
var
  drives: TStringList;
  node: TTreeNode;
  i: Integer;
begin
  drives := TStringList.Create;
  try
    // ใน Linux: เพิ่ม root /
    // ใน Windows: หา Drives
    {$IFDEF WINDOWS}
    var driveStr: string;
    for var c := 'A' to 'Z' do
    begin
      driveStr := c + ':\';
      if DirectoryExists(driveStr) then
        drives.Add(driveStr);
    end;
    {$ELSE}
    drives.Add('/');
    drives.Add('/home');
    drives.Add('/tmp');
    {$ENDIF}
    
    for i := 0 to drives.Count - 1 do
    begin
      node := TreeViewDirs.Items.Add(nil, drives[i]);
      node.Data := Pointer(PtrInt(0)); // Marker
      // เพิ่ม Dummy Child เพื่อให้มีปุ่ม Expand
      TreeViewDirs.Items.AddChild(node, '...');
    end;
  finally
    drives.Free;
  end;
end;

procedure TFileManagerForm.TreeViewDirsChange(Sender: TObject; Node: TTreeNode);
begin
  if Node = nil then Exit;
  DisplayFiles(Node.Text);
end;

procedure TFileManagerForm.DisplayFiles(const Path: string);
var
  searchRec: TSearchRec;
  item: TListItem;
  fileSize: Int64;
begin
  ListViewFiles.Items.Clear;
  
  if not DirectoryExists(Path) then Exit;
  
  try
    // โฟลเดอร์ก่อน
    if FindFirst(Path + DirectorySeparator + '*', faDirectory, searchRec) = 0 then
    begin
      try
        repeat
          if (searchRec.Name = '.') or (searchRec.Name = '..') then Continue;
          if (searchRec.Attr and faDirectory) = faDirectory then
          begin
            item := ListViewFiles.Items.Add;
            item.Caption := '📁 ' + searchRec.Name;
            item.SubItems.Add('-');
            item.SubItems.Add(FormatDateTime('dd/mm/yyyy hh:nn', 
              FileDateToDateTime(searchRec.Time)));
            item.SubItems.Add('โฟลเดอร์');
          end;
        until FindNext(searchRec) <> 0;
      finally
        FindClose(searchRec);
      end;
    end;
    
    // ไฟล์
    if FindFirst(Path + DirectorySeparator + '*', faAnyFile - faDirectory, searchRec) = 0 then
    begin
      try
        repeat
          if (searchRec.Attr and faDirectory) = 0 then
          begin
            item := ListViewFiles.Items.Add;
            item.Caption := '📄 ' + searchRec.Name;
            
            fileSize := searchRec.Size;
            if fileSize < 1024 then
              item.SubItems.Add(IntToStr(fileSize) + ' B')
            else if fileSize < 1024 * 1024 then
              item.SubItems.Add(FormatFloat('0.0', fileSize / 1024) + ' KB')
            else
              item.SubItems.Add(FormatFloat('0.0', fileSize / 1024 / 1024) + ' MB');
            
            item.SubItems.Add(FormatDateTime('dd/mm/yyyy hh:nn',
              FileDateToDateTime(searchRec.Time)));
            item.SubItems.Add(UpperCase(ExtractFileExt(searchRec.Name)));
          end;
        until FindNext(searchRec) <> 0;
      finally
        FindClose(searchRec);
      end;
    end;
  except
    on E: Exception do
      StatusBar.Panels[0].Text := 'ข้อผิดพลาด: ' + E.Message;
  end;
  
  StatusBar.Panels[0].Text := Path;
  StatusBar.Panels[1].Text := IntToStr(ListViewFiles.Items.Count) + ' รายการ';
end;

procedure TFileManagerForm.ListViewFilesDblClick(Sender: TObject);
begin
  if ListViewFiles.Selected = nil then Exit;
  
  var name := ListViewFiles.Selected.Caption;
  // ลบ Emoji Prefix
  if name[1] = '📁' then
    name := Trim(Copy(name, 3, MaxInt));
  if name[1] = '📄' then
    name := Trim(Copy(name, 3, MaxInt));
  
  // TODO: Navigate to folder or open file
end;

end.
```

---

## สรุปหลักสูตร Part 16-20

การเรียนรู้ใน Part 16-20 นี้ครอบคลุม:

### พื้นฐาน GUI Programming:
1. ทำความเข้าใจ Form และ Control Architecture
2. Event-Driven Programming Model
3. Menu และ Dialog Systems
4. Layout Management และ Responsive Design

### ทักษะเชิงปฏิบัติ:
1. สร้าง Form ที่ใช้งานได้จริง
2. จัดการ Events ทุกประเภท
3. สร้าง Dialogs ทั้ง Standard และ Custom
4. ออกแบบ Layout ที่ Flexible และ Responsive
5. สร้างโปรแกรมจริงด้วย Text Editor

### แนวทางต่อไป:
- เรียน Database Programming (Lazarus with SQLite/MySQL)
- เรียน Graphics Programming (Canvas, OpenGL)
- เรียน Network Programming (TCP/IP, HTTP)
- สร้างโปรเจคที่ซับซ้อนขึ้น
