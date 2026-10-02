# Part 38 - Multimedia

## บทนำ

Multimedia ในการพัฒนาโปรแกรมด้วย Lazarus ครอบคลุมการเล่นเสียง วิดีโอ การบันทึกเสียง และ Text-to-Speech Lazarus มี component หลายตัวสำหรับงาน Multimedia ทั้งแบบ built-in และ third-party

---

## 38.1 การเล่นเสียง (Audio Playback)

### ใช้ MediaPlayer Component

```pascal
unit AudioPlayer;

interface

uses
  Classes, SysUtils, Forms, Controls, MPlayer, 
  Buttons, StdCtrls, ExtCtrls, ComCtrls, Dialogs;

type
  TForm1 = class(TForm)
    MediaPlayer1: TMediaPlayer;
    btnOpen: TButton;
    btnPlay: TButton;
    btnPause: TButton;
    btnStop: TButton;
    TrackPosition: TTrackBar;
    lblTime: TLabel;
    Timer1: TTimer;
    VolumeTrack: TTrackBar;
    lblVolume: TLabel;
    procedure FormCreate(Sender: TObject);
    procedure btnOpenClick(Sender: TObject);
    procedure btnPlayClick(Sender: TObject);
    procedure btnPauseClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure TrackPositionChange(Sender: TObject);
    procedure VolumeTrackChange(Sender: TObject);
    procedure MediaPlayer1Notify(Sender: TObject);
  private
    FIsPlaying: Boolean;
    FDuration: Integer;
    procedure UpdateTimeDisplay;
    function FormatTime(Ms: Integer): string;
    procedure UpdateButtons;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  FIsPlaying := False;
  FDuration := 0;
  
  Timer1.Interval := 500;
  Timer1.Enabled := False;
  
  TrackPosition.Min := 0;
  TrackPosition.Max := 1000;
  TrackPosition.Position := 0;
  
  VolumeTrack.Min := 0;
  VolumeTrack.Max := 100;
  VolumeTrack.Position := 100;
  
  btnPlay.Enabled := False;
  btnPause.Enabled := False;
  btnStop.Enabled := False;
  
  MediaPlayer1.Notify := True;
  MediaPlayer1.Wait := False;
end;

procedure TForm1.btnOpenClick(Sender: TObject);
var
  OD: TOpenDialog;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.Filter := 'Audio Files|*.mp3;*.wav;*.ogg;*.flac;*.aac;*.wma|' +
                  'MP3|*.mp3|WAV|*.wav|OGG|*.ogg|All|*.*';
    if OD.Execute then
    begin
      MediaPlayer1.Stop;
      FIsPlaying := False;
      Timer1.Enabled := False;
      
      MediaPlayer1.FileName := OD.FileName;
      MediaPlayer1.Open;
      
      // หา Duration
      FDuration := MediaPlayer1.Length;
      TrackPosition.Position := 0;
      
      Caption := 'Audio Player - ' + ExtractFileName(OD.FileName);
      UpdateButtons;
      UpdateTimeDisplay;
    end;
  finally
    OD.Free;
  end;
end;

procedure TForm1.btnPlayClick(Sender: TObject);
begin
  if MediaPlayer1.FileName = '' then Exit;
  
  if FIsPlaying then
  begin
    // Resume from pause
    MediaPlayer1.Resume;
  end else
  begin
    MediaPlayer1.Play;
  end;
  
  FIsPlaying := True;
  Timer1.Enabled := True;
  UpdateButtons;
end;

procedure TForm1.btnPauseClick(Sender: TObject);
begin
  if FIsPlaying then
  begin
    MediaPlayer1.Pause;
    FIsPlaying := False;
    Timer1.Enabled := False;
    UpdateButtons;
  end;
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  MediaPlayer1.Stop;
  FIsPlaying := False;
  Timer1.Enabled := False;
  TrackPosition.Position := 0;
  UpdateTimeDisplay;
  UpdateButtons;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
var
  Pos: Integer;
begin
  if not FIsPlaying then Exit;
  
  Pos := MediaPlayer1.Position;
  
  if FDuration > 0 then
    TrackPosition.Position := Round(Pos / FDuration * 1000);
  
  UpdateTimeDisplay;
end;

procedure TForm1.TrackPositionChange(Sender: TObject);
begin
  if FDuration > 0 then
  begin
    var NewPos := Round(TrackPosition.Position / 1000 * FDuration);
    MediaPlayer1.Position := NewPos;
    UpdateTimeDisplay;
  end;
end;

procedure TForm1.VolumeTrackChange(Sender: TObject);
begin
  MediaPlayer1.Channels := VolumeTrack.Position * 65535 div 100;
  lblVolume.Caption := Format('Volume: %d%%', [VolumeTrack.Position]);
end;

procedure TForm1.MediaPlayer1Notify(Sender: TObject);
begin
  // เมื่อเล่นเสร็จ
  if MediaPlayer1.Mode = mpStopped then
  begin
    FIsPlaying := False;
    Timer1.Enabled := False;
    TrackPosition.Position := 0;
    UpdateButtons;
  end;
end;

function TForm1.FormatTime(Ms: Integer): string;
var
  Sec, Min, Hr: Integer;
begin
  Sec := Ms div 1000;
  Min := Sec div 60;
  Hr := Min div 60;
  Sec := Sec mod 60;
  Min := Min mod 60;
  
  if Hr > 0 then
    Result := Format('%d:%02d:%02d', [Hr, Min, Sec])
  else
    Result := Format('%d:%02d', [Min, Sec]);
end;

procedure TForm1.UpdateTimeDisplay;
var
  Pos: Integer;
begin
  Pos := MediaPlayer1.Position;
  lblTime.Caption := FormatTime(Pos) + ' / ' + FormatTime(FDuration);
end;

procedure TForm1.UpdateButtons;
begin
  if MediaPlayer1.FileName <> '' then
  begin
    btnPlay.Enabled := not FIsPlaying;
    btnPause.Enabled := FIsPlaying;
    btnStop.Enabled := FIsPlaying or (MediaPlayer1.Position > 0);
  end else
  begin
    btnPlay.Enabled := False;
    btnPause.Enabled := False;
    btnStop.Enabled := False;
  end;
end;

end.
```

---

## 38.2 การเล่นเสียงด้วย Bass Library

Bass เป็น Library ที่ทรงพลังสำหรับการเล่นเสียง รองรับ MP3, WAV, OGG และอีกมาก

```pascal
unit BassAudio;

interface

uses
  Classes, SysUtils, Windows, DynLibs;

// Bass API declarations
type
  HSTREAM = Cardinal;
  HSAMPLE = Cardinal;
  HCHANNEL = Cardinal;

const
  BASS_OK = 0;
  BASS_SAMPLE_FLOAT = $100;
  BASS_STREAM_PRESCAN = $20000;
  BASS_ACTIVE_STOPPED = 0;
  BASS_ACTIVE_PLAYING = 1;
  BASS_ACTIVE_STALLED = 2;
  BASS_ACTIVE_PAUSED = 3;
  
var
  // Function pointers
  BASS_Init: function(device, freq, flags: Integer; win: HWND; clsid: Pointer): BOOL; stdcall;
  BASS_Free: function: BOOL; stdcall;
  BASS_StreamCreateFile: function(mem: BOOL; f: Pointer; offset, length: QWORD; flags: Cardinal): HSTREAM; stdcall;
  BASS_ChannelPlay: function(handle: HCHANNEL; restart: BOOL): BOOL; stdcall;
  BASS_ChannelPause: function(handle: HCHANNEL): BOOL; stdcall;
  BASS_ChannelStop: function(handle: HCHANNEL): BOOL; stdcall;
  BASS_ChannelGetPosition: function(handle: HCHANNEL; mode: Cardinal): QWORD; stdcall;
  BASS_ChannelSetPosition: function(handle: HCHANNEL; pos: QWORD; mode: Cardinal): BOOL; stdcall;
  BASS_ChannelGetLength: function(handle: HCHANNEL; mode: Cardinal): QWORD; stdcall;
  BASS_ChannelBytes2Seconds: function(handle: HCHANNEL; pos: QWORD): Double; stdcall;
  BASS_ChannelSeconds2Bytes: function(handle: HCHANNEL; pos: Double): QWORD; stdcall;
  BASS_ChannelSetAttribute: function(handle: HCHANNEL; attrib: Cardinal; value: Single): BOOL; stdcall;
  BASS_ChannelIsActive: function(handle: HCHANNEL): Cardinal; stdcall;
  BASS_StreamFree: function(handle: HSTREAM): BOOL; stdcall;
  
var
  BassLib: TLibHandle;
  BassLoaded: Boolean;

procedure LoadBass(const DllPath: string);
procedure FreeBass;

type
  TBassPlayer = class
  private
    FStream: HSTREAM;
    FFileName: string;
    FVolume: Single;
  public
    constructor Create;
    destructor Destroy; override;
    function Load(const FileName: string): Boolean;
    procedure Play;
    procedure Pause;
    procedure Stop;
    function GetPosition: Double;
    procedure SetPosition(Pos: Double);
    function GetDuration: Double;
    procedure SetVolume(V: Single);
    function IsPlaying: Boolean;
    property FileName: string read FFileName;
    property Volume: Single read FVolume write SetVolume;
  end;

implementation

procedure LoadBass(const DllPath: string);
begin
  BassLib := LoadLibrary(PChar(DllPath + 'bass.dll'));
  if BassLib = 0 then
  begin
    BassLoaded := False;
    Exit;
  end;
  
  @BASS_Init := GetProcAddress(BassLib, 'BASS_Init');
  @BASS_Free := GetProcAddress(BassLib, 'BASS_Free');
  @BASS_StreamCreateFile := GetProcAddress(BassLib, 'BASS_StreamCreateFile');
  @BASS_ChannelPlay := GetProcAddress(BassLib, 'BASS_ChannelPlay');
  @BASS_ChannelPause := GetProcAddress(BassLib, 'BASS_ChannelPause');
  @BASS_ChannelStop := GetProcAddress(BassLib, 'BASS_ChannelStop');
  @BASS_ChannelGetPosition := GetProcAddress(BassLib, 'BASS_ChannelGetPosition');
  @BASS_ChannelSetPosition := GetProcAddress(BassLib, 'BASS_ChannelSetPosition');
  @BASS_ChannelGetLength := GetProcAddress(BassLib, 'BASS_ChannelGetLength');
  @BASS_ChannelBytes2Seconds := GetProcAddress(BassLib, 'BASS_ChannelBytes2Seconds');
  @BASS_ChannelSeconds2Bytes := GetProcAddress(BassLib, 'BASS_ChannelSeconds2Bytes');
  @BASS_ChannelSetAttribute := GetProcAddress(BassLib, 'BASS_ChannelSetAttribute');
  @BASS_ChannelIsActive := GetProcAddress(BassLib, 'BASS_ChannelIsActive');
  @BASS_StreamFree := GetProcAddress(BassLib, 'BASS_StreamFree');
  
  if Assigned(BASS_Init) then
    BassLoaded := BASS_Init(-1, 44100, 0, 0, nil)
  else
    BassLoaded := False;
end;

procedure FreeBass;
begin
  if BassLoaded and Assigned(BASS_Free) then
    BASS_Free;
  if BassLib <> 0 then
    FreeLibrary(BassLib);
  BassLib := 0;
  BassLoaded := False;
end;

constructor TBassPlayer.Create;
begin
  inherited;
  FStream := 0;
  FVolume := 1.0;
end;

destructor TBassPlayer.Destroy;
begin
  if FStream <> 0 then
  begin
    BASS_StreamFree(FStream);
    FStream := 0;
  end;
  inherited;
end;

function TBassPlayer.Load(const FileName: string): Boolean;
begin
  if FStream <> 0 then
  begin
    BASS_StreamFree(FStream);
    FStream := 0;
  end;
  
  if not BassLoaded then begin Result := False; Exit; end;
  
  FStream := BASS_StreamCreateFile(False, PChar(FileName), 0, 0, BASS_STREAM_PRESCAN);
  FFileName := FileName;
  Result := FStream <> 0;
end;

procedure TBassPlayer.Play;
begin
  if FStream <> 0 then
    BASS_ChannelPlay(FStream, False);
end;

procedure TBassPlayer.Pause;
begin
  if FStream <> 0 then
    BASS_ChannelPause(FStream);
end;

procedure TBassPlayer.Stop;
begin
  if FStream <> 0 then
  begin
    BASS_ChannelStop(FStream);
    BASS_ChannelSetPosition(FStream, 0, 0);
  end;
end;

function TBassPlayer.GetPosition: Double;
var
  BytePos: QWORD;
begin
  if FStream = 0 then begin Result := 0; Exit; end;
  BytePos := BASS_ChannelGetPosition(FStream, 0);
  Result := BASS_ChannelBytes2Seconds(FStream, BytePos);
end;

procedure TBassPlayer.SetPosition(Pos: Double);
var
  BytePos: QWORD;
begin
  if FStream = 0 then Exit;
  BytePos := BASS_ChannelSeconds2Bytes(FStream, Pos);
  BASS_ChannelSetPosition(FStream, BytePos, 0);
end;

function TBassPlayer.GetDuration: Double;
var
  ByteLen: QWORD;
begin
  if FStream = 0 then begin Result := 0; Exit; end;
  ByteLen := BASS_ChannelGetLength(FStream, 0);
  Result := BASS_ChannelBytes2Seconds(FStream, ByteLen);
end;

procedure TBassPlayer.SetVolume(V: Single);
begin
  FVolume := V;
  if FStream <> 0 then
    BASS_ChannelSetAttribute(FStream, 2, V);  // BASS_ATTRIB_VOL = 2
end;

function TBassPlayer.IsPlaying: Boolean;
begin
  if FStream = 0 then begin Result := False; Exit; end;
  Result := BASS_ChannelIsActive(FStream) = BASS_ACTIVE_PLAYING;
end;

end.
```

---

## 38.3 การใช้งาน TMediaPlayer

TMediaPlayer เป็น Component หลักของ Lazarus สำหรับ Multimedia

```pascal
unit MediaPlayerDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, MPlayer,
  StdCtrls, ExtCtrls, Dialogs, ComCtrls;

type
  TForm1 = class(TForm)
    MediaPlayer1: TMediaPlayer;
    PanelVideo: TPanel;
    PanelControls: TPanel;
    
    btnOpen: TButton;
    btnPlay: TButton;
    btnPause: TButton;
    btnStop: TButton;
    btnPrev: TButton;
    btnNext: TButton;
    
    SeekBar: TTrackBar;
    VolumeBar: TTrackBar;
    
    lblPosition: TLabel;
    lblVolume: TLabel;
    lblStatus: TLabel;
    
    Timer1: TTimer;
    
    procedure FormCreate(Sender: TObject);
    procedure btnOpenClick(Sender: TObject);
    procedure btnPlayClick(Sender: TObject);
    procedure btnPauseClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
    procedure btnPrevClick(Sender: TObject);
    procedure btnNextClick(Sender: TObject);
    procedure SeekBarChange(Sender: TObject);
    procedure VolumeBarChange(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure MediaPlayer1Notify(Sender: TObject);
  private
    FSeekbarDragging: Boolean;
    
    procedure UpdateUI;
    procedure SetupMediaPlayer;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  FSeekbarDragging := False;
  SetupMediaPlayer;
  
  SeekBar.Min := 0;
  SeekBar.Max := 1000;
  SeekBar.Position := 0;
  
  VolumeBar.Min := 0;
  VolumeBar.Max := 100;
  VolumeBar.Position := 100;
  
  Timer1.Interval := 250;
  Timer1.Enabled := False;
end;

procedure TForm1.SetupMediaPlayer;
begin
  // ตั้งค่า MediaPlayer ให้แสดงวิดีโอใน Panel
  MediaPlayer1.Display := PanelVideo;
  MediaPlayer1.DisplayRect := PanelVideo.ClientRect;
  MediaPlayer1.Notify := True;
  MediaPlayer1.Wait := False;
  MediaPlayer1.AutoOpen := False;
end;

procedure TForm1.btnOpenClick(Sender: TObject);
var
  OD: TOpenDialog;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.Filter := 'Video/Audio|*.mp4;*.avi;*.mkv;*.mov;*.mp3;*.wav;*.ogg|All|*.*';
    if OD.Execute then
    begin
      MediaPlayer1.Stop;
      Timer1.Enabled := False;
      
      MediaPlayer1.FileName := OD.FileName;
      MediaPlayer1.Open;
      
      Caption := 'Media Player - ' + ExtractFileName(OD.FileName);
      SeekBar.Position := 0;
      UpdateUI;
    end;
  finally
    OD.Free;
  end;
end;

procedure TForm1.btnPlayClick(Sender: TObject);
begin
  if MediaPlayer1.FileName = '' then Exit;
  MediaPlayer1.Play;
  Timer1.Enabled := True;
  UpdateUI;
end;

procedure TForm1.btnPauseClick(Sender: TObject);
begin
  case MediaPlayer1.Mode of
    mpPlaying: begin
      MediaPlayer1.Pause;
      Timer1.Enabled := False;
      lblStatus.Caption := 'หยุดชั่วคราว';
    end;
    mpPaused: begin
      MediaPlayer1.Resume;
      Timer1.Enabled := True;
      lblStatus.Caption := 'กำลังเล่น';
    end;
  end;
  UpdateUI;
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  MediaPlayer1.Stop;
  Timer1.Enabled := False;
  SeekBar.Position := 0;
  lblPosition.Caption := '0:00 / 0:00';
  lblStatus.Caption := 'หยุด';
  UpdateUI;
end;

procedure TForm1.btnPrevClick(Sender: TObject);
begin
  // ย้อนกลับ 10 วินาที
  if MediaPlayer1.Mode in [mpPlaying, mpPaused] then
  begin
    var NewPos := Max(0, MediaPlayer1.Position - 10000);
    MediaPlayer1.Position := NewPos;
  end;
end;

procedure TForm1.btnNextClick(Sender: TObject);
begin
  // ข้ามไป 10 วินาที
  if MediaPlayer1.Mode in [mpPlaying, mpPaused] then
  begin
    var NewPos := Min(MediaPlayer1.Length, MediaPlayer1.Position + 10000);
    MediaPlayer1.Position := NewPos;
  end;
end;

procedure TForm1.SeekBarChange(Sender: TObject);
begin
  if FSeekbarDragging and (MediaPlayer1.Length > 0) then
  begin
    var NewPos := Round(SeekBar.Position / 1000 * MediaPlayer1.Length);
    MediaPlayer1.Position := NewPos;
  end;
end;

procedure TForm1.VolumeBarChange(Sender: TObject);
begin
  MediaPlayer1.Channels := VolumeBar.Position * 65535 div 100;
  lblVolume.Caption := Format('เสียง: %d%%', [VolumeBar.Position]);
end;

procedure TForm1.Timer1Timer(Sender: TObject);
var
  Pos, Len: Integer;
begin
  if not FSeekbarDragging and (MediaPlayer1.Length > 0) then
  begin
    Pos := MediaPlayer1.Position;
    Len := MediaPlayer1.Length;
    
    SeekBar.Position := Round(Pos / Len * 1000);
    
    // แสดงเวลา
    var PosS := Pos div 1000;
    var LenS := Len div 1000;
    lblPosition.Caption := Format('%d:%02d / %d:%02d',
      [PosS div 60, PosS mod 60, LenS div 60, LenS mod 60]);
  end;
end;

procedure TForm1.MediaPlayer1Notify(Sender: TObject);
begin
  // เมื่อเล่นเสร็จ
  if MediaPlayer1.Mode = mpStopped then
  begin
    Timer1.Enabled := False;
    SeekBar.Position := 0;
    lblStatus.Caption := 'เล่นเสร็จแล้ว';
    UpdateUI;
  end;
end;

procedure TForm1.UpdateUI;
begin
  btnPlay.Enabled := (MediaPlayer1.FileName <> '') and 
                      (MediaPlayer1.Mode <> mpPlaying);
  btnPause.Enabled := MediaPlayer1.Mode in [mpPlaying, mpPaused];
  btnStop.Enabled := MediaPlayer1.Mode in [mpPlaying, mpPaused];
  
  case MediaPlayer1.Mode of
    mpPlaying: 
    begin
      lblStatus.Caption := 'กำลังเล่น';
      btnPause.Caption := 'หยุดชั่วคราว';
    end;
    mpPaused: 
    begin
      lblStatus.Caption := 'หยุดชั่วคราว';
      btnPause.Caption := 'เล่นต่อ';
    end;
    mpStopped: 
    begin
      lblStatus.Caption := 'หยุด';
      btnPause.Caption := 'หยุด/เล่นต่อ';
    end;
    mpOpen: 
    begin
      lblStatus.Caption := 'พร้อม';
    end;
  end;
end;

end.
```

---

## 38.4 Playlist Management

```pascal
unit PlaylistManager;

interface

uses
  Classes, SysUtils, Forms, Controls, MPlayer,
  StdCtrls, ExtCtrls, Dialogs, ComCtrls;

type
  TPlayMode = (pmNormal, pmShuffle, pmRepeatOne, pmRepeatAll);
  
  TPlaylistItem = record
    FileName: string;
    Title: string;
    Duration: Integer;  // milliseconds
  end;
  
  TPlaylist = class
  private
    FItems: array of TPlaylistItem;
    FCurrentIndex: Integer;
    FPlayMode: TPlayMode;
    FShuffleOrder: array of Integer;
    
    procedure GenerateShuffleOrder;
  public
    constructor Create;
    procedure Add(const FileName: string);
    procedure Remove(Index: Integer);
    procedure Clear;
    function GetNext: Integer;
    function GetPrevious: Integer;
    function GetCurrent: TPlaylistItem;
    procedure SetCurrent(Index: Integer);
    procedure Shuffle;
    
    property Count: Integer read (Length(FItems));
    property CurrentIndex: Integer read FCurrentIndex;
    property PlayMode: TPlayMode read FPlayMode write FPlayMode;
    property Items[Index: Integer]: TPlaylistItem read GetItem;
  end;
  
  TForm1 = class(TForm)
    MediaPlayer1: TMediaPlayer;
    ListBoxPlaylist: TListBox;
    
    btnAddFiles: TButton;
    btnAddDir: TButton;
    btnRemove: TButton;
    btnClear: TButton;
    
    btnPlay: TButton;
    btnPause: TButton;
    btnStop: TButton;
    btnPrev: TButton;
    btnNext: TButton;
    
    btnShuffle: TSpeedButton;
    btnRepeat: TSpeedButton;
    
    SeekBar: TTrackBar;
    VolumeBar: TTrackBar;
    
    lblNowPlaying: TLabel;
    lblTime: TLabel;
    
    Timer1: TTimer;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnAddFilesClick(Sender: TObject);
    procedure btnAddDirClick(Sender: TObject);
    procedure btnRemoveClick(Sender: TObject);
    procedure btnClearClick(Sender: TObject);
    procedure btnPlayClick(Sender: TObject);
    procedure btnPauseClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
    procedure btnPrevClick(Sender: TObject);
    procedure btnNextClick(Sender: TObject);
    procedure ListBoxPlaylistDblClick(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure MediaPlayer1Notify(Sender: TObject);
  private
    FPlaylist: TPlaylist;
    procedure LoadAndPlay(Index: Integer);
    procedure UpdatePlaylistDisplay;
    procedure UpdateNowPlaying;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

uses FileUtil, LazFileUtils;

constructor TPlaylist.Create;
begin
  inherited;
  FCurrentIndex := -1;
  FPlayMode := pmNormal;
  SetLength(FItems, 0);
end;

procedure TPlaylist.Add(const FileName: string);
var
  Item: TPlaylistItem;
begin
  Item.FileName := FileName;
  Item.Title := ExtractFileName(FileName);
  // ลบ extension
  Item.Title := ChangeFileExt(Item.Title, '');
  Item.Duration := 0;
  
  SetLength(FItems, Length(FItems) + 1);
  FItems[High(FItems)] := Item;
end;

procedure TPlaylist.Remove(Index: Integer);
var
  i: Integer;
begin
  if (Index < 0) or (Index >= Length(FItems)) then Exit;
  for i := Index to High(FItems) - 1 do
    FItems[i] := FItems[i + 1];
  SetLength(FItems, Length(FItems) - 1);
  if FCurrentIndex >= Length(FItems) then
    FCurrentIndex := Length(FItems) - 1;
end;

procedure TPlaylist.Clear;
begin
  SetLength(FItems, 0);
  FCurrentIndex := -1;
end;

procedure TPlaylist.GenerateShuffleOrder;
var
  i, j, tmp: Integer;
begin
  SetLength(FShuffleOrder, Length(FItems));
  for i := 0 to High(FShuffleOrder) do
    FShuffleOrder[i] := i;
  // Fisher-Yates shuffle
  for i := High(FShuffleOrder) downto 1 do
  begin
    j := Random(i + 1);
    tmp := FShuffleOrder[i];
    FShuffleOrder[i] := FShuffleOrder[j];
    FShuffleOrder[j] := tmp;
  end;
end;

function TPlaylist.GetNext: Integer;
begin
  if Length(FItems) = 0 then begin Result := -1; Exit; end;
  
  case FPlayMode of
    pmNormal:
    begin
      if FCurrentIndex >= High(FItems) then Result := -1
      else Result := FCurrentIndex + 1;
    end;
    pmRepeatAll:
    begin
      Result := (FCurrentIndex + 1) mod Length(FItems);
    end;
    pmRepeatOne:
    begin
      Result := FCurrentIndex;
    end;
    pmShuffle:
    begin
      GenerateShuffleOrder;
      Result := FShuffleOrder[0];
    end;
  end;
end;

function TPlaylist.GetPrevious: Integer;
begin
  if Length(FItems) = 0 then begin Result := -1; Exit; end;
  
  if FCurrentIndex <= 0 then
    Result := 0
  else
    Result := FCurrentIndex - 1;
end;

function TPlaylist.GetCurrent: TPlaylistItem;
begin
  if (FCurrentIndex >= 0) and (FCurrentIndex < Length(FItems)) then
    Result := FItems[FCurrentIndex]
  else
    FillChar(Result, SizeOf(Result), 0);
end;

procedure TPlaylist.SetCurrent(Index: Integer);
begin
  if (Index >= 0) and (Index < Length(FItems)) then
    FCurrentIndex := Index;
end;

procedure TPlaylist.Shuffle;
begin
  FPlayMode := pmShuffle;
  GenerateShuffleOrder;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FPlaylist := TPlaylist.Create;
  Timer1.Interval := 250;
  Timer1.Enabled := False;
  MediaPlayer1.Notify := True;
  MediaPlayer1.Wait := False;
  SeekBar.Min := 0;
  SeekBar.Max := 1000;
  VolumeBar.Min := 0;
  VolumeBar.Max := 100;
  VolumeBar.Position := 100;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FPlaylist.Free;
end;

procedure TForm1.btnAddFilesClick(Sender: TObject);
var
  OD: TOpenDialog;
  i: Integer;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.MultiSelect := True;
    OD.Filter := 'Audio|*.mp3;*.wav;*.ogg;*.flac;*.aac|Video|*.mp4;*.avi;*.mkv|All|*.*';
    if OD.Execute then
    begin
      for i := 0 to OD.Files.Count - 1 do
        FPlaylist.Add(OD.Files[i]);
      UpdatePlaylistDisplay;
    end;
  finally
    OD.Free;
  end;
end;

procedure TForm1.btnAddDirClick(Sender: TObject);
var
  DD: TSelectDirectoryDialog;
  Files: TStringList;
  i: Integer;
begin
  DD := TSelectDirectoryDialog.Create(nil);
  try
    if DD.Execute then
    begin
      Files := FindAllFiles(DD.FileName, '*.mp3;*.wav;*.ogg;*.flac;*.mp4;*.avi', True);
      try
        for i := 0 to Files.Count - 1 do
          FPlaylist.Add(Files[i]);
        UpdatePlaylistDisplay;
      finally
        Files.Free;
      end;
    end;
  finally
    DD.Free;
  end;
end;

procedure TForm1.btnRemoveClick(Sender: TObject);
begin
  if ListBoxPlaylist.ItemIndex >= 0 then
  begin
    FPlaylist.Remove(ListBoxPlaylist.ItemIndex);
    UpdatePlaylistDisplay;
  end;
end;

procedure TForm1.btnClearClick(Sender: TObject);
begin
  MediaPlayer1.Stop;
  Timer1.Enabled := False;
  FPlaylist.Clear;
  UpdatePlaylistDisplay;
  lblNowPlaying.Caption := 'ไม่มีเพลง';
end;

procedure TForm1.LoadAndPlay(Index: Integer);
var
  Item: TPlaylistItem;
begin
  if (Index < 0) or (Index >= FPlaylist.Count) then Exit;
  
  FPlaylist.SetCurrent(Index);
  Item := FPlaylist.GetCurrent;
  
  MediaPlayer1.Stop;
  MediaPlayer1.FileName := Item.FileName;
  MediaPlayer1.Open;
  MediaPlayer1.Play;
  
  Timer1.Enabled := True;
  
  // Highlight ใน playlist
  ListBoxPlaylist.ItemIndex := Index;
  
  UpdateNowPlaying;
end;

procedure TForm1.UpdatePlaylistDisplay;
var
  i: Integer;
begin
  ListBoxPlaylist.Clear;
  for i := 0 to FPlaylist.Count - 1 do
    ListBoxPlaylist.Items.Add(FPlaylist.Items[i].Title);
end;

procedure TForm1.UpdateNowPlaying;
var
  Item: TPlaylistItem;
begin
  Item := FPlaylist.GetCurrent;
  if Item.FileName <> '' then
    lblNowPlaying.Caption := 'กำลังเล่น: ' + Item.Title
  else
    lblNowPlaying.Caption := 'ไม่มีเพลง';
end;

procedure TForm1.btnPlayClick(Sender: TObject);
begin
  if FPlaylist.Count = 0 then Exit;
  
  if FPlaylist.CurrentIndex < 0 then
    LoadAndPlay(0)
  else
    MediaPlayer1.Play;
    
  Timer1.Enabled := True;
end;

procedure TForm1.btnPauseClick(Sender: TObject);
begin
  case MediaPlayer1.Mode of
    mpPlaying: begin MediaPlayer1.Pause; Timer1.Enabled := False; end;
    mpPaused: begin MediaPlayer1.Resume; Timer1.Enabled := True; end;
  end;
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  MediaPlayer1.Stop;
  Timer1.Enabled := False;
  SeekBar.Position := 0;
end;

procedure TForm1.btnPrevClick(Sender: TObject);
begin
  LoadAndPlay(FPlaylist.GetPrevious);
end;

procedure TForm1.btnNextClick(Sender: TObject);
var
  NextIdx: Integer;
begin
  NextIdx := FPlaylist.GetNext;
  if NextIdx >= 0 then
    LoadAndPlay(NextIdx);
end;

procedure TForm1.ListBoxPlaylistDblClick(Sender: TObject);
begin
  if ListBoxPlaylist.ItemIndex >= 0 then
    LoadAndPlay(ListBoxPlaylist.ItemIndex);
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  if MediaPlayer1.Length > 0 then
  begin
    SeekBar.Position := Round(MediaPlayer1.Position / MediaPlayer1.Length * 1000);
    var S := MediaPlayer1.Position div 1000;
    var D := MediaPlayer1.Length div 1000;
    lblTime.Caption := Format('%d:%02d / %d:%02d', [S div 60, S mod 60, D div 60, D mod 60]);
  end;
end;

procedure TForm1.MediaPlayer1Notify(Sender: TObject);
begin
  if MediaPlayer1.Mode = mpStopped then
  begin
    Timer1.Enabled := False;
    // เล่นเพลงถัดไปอัตโนมัติ
    var NextIdx := FPlaylist.GetNext;
    if NextIdx >= 0 then
      LoadAndPlay(NextIdx);
  end;
end;

end.
```

---

## 38.5 การบันทึกเสียง (Audio Recording)

```pascal
unit AudioRecorder;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  Windows, MMSystem;

type
  TWaveHeader = packed record
    ChunkID: array[0..3] of Char;    // "RIFF"
    ChunkSize: Cardinal;
    Format: array[0..3] of Char;     // "WAVE"
    Subchunk1ID: array[0..3] of Char; // "fmt "
    Subchunk1Size: Cardinal;
    AudioFormat: Word;
    NumChannels: Word;
    SampleRate: Cardinal;
    ByteRate: Cardinal;
    BlockAlign: Word;
    BitsPerSample: Word;
    Subchunk2ID: array[0..3] of Char; // "data"
    Subchunk2Size: Cardinal;
  end;
  
  TRecordingState = (rsStopped, rsRecording, rsPaused);
  
  TForm1 = class(TForm)
    btnRecord: TButton;
    btnPause: TButton;
    btnStop: TButton;
    btnPlay: TButton;
    btnSave: TButton;
    lblStatus: TLabel;
    lblTime: TLabel;
    Timer1: TTimer;
    VUMeter: TProgressBar;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnRecordClick(Sender: TObject);
    procedure btnPauseClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
    procedure btnPlayClick(Sender: TObject);
    procedure btnSaveClick(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
  private
    FState: TRecordingState;
    FRecordingData: TMemoryStream;
    FWaveInHandle: HWAVEIN;
    FRecordBuffer: array[0..65535] of Byte;
    FWaveHeader: TWAVEHDR;
    FStartTime: Cardinal;
    FSampleRate: Integer;
    
    procedure StartRecording;
    procedure StopRecording;
    procedure SaveToWAVFile(const FileName: string);
    function GetRecordingTime: Cardinal;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  FState := rsStopped;
  FRecordingData := TMemoryStream.Create;
  FSampleRate := 44100;
  
  Timer1.Interval := 100;
  Timer1.Enabled := False;
  
  btnRecord.Caption := 'บันทึก';
  btnPause.Enabled := False;
  btnStop.Enabled := False;
  btnPlay.Enabled := False;
  btnSave.Enabled := False;
  
  lblStatus.Caption := 'พร้อม';
  lblTime.Caption := '00:00';
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  if FState <> rsStopped then
    StopRecording;
  FRecordingData.Free;
end;

procedure TForm1.StartRecording;
var
  WaveFormat: TWAVEFORMATEX;
begin
  FRecordingData.Clear;
  
  // ตั้งค่า Wave Format
  WaveFormat.wFormatTag := WAVE_FORMAT_PCM;
  WaveFormat.nChannels := 1;          // Mono
  WaveFormat.nSamplesPerSec := FSampleRate;
  WaveFormat.wBitsPerSample := 16;
  WaveFormat.nBlockAlign := WaveFormat.nChannels * WaveFormat.wBitsPerSample div 8;
  WaveFormat.nAvgBytesPerSec := WaveFormat.nSamplesPerSec * WaveFormat.nBlockAlign;
  WaveFormat.cbSize := 0;
  
  // เปิด Wave Input Device
  if waveInOpen(@FWaveInHandle, WAVE_MAPPER, @WaveFormat, 0, 0, WAVE_FORMAT_QUERY) <> MMSYSERR_NOERROR then
  begin
    ShowMessage('ไม่สามารถเปิดไมโครโฟน');
    Exit;
  end;
  
  waveInOpen(@FWaveInHandle, WAVE_MAPPER, @WaveFormat, 0, 0, CALLBACK_NULL);
  
  // เตรียม Buffer
  FillChar(FWaveHeader, SizeOf(FWaveHeader), 0);
  FWaveHeader.lpData := @FRecordBuffer;
  FWaveHeader.dwBufferLength := SizeOf(FRecordBuffer);
  
  waveInPrepareHeader(FWaveInHandle, @FWaveHeader, SizeOf(FWaveHeader));
  waveInAddBuffer(FWaveInHandle, @FWaveHeader, SizeOf(FWaveHeader));
  
  waveInStart(FWaveInHandle);
  
  FState := rsRecording;
  FStartTime := GetTickCount;
  Timer1.Enabled := True;
  
  lblStatus.Caption := 'กำลังบันทึก...';
end;

procedure TForm1.StopRecording;
begin
  if FState = rsStopped then Exit;
  
  waveInStop(FWaveInHandle);
  waveInReset(FWaveInHandle);
  
  // บันทึกข้อมูลที่ได้
  if FWaveHeader.dwBytesRecorded > 0 then
    FRecordingData.Write(FRecordBuffer, FWaveHeader.dwBytesRecorded);
  
  waveInUnprepareHeader(FWaveInHandle, @FWaveHeader, SizeOf(FWaveHeader));
  waveInClose(FWaveInHandle);
  
  FState := rsStopped;
  Timer1.Enabled := False;
  
  lblStatus.Caption := Format('บันทึกแล้ว (%.1f วินาที)', 
    [FRecordingData.Size / (FSampleRate * 2)]);
end;

procedure TForm1.SaveToWAVFile(const FileName: string);
var
  FileStream: TFileStream;
  Header: TWaveHeader;
  DataSize: Cardinal;
begin
  DataSize := FRecordingData.Size;
  
  // สร้าง WAV header
  Header.ChunkID := 'RIFF';
  Header.ChunkSize := 36 + DataSize;
  Header.Format := 'WAVE';
  Header.Subchunk1ID := 'fmt ';
  Header.Subchunk1Size := 16;
  Header.AudioFormat := 1;  // PCM
  Header.NumChannels := 1;  // Mono
  Header.SampleRate := FSampleRate;
  Header.BitsPerSample := 16;
  Header.BlockAlign := Header.NumChannels * Header.BitsPerSample div 8;
  Header.ByteRate := Header.SampleRate * Header.BlockAlign;
  Header.Subchunk2ID := 'data';
  Header.Subchunk2Size := DataSize;
  
  FileStream := TFileStream.Create(FileName, fmCreate);
  try
    FileStream.Write(Header, SizeOf(Header));
    FRecordingData.Position := 0;
    FileStream.CopyFrom(FRecordingData, FRecordingData.Size);
  finally
    FileStream.Free;
  end;
end;

function TForm1.GetRecordingTime: Cardinal;
begin
  Result := GetTickCount - FStartTime;
end;

procedure TForm1.btnRecordClick(Sender: TObject);
begin
  case FState of
    rsStopped: 
    begin
      StartRecording;
      btnRecord.Enabled := False;
      btnPause.Enabled := True;
      btnStop.Enabled := True;
      btnPlay.Enabled := False;
      btnSave.Enabled := False;
    end;
  end;
end;

procedure TForm1.btnPauseClick(Sender: TObject);
begin
  case FState of
    rsRecording:
    begin
      waveInStop(FWaveInHandle);
      FState := rsPaused;
      lblStatus.Caption := 'หยุดชั่วคราว';
      btnPause.Caption := 'เล่นต่อ';
    end;
    rsPaused:
    begin
      waveInStart(FWaveInHandle);
      FState := rsRecording;
      lblStatus.Caption := 'กำลังบันทึก...';
      btnPause.Caption := 'หยุดชั่วคราว';
    end;
  end;
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  StopRecording;
  btnRecord.Enabled := True;
  btnPause.Enabled := False;
  btnStop.Enabled := False;
  btnPlay.Enabled := FRecordingData.Size > 0;
  btnSave.Enabled := FRecordingData.Size > 0;
end;

procedure TForm1.btnPlayClick(Sender: TObject);
var
  TempFile: string;
begin
  // บันทึกชั่วคราวแล้วเล่น
  TempFile := GetTempDir + 'recording_temp.wav';
  SaveToWAVFile(TempFile);
  
  ShellExecute(0, 'open', PChar(TempFile), nil, nil, SW_SHOWNORMAL);
end;

procedure TForm1.btnSaveClick(Sender: TObject);
var
  SD: TSaveDialog;
begin
  SD := TSaveDialog.Create(nil);
  try
    SD.Filter := 'WAV Files|*.wav';
    SD.DefaultExt := 'wav';
    if SD.Execute then
      SaveToWAVFile(SD.FileName);
  finally
    SD.Free;
  end;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
var
  ElapsedMs: Cardinal;
  Sec, Min: Cardinal;
begin
  if FState = rsRecording then
  begin
    ElapsedMs := GetRecordingTime;
    Sec := ElapsedMs div 1000;
    Min := Sec div 60;
    Sec := Sec mod 60;
    lblTime.Caption := Format('%02d:%02d', [Min, Sec]);
    
    // VU Meter อย่างง่าย (ใช้ค่าสุ่มเป็น simulation)
    VUMeter.Position := Random(80) + 20;
  end;
end;

end.
```

---

## 38.6 Text-to-Speech (TTS)

```pascal
unit TextToSpeech;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  Windows, ComObj, ActiveX;

type
  TSpVoice = interface(IUnknown)
    ['{269316D8-57BD-11D2-9EEE-00C04F797396}']
    function Speak(const pwcs: WideString; dwFlags: Integer; 
                    pStreamNumber: PInteger): HResult; stdcall;
    function SpeakStream(const pStream: IUnknown; dwFlags: Integer;
                          pStreamNumber: PInteger): HResult; stdcall;
    function GetStatus(out pStatus: Pointer; pBookmark: PBSTR): HResult; stdcall;
    function Skip(const pItemType: WideString; lNumItems: Longint;
                   pulNumSkipped: PLongint): HResult; stdcall;
    function SetPriority(ePriority: Integer): HResult; stdcall;
    function GetPriority(out pePriority: Integer): HResult; stdcall;
    function SetAlertBoundary(eBoundary: Integer): HResult; stdcall;
    function GetAlertBoundary(out peBoundary: Integer): HResult; stdcall;
    function SetRate(RateAdjust: Integer): HResult; stdcall;
    function GetRate(out pRateAdjust: Integer): HResult; stdcall;
    function SetVolume(usVolume: Word): HResult; stdcall;
    function GetVolume(out pusVolume: Word): HResult; stdcall;
    function WaitUntilDone(msTimeout: Cardinal): HResult; stdcall;
    function SetSyncSpeakTimeout(msTimeout: Cardinal): HResult; stdcall;
    function GetSyncSpeakTimeout(out pmsTimeout: Cardinal): HResult; stdcall;
    function SpeakCompleteEvent: THandle; stdcall;
    function IsUISupported(const pszTypeOfUI: WideString;
                            pvExtraData: Pointer; cbExtraData: Cardinal;
                            pbSupported: PInteger): HResult; stdcall;
    function DisplayUI(hwndParent: HWND; const pszTitle: WideString;
                        const pszTypeOfUI: WideString; pvExtraData: Pointer;
                        cbExtraData: Cardinal): HResult; stdcall;
    function GetVoices(pAttributes: Pointer; 
                        pRequiredAttributes: Pointer;
                        out ppEnum: Pointer): HResult; stdcall;
    function SetVoice(const pVoice: IUnknown): HResult; stdcall;
    function GetVoice(out ppVoice: IUnknown): HResult; stdcall;
  end;

const
  SPF_DEFAULT = 0;
  SPF_ASYNC = 1;
  SPF_PURGEBEFORESPEAK = 2;
  SPF_IS_FILENAME = 4;
  SPF_IS_XML = 8;
  SPF_IS_NOT_XML = 16;
  
  CLSID_SpVoice: TGUID = '{96749377-3391-11D2-9EE3-00C04F797396}';

type
  TSimpleTTS = class
  private
    FVoice: IDispatch;
    FInitialized: Boolean;
  public
    constructor Create;
    destructor Destroy; override;
    function Speak(const Text: string; Async: Boolean = True): Boolean;
    procedure Stop;
    procedure SetRate(Value: Integer);  // -10 ถึง +10
    procedure SetVolume(Value: Integer);  // 0-100
    function IsAvailable: Boolean;
  end;
  
  TForm1 = class(TForm)
    MemoText: TMemo;
    btnSpeak: TButton;
    btnStop: TButton;
    TrackRate: TTrackBar;
    TrackVolume: TTrackBar;
    lblRate: TLabel;
    lblVolume: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnSpeakClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
    procedure TrackRateChange(Sender: TObject);
    procedure TrackVolumeChange(Sender: TObject);
  private
    FTTS: TSimpleTTS;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

constructor TSimpleTTS.Create;
begin
  inherited;
  FInitialized := False;
  try
    // สร้าง SAPI 5.0 Voice object
    FVoice := CreateComObject(CLSID_SpVoice) as IDispatch;
    FInitialized := True;
  except
    FVoice := nil;
  end;
end;

destructor TSimpleTTS.Destroy;
begin
  FVoice := nil;
  inherited;
end;

function TSimpleTTS.IsAvailable: Boolean;
begin
  Result := FInitialized and Assigned(FVoice);
end;

function TSimpleTTS.Speak(const Text: string; Async: Boolean): Boolean;
var
  Flags: Integer;
begin
  Result := False;
  if not IsAvailable then Exit;
  
  if Async then Flags := SPF_ASYNC
  else Flags := SPF_DEFAULT;
  
  try
    var VText: OleVariant := WideString(Text);
    var VFlags: OleVariant := Integer(Flags);
    FVoice.Invoke(1, GUID_NULL, 0, DISPATCH_METHOD,
                  VText, nil, nil, nil);
    Result := True;
  except
    // ถ้า Invoke ไม่ได้ ใช้ Shell command แทน
    Result := False;
  end;
end;

procedure TSimpleTTS.Stop;
begin
  if not IsAvailable then Exit;
  try
    var Flags: OleVariant := Integer(SPF_PURGEBEFORESPEAK);
    var EmptyStr: OleVariant := WideString('');
    // Stop โดยพูดข้อความว่าง
  except
  end;
end;

procedure TSimpleTTS.SetRate(Value: Integer);
begin
  if not IsAvailable then Exit;
  // Rate ผ่าน IDispatch
end;

procedure TSimpleTTS.SetVolume(Value: Integer);
begin
  if not IsAvailable then Exit;
  // Volume ผ่าน IDispatch
end;

// วิธีง่ายกว่าโดยใช้ PowerShell (Windows)
function SpeakWithPowerShell(const Text: string): Boolean;
var
  PS: string;
begin
  PS := 'powershell -command "Add-Type -AssemblyName System.Speech; ' +
        '(New-Object System.Speech.Synthesis.SpeechSynthesizer).Speak(''' + 
        Text + ''')"';
  Result := ShellExecute(0, 'open', 'cmd.exe', 
    PChar('/c ' + PS), nil, SW_HIDE) > 32;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FTTS := TSimpleTTS.Create;
  
  TrackRate.Min := -10;
  TrackRate.Max := 10;
  TrackRate.Position := 0;
  
  TrackVolume.Min := 0;
  TrackVolume.Max := 100;
  TrackVolume.Position := 80;
  
  if not FTTS.IsAvailable then
    lblRate.Caption := 'TTS ไม่พร้อมใช้งาน (ใช้ PowerShell แทน)';
    
  MemoText.Lines.Add('สวัสดี ยินดีต้อนรับสู่โปรแกรม Text to Speech');
  MemoText.Lines.Add('พิมพ์ข้อความที่ต้องการอ่าน แล้วกดปุ่ม พูด');
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FTTS.Free;
end;

procedure TForm1.btnSpeakClick(Sender: TObject);
var
  Text: string;
begin
  Text := MemoText.Text;
  if Text = '' then Exit;
  
  if FTTS.IsAvailable then
    FTTS.Speak(Text)
  else
    SpeakWithPowerShell(Text);
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  FTTS.Stop;
end;

procedure TForm1.TrackRateChange(Sender: TObject);
begin
  FTTS.SetRate(TrackRate.Position);
  lblRate.Caption := Format('ความเร็ว: %+d', [TrackRate.Position]);
end;

procedure TForm1.TrackVolumeChange(Sender: TObject);
begin
  FTTS.SetVolume(TrackVolume.Position);
  lblVolume.Caption := Format('ระดับเสียง: %d%%', [TrackVolume.Position]);
end;

end.
```

---

## 38.7 เสียงของระบบ (System Sounds)

```pascal
unit SystemSounds;

interface

uses
  Classes, SysUtils, Windows, MMSystem;

// เล่นเสียงระบบ
procedure PlaySystemSound(SoundType: Integer);
// SoundType: 0=Default, 1=Error, 2=Warning, 3=Question, 4=Asterisk

// เล่นไฟล์ WAV แบบ Sync
function PlayWAVSync(const FileName: string): Boolean;

// เล่นไฟล์ WAV แบบ Async
function PlayWAVAsync(const FileName: string): Boolean;

// หยุดเล่น WAV
procedure StopWAV;

// เล่นเสียง Beep
procedure PlayBeep(Frequency, Duration: Integer);

implementation

procedure PlaySystemSound(SoundType: Integer);
const
  Sounds: array[0..4] of LPCTSTR = (
    nil,             // Default
    'SystemExclamation',  // Error
    'SystemQuestion',     // Warning
    'SystemQuestion',     // Question
    'SystemAsterisk'      // Asterisk
  );
begin
  case SoundType of
    0: MessageBeep(MB_OK);
    1: MessageBeep(MB_ICONERROR);
    2: MessageBeep(MB_ICONWARNING);
    3: MessageBeep(MB_ICONQUESTION);
    4: MessageBeep(MB_ICONINFORMATION);
  end;
end;

function PlayWAVSync(const FileName: string): Boolean;
begin
  Result := PlaySound(PChar(FileName), 0, SND_FILENAME or SND_SYNC);
end;

function PlayWAVAsync(const FileName: string): Boolean;
begin
  Result := PlaySound(PChar(FileName), 0, SND_FILENAME or SND_ASYNC);
end;

procedure StopWAV;
begin
  PlaySound(nil, 0, 0);
end;

procedure PlayBeep(Frequency, Duration: Integer);
begin
  Windows.Beep(Frequency, Duration);
end;

// ตัวอย่างการใช้งาน
procedure DemoSystemSounds;
begin
  // เสียง Beep ที่ความถี่ต่างๆ
  PlayBeep(440, 500);   // A4
  PlayBeep(493, 500);   // B4
  PlayBeep(523, 500);   // C5
  
  // เสียงระบบ
  PlaySystemSound(1);  // Error sound
  
  Sleep(1000);
  
  // เล่นไฟล์ WAV
  PlayWAVAsync('C:\Windows\Media\tada.wav');
end;

end.
```

---

## 38.8 Background Music

```pascal
unit BackgroundMusic;

interface

uses
  Classes, SysUtils, MPlayer, ExtCtrls;

type
  TBackgroundMusic = class
  private
    FMediaPlayer: TMediaPlayer;
    FPlaylist: TStringList;
    FCurrentTrack: Integer;
    FLooping: Boolean;
    FVolume: Integer;
    procedure OnMediaNotify(Sender: TObject);
  public
    constructor Create(AOwner: TComponent);
    destructor Destroy; override;
    procedure AddTrack(const FileName: string);
    procedure Play;
    procedure Stop;
    procedure FadeIn(Duration: Integer);
    procedure FadeOut(Duration: Integer);
    property Looping: Boolean read FLooping write FLooping;
    property Volume: Integer read FVolume write FVolume;
  end;

implementation

constructor TBackgroundMusic.Create(AOwner: TComponent);
begin
  inherited Create;
  FMediaPlayer := TMediaPlayer.Create(AOwner);
  FMediaPlayer.Notify := True;
  FMediaPlayer.Wait := False;
  FMediaPlayer.OnNotify := OnMediaNotify;
  FPlaylist := TStringList.Create;
  FCurrentTrack := 0;
  FLooping := True;
  FVolume := 100;
end;

destructor TBackgroundMusic.Destroy;
begin
  FMediaPlayer.Stop;
  FPlaylist.Free;
  inherited;
end;

procedure TBackgroundMusic.AddTrack(const FileName: string);
begin
  FPlaylist.Add(FileName);
end;

procedure TBackgroundMusic.Play;
begin
  if FPlaylist.Count = 0 then Exit;
  FCurrentTrack := FCurrentTrack mod FPlaylist.Count;
  
  FMediaPlayer.FileName := FPlaylist[FCurrentTrack];
  FMediaPlayer.Open;
  FMediaPlayer.Channels := FVolume * 65535 div 100;
  FMediaPlayer.Play;
end;

procedure TBackgroundMusic.Stop;
begin
  FMediaPlayer.Stop;
end;

procedure TBackgroundMusic.OnMediaNotify(Sender: TObject);
begin
  if FMediaPlayer.Mode = mpStopped then
  begin
    if FLooping then
    begin
      Inc(FCurrentTrack);
      if FCurrentTrack >= FPlaylist.Count then
        FCurrentTrack := 0;
      Play;
    end;
  end;
end;

procedure TBackgroundMusic.FadeIn(Duration: Integer);
var
  Timer: TTimer;
  StartVol: Integer;
begin
  StartVol := 0;
  FMediaPlayer.Channels := 0;
  Play;
  
  // อย่างง่าย: ค่อยๆ เพิ่ม Volume
  var StartTime := GetTickCount;
  while GetTickCount - StartTime < Cardinal(Duration) do
  begin
    var Progress := (GetTickCount - StartTime) / Duration;
    FMediaPlayer.Channels := Round(FVolume * Progress * 65535 div 100);
    Application.ProcessMessages;
    Sleep(50);
  end;
  FMediaPlayer.Channels := FVolume * 65535 div 100;
end;

procedure TBackgroundMusic.FadeOut(Duration: Integer);
begin
  var StartTime := GetTickCount;
  var StartVol := FVolume;
  
  while GetTickCount - StartTime < Cardinal(Duration) do
  begin
    var Progress := 1 - (GetTickCount - StartTime) / Duration;
    FMediaPlayer.Channels := Round(StartVol * Progress * 65535 div 100);
    Application.ProcessMessages;
    Sleep(50);
  end;
  
  FMediaPlayer.Stop;
  FMediaPlayer.Channels := FVolume * 65535 div 100;
end;

end.
```

---

## 38.9 ตัวอย่างสมบูรณ์: Media Player Application

```pascal
unit FullMediaPlayer;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  StdCtrls, ExtCtrls, ComCtrls, Buttons, Dialogs,
  MPlayer, Menus, FileUtil;

type
  TPlayMode = (pmNormal, pmRepeat, pmRepeatAll, pmShuffle);
  
  TForm1 = class(TForm)
    // Main layout
    PanelTop: TPanel;
    PanelVideo: TPanel;
    PanelControls: TPanel;
    PanelPlaylist: TPanel;
    
    // Media Player
    MediaPlayer1: TMediaPlayer;
    
    // Playlist
    ListBoxPlaylist: TListBox;
    SplitterMain: TSplitter;
    
    // Controls
    BtnPlay: TSpeedButton;
    BtnPause: TSpeedButton;
    BtnStop: TSpeedButton;
    BtnPrev: TSpeedButton;
    BtnNext: TSpeedButton;
    BtnShuffle: TSpeedButton;
    BtnRepeat: TSpeedButton;
    BtnMute: TSpeedButton;
    BtnPlaylist: TSpeedButton;
    BtnFullscreen: TSpeedButton;
    
    SeekBar: TTrackBar;
    VolumeBar: TTrackBar;
    
    LblTime: TLabel;
    LblTitle: TLabel;
    LblVolume: TLabel;
    
    Timer1: TTimer;
    
    // Menu
    MainMenu1: TMainMenu;
    MenuFile: TMenuItem;
    MenuOpen: TMenuItem;
    MenuOpenURL: TMenuItem;
    MenuSep1: TMenuItem;
    MenuExit: TMenuItem;
    MenuView: TMenuItem;
    MenuPlaylist: TMenuItem;
    MenuOptions: TMenuItem;
    MenuAlwaysOnTop: TMenuItem;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure FormResize(Sender: TObject);
    procedure BtnPlayClick(Sender: TObject);
    procedure BtnPauseClick(Sender: TObject);
    procedure BtnStopClick(Sender: TObject);
    procedure BtnPrevClick(Sender: TObject);
    procedure BtnNextClick(Sender: TObject);
    procedure BtnShuffleClick(Sender: TObject);
    procedure BtnRepeatClick(Sender: TObject);
    procedure BtnMuteClick(Sender: TObject);
    procedure BtnPlaylistClick(Sender: TObject);
    procedure BtnFullscreenClick(Sender: TObject);
    procedure SeekBarChange(Sender: TObject);
    procedure VolumeBarChange(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure MediaPlayer1Notify(Sender: TObject);
    procedure ListBoxPlaylistDblClick(Sender: TObject);
    procedure MenuOpenClick(Sender: TObject);
    procedure MenuOpenURLClick(Sender: TObject);
    procedure MenuExitClick(Sender: TObject);
    procedure MenuPlaylistClick(Sender: TObject);
    procedure MenuAlwaysOnTopClick(Sender: TObject);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
  private
    FPlaylist: TStringList;
    FCurrentIndex: Integer;
    FPlayMode: TPlayMode;
    FMuted: Boolean;
    FLastVolume: Integer;
    FIsFullscreen: Boolean;
    FWindowState: TWindowState;
    FSavedBounds: TRect;
    FDraggingSeek: Boolean;
    
    procedure OpenFile(const FileName: string);
    procedure OpenURL(const URL: string);
    procedure PlayItem(Index: Integer);
    procedure UpdatePlaylistUI;
    procedure UpdateControls;
    procedure UpdateTimeDisplay;
    procedure SetupHotkeys;
    function GetNextIndex: Integer;
    function GetPrevIndex: Integer;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  FPlaylist := TStringList.Create;
  FCurrentIndex := -1;
  FPlayMode := pmNormal;
  FMuted := False;
  FLastVolume := 100;
  FIsFullscreen := False;
  FDraggingSeek := False;
  
  // ตั้งค่า MediaPlayer
  MediaPlayer1.Display := PanelVideo;
  MediaPlayer1.DisplayRect := PanelVideo.ClientRect;
  MediaPlayer1.Notify := True;
  MediaPlayer1.Wait := False;
  
  // ตั้งค่า Controls
  SeekBar.Min := 0;
  SeekBar.Max := 1000;
  VolumeBar.Min := 0;
  VolumeBar.Max := 100;
  VolumeBar.Position := 100;
  
  Timer1.Interval := 250;
  Timer1.Enabled := False;
  
  // รับ Keyboard input
  KeyPreview := True;
  
  UpdateControls;
  
  Caption := 'Lazarus Media Player';
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  MediaPlayer1.Stop;
  FPlaylist.Free;
end;

procedure TForm1.FormResize(Sender: TObject);
begin
  if Assigned(MediaPlayer1) and Assigned(PanelVideo) then
    MediaPlayer1.DisplayRect := PanelVideo.ClientRect;
end;

procedure TForm1.OpenFile(const FileName: string);
begin
  FPlaylist.Add(FileName);
  UpdatePlaylistUI;
  
  if FCurrentIndex < 0 then
    PlayItem(0);
end;

procedure TForm1.PlayItem(Index: Integer);
begin
  if (Index < 0) or (Index >= FPlaylist.Count) then Exit;
  
  FCurrentIndex := Index;
  
  MediaPlayer1.Stop;
  MediaPlayer1.FileName := FPlaylist[Index];
  MediaPlayer1.Open;
  
  // ตั้งค่า Video display
  MediaPlayer1.DisplayRect := PanelVideo.ClientRect;
  
  MediaPlayer1.Play;
  Timer1.Enabled := True;
  
  // อัปเดต UI
  LblTitle.Caption := ExtractFileName(ChangeFileExt(FPlaylist[Index], ''));
  ListBoxPlaylist.ItemIndex := Index;
  
  UpdateControls;
  
  Caption := 'Lazarus Media Player - ' + ExtractFileName(FPlaylist[Index]);
end;

function TForm1.GetNextIndex: Integer;
begin
  case FPlayMode of
    pmNormal:
    begin
      if FCurrentIndex >= FPlaylist.Count - 1 then Result := -1
      else Result := FCurrentIndex + 1;
    end;
    pmRepeat: Result := FCurrentIndex;
    pmRepeatAll: Result := (FCurrentIndex + 1) mod FPlaylist.Count;
    pmShuffle: 
    begin
      if FPlaylist.Count <= 1 then Result := 0
      else
      begin
        Result := Random(FPlaylist.Count);
        while Result = FCurrentIndex do
          Result := Random(FPlaylist.Count);
      end;
    end;
  end;
end;

function TForm1.GetPrevIndex: Integer;
begin
  if FCurrentIndex <= 0 then Result := 0
  else Result := FCurrentIndex - 1;
end;

procedure TForm1.UpdatePlaylistUI;
var
  i: Integer;
begin
  ListBoxPlaylist.Clear;
  for i := 0 to FPlaylist.Count - 1 do
    ListBoxPlaylist.Items.Add(ExtractFileName(FPlaylist[i]));
end;

procedure TForm1.UpdateControls;
var
  HasMedia: Boolean;
  IsPlaying: Boolean;
begin
  HasMedia := MediaPlayer1.FileName <> '';
  IsPlaying := MediaPlayer1.Mode = mpPlaying;
  
  BtnPlay.Enabled := HasMedia and not IsPlaying;
  BtnPause.Enabled := HasMedia and IsPlaying;
  BtnStop.Enabled := HasMedia;
  BtnPrev.Enabled := FCurrentIndex > 0;
  BtnNext.Enabled := (FCurrentIndex < FPlaylist.Count - 1) or
                      (FPlayMode in [pmRepeatAll, pmShuffle]);
  
  BtnRepeat.Down := FPlayMode in [pmRepeat, pmRepeatAll];
  BtnShuffle.Down := FPlayMode = pmShuffle;
  BtnMute.Down := FMuted;
end;

procedure TForm1.UpdateTimeDisplay;
var
  Pos, Len: Integer;
begin
  if MediaPlayer1.Length = 0 then
  begin
    LblTime.Caption := '0:00 / 0:00';
    Exit;
  end;
  
  Pos := MediaPlayer1.Position div 1000;
  Len := MediaPlayer1.Length div 1000;
  LblTime.Caption := Format('%d:%02d / %d:%02d',
    [Pos div 60, Pos mod 60, Len div 60, Len mod 60]);
end;

procedure TForm1.BtnPlayClick(Sender: TObject);
begin
  if FPlaylist.Count = 0 then
  begin
    MenuOpenClick(nil);
    Exit;
  end;
  
  if FCurrentIndex < 0 then
    PlayItem(0)
  else
    MediaPlayer1.Play;
    
  Timer1.Enabled := True;
  UpdateControls;
end;

procedure TForm1.BtnPauseClick(Sender: TObject);
begin
  case MediaPlayer1.Mode of
    mpPlaying: 
    begin
      MediaPlayer1.Pause;
      Timer1.Enabled := False;
    end;
    mpPaused:
    begin
      MediaPlayer1.Resume;
      Timer1.Enabled := True;
    end;
  end;
  UpdateControls;
end;

procedure TForm1.BtnStopClick(Sender: TObject);
begin
  MediaPlayer1.Stop;
  Timer1.Enabled := False;
  SeekBar.Position := 0;
  UpdateTimeDisplay;
  UpdateControls;
end;

procedure TForm1.BtnPrevClick(Sender: TObject);
begin
  PlayItem(GetPrevIndex);
end;

procedure TForm1.BtnNextClick(Sender: TObject);
var
  NextIdx: Integer;
begin
  NextIdx := GetNextIndex;
  if NextIdx >= 0 then
    PlayItem(NextIdx)
  else
    BtnStopClick(nil);
end;

procedure TForm1.BtnShuffleClick(Sender: TObject);
begin
  if FPlayMode = pmShuffle then FPlayMode := pmNormal
  else FPlayMode := pmShuffle;
  UpdateControls;
end;

procedure TForm1.BtnRepeatClick(Sender: TObject);
begin
  case FPlayMode of
    pmNormal: FPlayMode := pmRepeat;
    pmRepeat: FPlayMode := pmRepeatAll;
    pmRepeatAll: FPlayMode := pmNormal;
    else FPlayMode := pmRepeat;
  end;
  UpdateControls;
end;

procedure TForm1.BtnMuteClick(Sender: TObject);
begin
  FMuted := not FMuted;
  if FMuted then
  begin
    FLastVolume := VolumeBar.Position;
    VolumeBar.Position := 0;
    MediaPlayer1.Channels := 0;
  end else
  begin
    VolumeBar.Position := FLastVolume;
    MediaPlayer1.Channels := FLastVolume * 65535 div 100;
  end;
  UpdateControls;
end;

procedure TForm1.BtnPlaylistClick(Sender: TObject);
begin
  PanelPlaylist.Visible := not PanelPlaylist.Visible;
  SplitterMain.Visible := PanelPlaylist.Visible;
end;

procedure TForm1.BtnFullscreenClick(Sender: TObject);
begin
  FIsFullscreen := not FIsFullscreen;
  
  if FIsFullscreen then
  begin
    FSavedBounds := BoundsRect;
    FWindowState := WindowState;
    BorderStyle := bsNone;
    WindowState := wsMaximized;
    PanelControls.Visible := False;
    PanelPlaylist.Visible := False;
  end else
  begin
    BorderStyle := bsSizeable;
    WindowState := FWindowState;
    BoundsRect := FSavedBounds;
    PanelControls.Visible := True;
  end;
end;

procedure TForm1.SeekBarChange(Sender: TObject);
begin
  if MediaPlayer1.Length > 0 then
  begin
    var NewPos := Round(SeekBar.Position / 1000 * MediaPlayer1.Length);
    MediaPlayer1.Position := NewPos;
    UpdateTimeDisplay;
  end;
end;

procedure TForm1.VolumeBarChange(Sender: TObject);
begin
  if not FMuted then
  begin
    MediaPlayer1.Channels := VolumeBar.Position * 65535 div 100;
    LblVolume.Caption := Format('%d%%', [VolumeBar.Position]);
  end;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
begin
  if not FDraggingSeek and (MediaPlayer1.Length > 0) then
    SeekBar.Position := Round(MediaPlayer1.Position / MediaPlayer1.Length * 1000);
  
  UpdateTimeDisplay;
end;

procedure TForm1.MediaPlayer1Notify(Sender: TObject);
begin
  if MediaPlayer1.Mode = mpStopped then
  begin
    Timer1.Enabled := False;
    var NextIdx := GetNextIndex;
    if NextIdx >= 0 then
      PlayItem(NextIdx)
    else
    begin
      SeekBar.Position := 0;
      UpdateControls;
    end;
  end;
end;

procedure TForm1.ListBoxPlaylistDblClick(Sender: TObject);
begin
  if ListBoxPlaylist.ItemIndex >= 0 then
    PlayItem(ListBoxPlaylist.ItemIndex);
end;

procedure TForm1.MenuOpenClick(Sender: TObject);
var
  OD: TOpenDialog;
  i: Integer;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.MultiSelect := True;
    OD.Filter := 'Media Files|*.mp3;*.wav;*.ogg;*.mp4;*.avi;*.mkv;*.mov;*.wmv|All|*.*';
    if OD.Execute then
    begin
      for i := 0 to OD.Files.Count - 1 do
        FPlaylist.Add(OD.Files[i]);
      UpdatePlaylistUI;
      
      if FCurrentIndex < 0 then
        PlayItem(0);
    end;
  finally
    OD.Free;
  end;
end;

procedure TForm1.MenuOpenURLClick(Sender: TObject);
var
  URL: string;
begin
  URL := '';
  if InputQuery('เปิด URL', 'ใส่ URL:', URL) and (URL <> '') then
    OpenURL(URL);
end;

procedure TForm1.OpenURL(const URL: string);
begin
  MediaPlayer1.Stop;
  MediaPlayer1.FileName := URL;
  MediaPlayer1.Open;
  MediaPlayer1.Play;
  Timer1.Enabled := True;
  LblTitle.Caption := URL;
  UpdateControls;
end;

procedure TForm1.MenuExitClick(Sender: TObject);
begin
  Close;
end;

procedure TForm1.MenuPlaylistClick(Sender: TObject);
begin
  BtnPlaylistClick(nil);
end;

procedure TForm1.MenuAlwaysOnTopClick(Sender: TObject);
begin
  MenuAlwaysOnTop.Checked := not MenuAlwaysOnTop.Checked;
  if MenuAlwaysOnTop.Checked then
    FormStyle := fsStayOnTop
  else
    FormStyle := fsNormal;
end;

procedure TForm1.FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  case Key of
    VK_SPACE:
      if MediaPlayer1.Mode = mpPlaying then BtnPauseClick(nil)
      else BtnPlayClick(nil);
    
    VK_LEFT:
      if MediaPlayer1.Mode in [mpPlaying, mpPaused] then
        MediaPlayer1.Position := Max(0, MediaPlayer1.Position - 5000);
    
    VK_RIGHT:
      if MediaPlayer1.Mode in [mpPlaying, mpPaused] then
        MediaPlayer1.Position := Min(MediaPlayer1.Length, 
                                       MediaPlayer1.Position + 5000);
    
    VK_UP:
    begin
      if VolumeBar.Position < 100 then
        VolumeBar.Position := VolumeBar.Position + 5;
    end;
    
    VK_DOWN:
    begin
      if VolumeBar.Position > 0 then
        VolumeBar.Position := VolumeBar.Position - 5;
    end;
    
    VK_F11: BtnFullscreenClick(nil);
    
    VK_ESCAPE:
      if FIsFullscreen then BtnFullscreenClick(nil);
    
    Ord('N'): BtnNextClick(nil);
    Ord('P'): BtnPrevClick(nil);
    Ord('M'): BtnMuteClick(nil);
  end;
end;

procedure TForm1.SetupHotkeys;
begin
  // Setup defined above in FormKeyDown
end;

end.
```

---

## แบบฝึกหัดบทที่ 38

### ข้อ 1: Audio Waveform Visualizer
สร้าง Visualizer ที่แสดง Waveform ของเสียงที่กำลังเล่น

### ข้อ 2: Volume Normalizer
สร้างโปรแกรมวิเคราะห์และปรับ Volume ของไฟล์เสียงหลายไฟล์ให้เท่ากัน

### ข้อ 3: Audio Cutter
สร้างโปรแกรมตัด/ต่อไฟล์ WAV

### ข้อ 4: Screen Recorder
สร้างโปรแกรม Record หน้าจอ (Screenshot แบบ Time-lapse)

### ข้อ 5: Karaoke Player
สร้าง Karaoke Player ที่โหลดไฟล์ LRC (Lyrics) และ Sync กับเพลง

```pascal
// แนวทาง: โหลด LRC file
type
  TLyricLine = record
    TimeMs: Integer;
    Text: string;
  end;

function LoadLRC(const FileName: string): array of TLyricLine;
var
  Lines: TStringList;
  i: Integer;
  // Parse [mm:ss.xx]Text format
begin
  // Implementation here
end;
```

### ข้อ 6: Podcast Player
สร้าง Podcast Player ที่โหลด RSS feed และเล่น MP3 stream

### ข้อ 7: Sound Effects Board
สร้าง Sound Effect Board ที่มีปุ่มเล่น Sound Effects ต่างๆ (เหมาะกับ Streamer)

### ข้อ 8: Noise Generator
สร้าง White Noise / Brown Noise Generator เพื่อช่วยในการนอนหรือ Focus

### ข้อ 9: Audio Equalizer
สร้าง Equalizer ที่มี Band ปรับ Bass, Mid, Treble

### ข้อ 10: Metronome
สร้าง Metronome ที่ปรับ BPM ได้ พร้อม Visual Beat indicator

```pascal
// แนวทาง:
type
  TMetronome = class
  private
    FTimer: TTimer;
    FBPM: Integer;
    FBeat: Integer;
    FBeatsPerBar: Integer;
    procedure OnTimer(Sender: TObject);
  public
    constructor Create;
    procedure Start;
    procedure Stop;
    property BPM: Integer read FBPM write FBPM;
    property BeatsPerBar: Integer read FBeatsPerBar write FBeatsPerBar;
    OnBeat: procedure(Beat: Integer) of object;
  end;
  
procedure TMetronome.OnTimer(Sender: TObject);
begin
  Inc(FBeat);
  if FBeat > FBeatsPerBar then FBeat := 1;
  
  if FBeat = 1 then
    PlayBeep(880, 50)  // เสียงแรงที่จังหวะที่ 1
  else
    PlayBeep(440, 30);  // เสียงเบา
    
  if Assigned(OnBeat) then OnBeat(FBeat);
end;
```

---

## สรุปบทที่ 38

ในบทนี้เราได้เรียนรู้:
1. **การเล่นเสียง** - TMediaPlayer, Bass Library
2. **Video Playback** - MediaPlayer กับ Video Panel
3. **TMediaPlayer** - Open, Play, Pause, Stop, Seek
4. **Volume Control** - การปรับระดับเสียง
5. **Playlist Management** - การจัดการ Playlist
6. **Audio Recording** - WaveIn API บน Windows
7. **Text-to-Speech** - SAPI 5.0, PowerShell TTS
8. **System Sounds** - PlaySound, MessageBeep
9. **Background Music** - เล่นเพลงพื้นหลังในแอป
10. **Complete Examples** - Full Media Player Application
