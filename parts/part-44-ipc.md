# Part 44 - Inter-Process Communication (IPC) ใน Lazarus/Pascal

## บทนำ

Inter-Process Communication (IPC) คือกลไกที่ช่วยให้ processes ต่างๆ สามารถสื่อสารและแลกเปลี่ยนข้อมูลกันได้ ใน Lazarus/FPC มีหลายวิธีในการทำ IPC เช่น Pipes, Shared Memory, Sockets และ Message Queues

---

## 44.1 Pipes

### Anonymous Pipes

```pascal
program AnonymousPipes;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Process;

// ใช้ TProcess เพื่อสื่อสารกับ child process ผ่าน pipes
procedure RunChildWithPipes;
var
  AProcess: TProcess;
  OutputLines: TStringList;
  OutputStr: string;
  BytesRead: integer;
  Buffer: array[0..4095] of byte;
begin
  AProcess := TProcess.Create(nil);
  OutputLines := TStringList.Create;
  
  try
    // กำหนด command ที่จะ run
    {$IFDEF WINDOWS}
    AProcess.Executable := 'cmd.exe';
    AProcess.Parameters.Add('/c');
    AProcess.Parameters.Add('dir /b');
    {$ELSE}
    AProcess.Executable := '/bin/ls';
    AProcess.Parameters.Add('-la');
    {$ENDIF}
    
    // เปิด option สำหรับ read output
    AProcess.Options := AProcess.Options + [poUsePipes];
    AProcess.ShowWindow := swoHide;
    
    // เริ่ม process
    AProcess.Execute;
    
    // อ่าน output
    OutputStr := '';
    repeat
      BytesRead := AProcess.Output.Read(Buffer, SizeOf(Buffer));
      if BytesRead > 0 then
      begin
        SetLength(OutputStr, Length(OutputStr) + BytesRead);
        Move(Buffer[0], OutputStr[Length(OutputStr) - BytesRead + 1], BytesRead);
      end;
    until BytesRead = 0;
    
    // รอ process จบ
    AProcess.WaitOnExit;
    
    // แสดงผล
    OutputLines.Text := OutputStr;
    WriteLn('Output from child process:');
    WriteLn('Exit code: ', AProcess.ExitCode);
    WriteLn('Lines: ', OutputLines.Count);
    WriteLn('---');
    var i: integer;
    for i := 0 to Min(9, OutputLines.Count - 1) do
      WriteLn('  ', OutputLines[i]);
    if OutputLines.Count > 10 then
      WriteLn('  ... (', OutputLines.Count - 10, ' more lines)');
      
  finally
    AProcess.Free;
    OutputLines.Free;
  end;
end;

// ส่ง input ไปยัง process ผ่าน stdin pipe
procedure RunWithInput;
var
  AProcess: TProcess;
  InputData: string;
  BytesWritten: integer;
  OutputStr: string;
  Buffer: array[0..4095] of byte;
  BytesRead: integer;
begin
  AProcess := TProcess.Create(nil);
  try
    {$IFDEF UNIX}
    AProcess.Executable := '/usr/bin/sort';
    AProcess.Options := AProcess.Options + [poUsePipes];
    AProcess.Execute;
    
    // ส่ง input ผ่าน stdin
    InputData := 'กล้วย' + #10 + 'แอปเปิ้ล' + #10 + 'ส้ม' + #10 + 'มะม่วง' + #10;
    BytesWritten := AProcess.Input.Write(InputData[1], Length(InputData));
    AProcess.CloseInput;
    
    WriteLn('Sent ', BytesWritten, ' bytes to stdin');
    
    // อ่าน output
    OutputStr := '';
    AProcess.WaitOnExit;
    
    repeat
      BytesRead := AProcess.Output.Read(Buffer, SizeOf(Buffer));
      if BytesRead > 0 then
      begin
        SetLength(OutputStr, Length(OutputStr) + BytesRead);
        Move(Buffer[0], OutputStr[Length(OutputStr) - BytesRead + 1], BytesRead);
      end;
    until BytesRead = 0;
    
    WriteLn('Sorted output:');
    WriteLn(OutputStr);
    {$ELSE}
    WriteLn('(Unix-only example)');
    {$ENDIF}
    
  finally
    AProcess.Free;
  end;
end;

begin
  WriteLn('=== Anonymous Pipes Demo ===');
  WriteLn('');
  RunChildWithPipes;
  WriteLn('');
  RunWithInput;
end.
```

---

## 44.2 Named Pipes

### Named Pipes บน Windows

```pascal
program NamedPipesWindows;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Windows, SysUtils, Classes;

const
  PIPE_NAME = '\\.\pipe\MyAppPipe';
  BUFFER_SIZE = 4096;

type
  TPipeServer = class(TThread)
  private
    FPipeHandle: HANDLE;
    FRunning: boolean;
    
    procedure HandleClient(PipeHandle: HANDLE);
  protected
    procedure Execute; override;
  public
    constructor Create;
    procedure Stop;
  end;

constructor TPipeServer.Create;
begin
  inherited Create(True);
  FPipeHandle := INVALID_HANDLE_VALUE;
  FRunning := False;
  FreeOnTerminate := False;
end;

procedure TPipeServer.HandleClient(PipeHandle: HANDLE);
var
  Buffer: array[0..BUFFER_SIZE-1] of char;
  BytesRead, BytesWritten: DWORD;
  Request, Response: string;
begin
  WriteLn('[Server] Client connected');
  
  // อ่าน request
  FillChar(Buffer, SizeOf(Buffer), 0);
  if ReadFile(PipeHandle, Buffer, SizeOf(Buffer) - 1, BytesRead, nil) then
  begin
    SetString(Request, PChar(Buffer), BytesRead);
    WriteLn('[Server] Received: ', Trim(Request));
    
    // สร้าง response
    Response := 'Echo: ' + Trim(Request) + ' [Processed at ' + TimeToStr(Now) + ']';
    
    // ส่ง response
    WriteFile(PipeHandle, Response[1], Length(Response), BytesWritten, nil);
    WriteLn('[Server] Sent response (', BytesWritten, ' bytes)');
  end;
  
  DisconnectNamedPipe(PipeHandle);
  CloseHandle(PipeHandle);
  WriteLn('[Server] Client disconnected');
end;

procedure TPipeServer.Stop;
begin
  FRunning := False;
  // ยกเลิก blocking ConnectNamedPipe
  if FPipeHandle <> INVALID_HANDLE_VALUE then
    CancelIoEx(FPipeHandle, nil);
end;

procedure TPipeServer.Execute;
var
  NewPipe: HANDLE;
begin
  FRunning := True;
  WriteLn('[Server] Starting on ', PIPE_NAME);
  
  while FRunning and not Terminated do
  begin
    // สร้าง named pipe instance
    FPipeHandle := CreateNamedPipe(
      PIPE_NAME,
      PIPE_ACCESS_DUPLEX,
      PIPE_TYPE_MESSAGE or PIPE_READMODE_MESSAGE or PIPE_WAIT,
      PIPE_UNLIMITED_INSTANCES,
      BUFFER_SIZE,
      BUFFER_SIZE,
      0,
      nil
    );
    
    if FPipeHandle = INVALID_HANDLE_VALUE then
    begin
      WriteLn('[Server] CreateNamedPipe failed: ', GetLastError);
      Break;
    end;
    
    WriteLn('[Server] Waiting for client...');
    
    // รอ client เชื่อมต่อ
    if ConnectNamedPipe(FPipeHandle, nil) or (GetLastError = ERROR_PIPE_CONNECTED) then
    begin
      NewPipe := FPipeHandle;
      FPipeHandle := INVALID_HANDLE_VALUE;
      HandleClient(NewPipe);
    end
    else
    begin
      CloseHandle(FPipeHandle);
      FPipeHandle := INVALID_HANDLE_VALUE;
    end;
    
    if not FRunning then Break;
  end;
  
  WriteLn('[Server] Stopped');
end;

procedure RunPipeClient(const Message: string);
var
  PipeHandle: HANDLE;
  BytesRead, BytesWritten: DWORD;
  Buffer: array[0..BUFFER_SIZE-1] of char;
  Response: string;
  Retries: integer;
begin
  WriteLn('[Client] Connecting to pipe...');
  
  // รอจนมี pipe ว่าง
  Retries := 10;
  repeat
    if WaitNamedPipe(PIPE_NAME, 5000) then Break;
    Dec(Retries);
    Sleep(200);
  until Retries = 0;
  
  PipeHandle := CreateFile(
    PIPE_NAME,
    GENERIC_READ or GENERIC_WRITE,
    0,
    nil,
    OPEN_EXISTING,
    0,
    0
  );
  
  if PipeHandle = INVALID_HANDLE_VALUE then
  begin
    WriteLn('[Client] Failed to connect: ', GetLastError);
    Exit;
  end;
  
  try
    // ส่ง message
    WriteFile(PipeHandle, Message[1], Length(Message), BytesWritten, nil);
    WriteLn('[Client] Sent: ', Message, ' (', BytesWritten, ' bytes)');
    
    // อ่าน response
    FillChar(Buffer, SizeOf(Buffer), 0);
    if ReadFile(PipeHandle, Buffer, SizeOf(Buffer) - 1, BytesRead, nil) then
    begin
      SetString(Response, PChar(Buffer), BytesRead);
      WriteLn('[Client] Received: ', Trim(Response));
    end;
  finally
    CloseHandle(PipeHandle);
  end;
end;

var
  Server: TPipeServer;
  i: integer;
begin
  WriteLn('=== Named Pipes Demo (Windows) ===');
  WriteLn('');
  
  Server := TPipeServer.Create;
  try
    Server.Start;
    Sleep(200); // รอ server เริ่ม
    
    // ส่ง messages หลายอัน
    for i := 1 to 3 do
    begin
      Sleep(100);
      RunPipeClient('Hello from client ' + IntToStr(i));
      WriteLn('');
    end;
    
    Sleep(500);
    Server.Stop;
    Server.WaitFor;
    
  finally
    Server.Free;
  end;
end.

{$ELSE}
begin
  WriteLn('Named Pipes demo is Windows-only');
  WriteLn('On Linux/macOS, use UNIX domain sockets instead');
end.
{$ENDIF}
```

---

## 44.3 Shared Memory

### การใช้ Shared Memory

```pascal
program SharedMemory;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes
  {$IFDEF WINDOWS}, Windows{$ENDIF}
  {$IFDEF UNIX}, BaseUnix, UnixType, PThreads{$ENDIF};

type
  // Data structure ใน shared memory
  TSharedData = packed record
    Counter: integer;
    LastUpdate: TDateTime;
    Message: array[0..255] of char;
    Ready: boolean;
  end;
  PSharedData = ^TSharedData;

  TSharedMemory = class
  private
    FName: string;
    FSize: integer;
    FData: Pointer;
    {$IFDEF WINDOWS}
    FMapHandle: HANDLE;
    {$ENDIF}
    {$IFDEF UNIX}
    FSHMID: integer;
    FFilename: string;
    {$ENDIF}
    FIsOwner: boolean;
  public
    constructor Create(const Name: string; Size: integer; Owner: boolean = True);
    destructor Destroy; override;
    
    function GetPointer: Pointer;
    property Size: integer read FSize;
    property IsOwner: boolean read FIsOwner;
  end;

constructor TSharedMemory.Create(const Name: string; Size: integer; Owner: boolean);
begin
  inherited Create;
  FName := Name;
  FSize := Size;
  FIsOwner := Owner;
  FData := nil;
  
  {$IFDEF WINDOWS}
  if Owner then
  begin
    FMapHandle := CreateFileMapping(
      INVALID_HANDLE_VALUE,
      nil,
      PAGE_READWRITE,
      0,
      Size,
      PChar(Name)
    );
  end
  else
  begin
    FMapHandle := OpenFileMapping(
      FILE_MAP_ALL_ACCESS,
      False,
      PChar(Name)
    );
  end;
  
  if FMapHandle = 0 then
    raise Exception.CreateFmt('Cannot create/open shared memory "%s": %d', 
      [Name, GetLastError]);
      
  FData := MapViewOfFile(FMapHandle, FILE_MAP_ALL_ACCESS, 0, 0, Size);
  if FData = nil then
    raise Exception.Create('MapViewOfFile failed: ' + IntToStr(GetLastError));
    
  if Owner then
    FillChar(FData^, Size, 0);
  {$ENDIF}
  
  {$IFDEF UNIX}
  // ใช้ POSIX shared memory
  var Flags: cint;
  if Owner then
    Flags := O_CREAT or O_RDWR
  else
    Flags := O_RDWR;
    
  var ShmFd := fpshm_open(PChar('/' + Name), Flags, $1B6); // 0666
  if ShmFd < 0 then
    raise Exception.CreateFmt('shm_open failed for "%s"', [Name]);
  
  if Owner then
    fpftruncate(ShmFd, Size);
    
  FData := fpmmap(nil, Size, PROT_READ or PROT_WRITE, MAP_SHARED, ShmFd, 0);
  fpClose(ShmFd);
  
  if FData = MAP_FAILED then
    raise Exception.Create('mmap failed');
    
  if Owner then
    FillChar(FData^, Size, 0);
  {$ENDIF}
end;

destructor TSharedMemory.Destroy;
begin
  {$IFDEF WINDOWS}
  if Assigned(FData) then
    UnmapViewOfFile(FData);
  if FMapHandle <> 0 then
    CloseHandle(FMapHandle);
  {$ENDIF}
  {$IFDEF UNIX}
  if Assigned(FData) and (FData <> MAP_FAILED) then
    fpmunmap(FData, FSize);
  if FIsOwner then
    fpshm_unlink(PChar('/' + FName));
  {$ENDIF}
  inherited Destroy;
end;

function TSharedMemory.GetPointer: Pointer;
begin
  Result := FData;
end;

// IPC Channel ที่ใช้ shared memory
type
  TIPCChannel = class
  private
    FSharedMem: TSharedMemory;
    FLock: TCriticalSection;
    
    function GetData: PSharedData;
  public
    constructor Create(const ChannelName: string; IsServer: boolean);
    destructor Destroy; override;
    
    procedure WriteMessage(const Msg: string);
    function ReadMessage: string;
    procedure IncrementCounter;
    function GetCounter: integer;
    function IsReady: boolean;
  end;

constructor TIPCChannel.Create(const ChannelName: string; IsServer: boolean);
begin
  inherited Create;
  FLock := TCriticalSection.Create;
  FSharedMem := TSharedMemory.Create(ChannelName, SizeOf(TSharedData), IsServer);
end;

destructor TIPCChannel.Destroy;
begin
  FSharedMem.Free;
  FLock.Free;
  inherited Destroy;
end;

function TIPCChannel.GetData: PSharedData;
begin
  Result := PSharedData(FSharedMem.GetPointer);
end;

procedure TIPCChannel.WriteMessage(const Msg: string);
var
  MsgLen: integer;
begin
  FLock.Acquire;
  try
    MsgLen := Min(Length(Msg), SizeOf(TSharedData.Message) - 1);
    Move(Msg[1], GetData^.Message[0], MsgLen);
    GetData^.Message[MsgLen] := #0;
    GetData^.LastUpdate := Now;
  finally
    FLock.Release;
  end;
end;

function TIPCChannel.ReadMessage: string;
begin
  FLock.Acquire;
  try
    Result := string(PChar(@GetData^.Message[0]));
  finally
    FLock.Release;
  end;
end;

procedure TIPCChannel.IncrementCounter;
begin
  FLock.Acquire;
  try
    Inc(GetData^.Counter);
  finally
    FLock.Release;
  end;
end;

function TIPCChannel.GetCounter: integer;
begin
  FLock.Acquire;
  try
    Result := GetData^.Counter;
  finally
    FLock.Release;
  end;
end;

function TIPCChannel.IsReady: boolean;
begin
  Result := GetData^.Ready;
end;

var
  Channel: TIPCChannel;
  i: integer;
begin
  WriteLn('=== Shared Memory Demo ===');
  WriteLn('');
  
  // สร้าง shared memory channel
  Channel := TIPCChannel.Create('MyAppChannel', True);
  try
    WriteLn('Shared memory created');
    
    // เขียนข้อมูล
    Channel.WriteMessage('Hello from Process A!');
    WriteLn('Message written: ', Channel.ReadMessage);
    
    // Increment counter
    for i := 1 to 10 do
      Channel.IncrementCounter;
    
    WriteLn('Counter after 10 increments: ', Channel.GetCounter);
    
    // Update message
    Channel.WriteMessage('Updated at ' + TimeToStr(Now));
    WriteLn('Updated message: ', Channel.ReadMessage);
    
  finally
    Channel.Free;
  end;
  
  WriteLn('');
  WriteLn('Shared Memory demo complete!');
end.
```

---

## 44.4 Message Queues

### Message Queue Implementation

```pascal
program MessageQueues;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Contnrs;

type
  TMessageType = (mtData, mtControl, mtError, mtShutdown);
  
  TIPCMessage = class
  private
    FMessageType: TMessageType;
    FPayload: string;
    FPriority: integer;
    FTimestamp: TDateTime;
    FSenderID: string;
  public
    constructor Create(MsgType: TMessageType; const Payload: string;
      Priority: integer = 5; const SenderID: string = '');
    
    property MessageType: TMessageType read FMessageType;
    property Payload: string read FPayload;
    property Priority: integer read FPriority;
    property Timestamp: TDateTime read FTimestamp;
    property SenderID: string read FSenderID;
  end;
  
  TPriorityMessageQueue = class
  private
    FMessages: TObjectList;
    FLock: TCriticalSection;
    FNotEmpty: TEvent;
    FMaxSize: integer;
    
    function GetCount: integer;
    procedure SortByPriority;
  public
    constructor Create(MaxSize: integer = 1000);
    destructor Destroy; override;
    
    function Enqueue(Message: TIPCMessage): boolean;
    function Dequeue(TimeoutMs: cardinal = INFINITE): TIPCMessage;
    function TryDequeue(out Message: TIPCMessage): boolean;
    procedure Clear;
    
    property Count: integer read GetCount;
    property MaxSize: integer read FMaxSize;
  end;
  
  // Message Worker
  TMessageWorker = class(TThread)
  private
    FQueue: TPriorityMessageQueue;
    FWorkerID: string;
    FProcessedCount: integer;
    
    procedure ProcessMessage(Msg: TIPCMessage);
  protected
    procedure Execute; override;
  public
    constructor Create(Queue: TPriorityMessageQueue; const WorkerID: string);
    property ProcessedCount: integer read FProcessedCount;
  end;

constructor TIPCMessage.Create(MsgType: TMessageType; const Payload: string;
  Priority: integer; const SenderID: string);
begin
  inherited Create;
  FMessageType := MsgType;
  FPayload := Payload;
  FPriority := Priority;
  FTimestamp := Now;
  FSenderID := SenderID;
end;

constructor TPriorityMessageQueue.Create(MaxSize: integer);
begin
  inherited Create;
  FMaxSize := MaxSize;
  FMessages := TObjectList.Create(True); // owns objects
  FLock := TCriticalSection.Create;
  FNotEmpty := TEvent.Create(nil, False, False, '');
end;

destructor TPriorityMessageQueue.Destroy;
begin
  FNotEmpty.Free;
  FLock.Free;
  FMessages.Free;
  inherited Destroy;
end;

function TPriorityMessageQueue.GetCount: integer;
begin
  FLock.Acquire;
  try
    Result := FMessages.Count;
  finally
    FLock.Release;
  end;
end;

procedure TPriorityMessageQueue.SortByPriority;
var
  i, j: integer;
  MsgI, MsgJ: TIPCMessage;
begin
  // Bubble sort (ใน production ควรใช้ algorithm ที่ดีกว่า)
  for i := 0 to FMessages.Count - 2 do
    for j := 0 to FMessages.Count - 2 - i do
    begin
      MsgI := TIPCMessage(FMessages[j]);
      MsgJ := TIPCMessage(FMessages[j + 1]);
      if MsgJ.Priority > MsgI.Priority then
        FMessages.Exchange(j, j + 1);
    end;
end;

function TPriorityMessageQueue.Enqueue(Message: TIPCMessage): boolean;
begin
  FLock.Acquire;
  try
    if FMessages.Count >= FMaxSize then
    begin
      Result := False;
      Exit;
    end;
    
    FMessages.Add(Message);
    SortByPriority;
    Result := True;
  finally
    FLock.Release;
  end;
  
  FNotEmpty.SetEvent;
end;

function TPriorityMessageQueue.Dequeue(TimeoutMs: cardinal): TIPCMessage;
begin
  Result := nil;
  
  if FNotEmpty.WaitFor(TimeoutMs) <> wrSignaled then
    Exit;
  
  FLock.Acquire;
  try
    if FMessages.Count > 0 then
    begin
      Result := TIPCMessage(FMessages.Extract(FMessages.First));
      if FMessages.Count > 0 then
        FNotEmpty.SetEvent;
    end;
  finally
    FLock.Release;
  end;
end;

function TPriorityMessageQueue.TryDequeue(out Message: TIPCMessage): boolean;
begin
  Message := nil;
  
  FLock.Acquire;
  try
    if FMessages.Count = 0 then
      Exit(False);
      
    Message := TIPCMessage(FMessages.Extract(FMessages.First));
    Result := True;
  finally
    FLock.Release;
  end;
end;

procedure TPriorityMessageQueue.Clear;
begin
  FLock.Acquire;
  try
    FMessages.Clear;
  finally
    FLock.Release;
  end;
end;

constructor TMessageWorker.Create(Queue: TPriorityMessageQueue; const WorkerID: string);
begin
  inherited Create(True);
  FQueue := Queue;
  FWorkerID := WorkerID;
  FProcessedCount := 0;
  FreeOnTerminate := False;
end;

procedure TMessageWorker.ProcessMessage(Msg: TIPCMessage);
const
  MsgTypeNames: array[TMessageType] of string = (
    'Data', 'Control', 'Error', 'Shutdown'
  );
begin
  case Msg.MessageType of
    mtData:
      WriteLn(Format('[%s] Processing DATA (P%d): %s', [
        FWorkerID, Msg.Priority, Msg.Payload
      ]));
    mtControl:
    begin
      WriteLn(Format('[%s] Processing CONTROL: %s', [FWorkerID, Msg.Payload]));
      Sleep(50); // control messages ใช้เวลาน้อยกว่า
    end;
    mtError:
      WriteLn(Format('[%s] Handling ERROR: %s', [FWorkerID, Msg.Payload]));
    mtShutdown:
    begin
      WriteLn(Format('[%s] Received SHUTDOWN signal', [FWorkerID]));
      Terminate;
    end;
  end;
  
  Inc(FProcessedCount);
  Sleep(100 + Random(200)); // simulate processing time
end;

procedure TMessageWorker.Execute;
var
  Msg: TIPCMessage;
begin
  WriteLn('[', FWorkerID, '] Started');
  
  while not Terminated do
  begin
    Msg := FQueue.Dequeue(500); // รอ 500ms
    
    if Assigned(Msg) then
    begin
      try
        ProcessMessage(Msg);
      finally
        Msg.Free;
      end;
    end;
  end;
  
  WriteLn('[', FWorkerID, '] Stopped, processed ', FProcessedCount, ' messages');
end;

var
  Queue: TPriorityMessageQueue;
  Workers: array[0..1] of TMessageWorker;
  i: integer;
begin
  Randomize;
  Queue := TPriorityMessageQueue.Create(50);
  
  try
    WriteLn('=== Message Queue Demo ===');
    WriteLn('');
    
    // สร้าง workers
    for i := 0 to 1 do
    begin
      Workers[i] := TMessageWorker.Create(Queue, Format('Worker-%d', [i + 1]));
      Workers[i].Start;
    end;
    
    Sleep(100);
    
    // ส่ง messages ด้วย priorities ต่างกัน
    WriteLn('Sending messages...');
    WriteLn('');
    
    Queue.Enqueue(TIPCMessage.Create(mtData, 'Low priority data', 1, 'Producer'));
    Queue.Enqueue(TIPCMessage.Create(mtError, 'Critical error!', 10, 'Monitor'));
    Queue.Enqueue(TIPCMessage.Create(mtData, 'Normal data', 5, 'Producer'));
    Queue.Enqueue(TIPCMessage.Create(mtControl, 'Flush cache', 8, 'Admin'));
    Queue.Enqueue(TIPCMessage.Create(mtData, 'Another data', 3, 'Producer'));
    Queue.Enqueue(TIPCMessage.Create(mtControl, 'Reload config', 7, 'Admin'));
    Queue.Enqueue(TIPCMessage.Create(mtData, 'Important data', 9, 'Producer'));
    
    // รอให้ messages ถูกประมวลผล
    Sleep(3000);
    
    // ส่ง shutdown signals
    for i := 0 to 1 do
      Queue.Enqueue(TIPCMessage.Create(mtShutdown, 'Stop', 10));
    
    // รอ workers จบ
    for i := 0 to 1 do
      Workers[i].WaitFor;
    
    WriteLn('');
    WriteLn('Total messages processed:');
    for i := 0 to 1 do
      WriteLn(Format('  Worker-%d: %d messages', [i+1, Workers[i].ProcessedCount]));
    
  finally
    for i := 0 to 1 do
      Workers[i].Free;
    Queue.Free;
  end;
end.
```

---

## 44.5 Signals (Unix)

### การใช้ Signals บน Unix

```pascal
program UnixSignals;

{$mode objfpc}{$H+}

{$IFDEF UNIX}
uses
  BaseUnix, UnixType, SysUtils, Classes;

var
  GRunning: boolean = True;
  GSignalReceived: integer = 0;

// Signal handler
procedure SigHandler(Signal: cint); cdecl;
begin
  GSignalReceived := Signal;
  
  case Signal of
    SIGINT:
    begin
      WriteLn('');
      WriteLn('[Signal] SIGINT received (Ctrl+C) - initiating shutdown...');
      GRunning := False;
    end;
    SIGTERM:
    begin
      WriteLn('[Signal] SIGTERM received - terminating...');
      GRunning := False;
    end;
    SIGHUP:
      WriteLn('[Signal] SIGHUP received - reloading config...');
    SIGUSR1:
      WriteLn('[Signal] SIGUSR1 received - custom action 1');
    SIGUSR2:
      WriteLn('[Signal] SIGUSR2 received - custom action 2');
  end;
end;

procedure SetupSignalHandlers;
var
  SigAction: SigActionRec;
begin
  FillChar(SigAction, SizeOf(SigAction), 0);
  SigAction.__sigaction_handler.sa_handler := @SigHandler;
  sigemptyset(SigAction.sa_mask);
  SigAction.sa_flags := SA_RESTART;
  
  // Register handlers
  fpSigAction(SIGINT, @SigAction, nil);
  fpSigAction(SIGTERM, @SigAction, nil);
  fpSigAction(SIGHUP, @SigAction, nil);
  fpSigAction(SIGUSR1, @SigAction, nil);
  fpSigAction(SIGUSR2, @SigAction, nil);
  
  WriteLn('Signal handlers registered');
  WriteLn('PID: ', fpGetPid);
  WriteLn('Press Ctrl+C to stop, or send signals:');
  WriteLn('  kill -HUP ', fpGetPid, '  (reload)');
  WriteLn('  kill -USR1 ', fpGetPid, '  (action 1)');
  WriteLn('  kill -USR2 ', fpGetPid, '  (action 2)');
end;

// ส่ง signal ไปยัง process อื่น
procedure SendSignalToProcess(PID: integer; Signal: cint);
begin
  if fpKill(PID, Signal) = 0 then
    WriteLn('Signal ', Signal, ' sent to PID ', PID)
  else
    WriteLn('Failed to send signal: ', fpGetErrno);
end;

begin
  WriteLn('=== Unix Signals Demo ===');
  WriteLn('');
  
  SetupSignalHandlers;
  WriteLn('');
  
  // Main loop
  var Counter := 0;
  while GRunning do
  begin
    Inc(Counter);
    Write(#13'Running... (', Counter, ')    ');
    Flush(Output);
    Sleep(1000);
    
    if Counter >= 10 then // จำกัดเวลาสำหรับ demo
    begin
      WriteLn('');
      WriteLn('Demo timeout reached, exiting...');
      Break;
    end;
  end;
  
  WriteLn('');
  WriteLn('Cleanup and exit');
end.

{$ELSE}
begin
  WriteLn('Unix Signals demo is Unix-only');
end.
{$ENDIF}
```

---

## 44.6 DDE (Windows)

### DDE (Dynamic Data Exchange) บน Windows

```pascal
program DDEDemo;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Windows, SysUtils, DdeMan;

// DDE Client ที่สื่อสารกับ Excel หรือ application อื่น
type
  TDDEClient = class
  private
    FDDEClient: TDDEClientConv;
    FConnected: boolean;
  public
    constructor Create;
    destructor Destroy; override;
    
    function Connect(const ServiceName, TopicName: string): boolean;
    procedure Disconnect;
    function Request(const ItemName: string): string;
    function Execute(const Command: string): boolean;
    function Poke(const ItemName, Value: string): boolean;
  end;

constructor TDDEClient.Create;
begin
  inherited Create;
  FDDEClient := TDDEClientConv.Create(nil);
  FConnected := False;
end;

destructor TDDEClient.Destroy;
begin
  Disconnect;
  FDDEClient.Free;
  inherited Destroy;
end;

function TDDEClient.Connect(const ServiceName, TopicName: string): boolean;
begin
  try
    Result := FDDEClient.OpenLink(ServiceName, TopicName);
    FConnected := Result;
    if Result then
      WriteLn('DDE Connected to ', ServiceName, '|', TopicName)
    else
      WriteLn('DDE Connection failed');
  except
    on E: Exception do
    begin
      WriteLn('DDE Error: ', E.Message);
      Result := False;
    end;
  end;
end;

procedure TDDEClient.Disconnect;
begin
  if FConnected then
  begin
    FDDEClient.CloseLink;
    FConnected := False;
    WriteLn('DDE Disconnected');
  end;
end;

function TDDEClient.Request(const ItemName: string): string;
var
  Data: string;
begin
  Result := '';
  if not FConnected then Exit;
  
  Data := FDDEClient.RequestData(ItemName);
  Result := Data;
end;

function TDDEClient.Execute(const Command: string): boolean;
begin
  Result := False;
  if not FConnected then Exit;
  
  Result := FDDEClient.ExecuteMacro(Command, False);
end;

function TDDEClient.Poke(const ItemName, Value: string): boolean;
begin
  Result := False;
  if not FConnected then Exit;
  
  Result := FDDEClient.PokeData(ItemName, Value);
end;

begin
  WriteLn('=== DDE Demo (Windows) ===');
  WriteLn('');
  WriteLn('DDE สามารถใช้สื่อสารกับ:');
  WriteLn('- Microsoft Excel');
  WriteLn('- Microsoft Word');
  WriteLn('- Other DDE-capable applications');
  WriteLn('');
  WriteLn('Example: เชื่อมต่อกับ Excel');
  WriteLn('  Client.Connect(''Excel'', ''Sheet1'')');
  WriteLn('  Data := Client.Request(''R1C1'')  // อ่านค่า Cell A1');
  WriteLn('  Client.Poke(''R1C1'', ''Hello'')   // เขียนค่า Cell A1');
  WriteLn('');
  WriteLn('(ต้องมี Excel เปิดอยู่และมี DDE enabled)');
end.

{$ELSE}
begin
  WriteLn('DDE is Windows-only. On Linux/macOS, use:');
  WriteLn('- DBUS for desktop integration');
  WriteLn('- Sockets for application communication');
  WriteLn('- Shared memory for performance-critical IPC');
end.
{$ENDIF}
```

---

## 44.7 COM/OLE Automation

### COM Automation (Windows)

```pascal
program COMAutomation;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Windows, SysUtils, ComObj, Variants, Classes;

// ตัวอย่าง: ควบคุม Microsoft Word ด้วย COM Automation
procedure AutomateWord;
var
  WordApp, Documents, Doc, Selection: OLEVariant;
  TempFile: string;
begin
  WriteLn('Starting Microsoft Word automation...');
  
  try
    // สร้าง Word instance
    WordApp := CreateOleObject('Word.Application');
    
    // ซ่อน Word window
    WordApp.Visible := True; // แสดง Window
    
    // สร้าง document ใหม่
    Documents := WordApp.Documents;
    Doc := Documents.Add;
    
    // เขียนข้อความ
    Selection := WordApp.Selection;
    Selection.TypeText('สวัสดีครับ!' + #13#10);
    Selection.TypeText('นี่คือตัวอย่างการใช้ COM Automation กับ Microsoft Word' + #13#10);
    Selection.TypeText('เขียนโดย: Lazarus Pascal' + #13#10);
    
    // เพิ่ม heading
    Selection.Style := 'Heading 1';
    Selection.TypeText('หัวข้อหลัก');
    Selection.TypeParagraph;
    
    // บันทึกไฟล์
    TempFile := GetTempDir + 'TestDoc.docx';
    Doc.SaveAs2(TempFile, 16); // 16 = wdFormatXMLDocument
    WriteLn('Document saved: ', TempFile);
    
    // ปิด Word
    Doc.Close;
    WordApp.Quit;
    
    WriteLn('Word automation completed!');
    
    // ลบไฟล์ทดสอบ
    if FileExists(TempFile) then
      DeleteFile(TempFile);
    
  except
    on E: EOleSysError do
      WriteLn('COM Error: ', E.Message)
    on E: Exception do
      WriteLn('Error: ', E.Message);
  end;
end;

// ตัวอย่าง: ควบคุม Excel
procedure AutomateExcel;
var
  ExcelApp, Workbooks, Workbook, Sheets, Sheet, Cell: OLEVariant;
  i, j: integer;
  TempFile: string;
begin
  WriteLn('Starting Microsoft Excel automation...');
  
  try
    ExcelApp := CreateOleObject('Excel.Application');
    ExcelApp.Visible := False;
    ExcelApp.DisplayAlerts := False;
    
    // สร้าง workbook ใหม่
    Workbooks := ExcelApp.Workbooks;
    Workbook := Workbooks.Add;
    
    Sheets := Workbook.Sheets;
    Sheet := Sheets.Item[1];
    Sheet.Name := 'รายงาน';
    
    // เขียน headers
    Sheet.Cells.Item[1, 1].Value := 'ลำดับ';
    Sheet.Cells.Item[1, 2].Value := 'ชื่อ';
    Sheet.Cells.Item[1, 3].Value := 'ยอดขาย';
    Sheet.Cells.Item[1, 4].Value := 'หมายเหตุ';
    
    // เขียนข้อมูล
    var Names: array[1..5] of string = (
      'สมชาย', 'สมหญิง', 'วิชัย', 'นิภา', 'ประยุทธ์'
    );
    var Sales: array[1..5] of integer = (45000, 62000, 38000, 75000, 51000);
    
    for i := 1 to 5 do
    begin
      Sheet.Cells.Item[i + 1, 1].Value := i;
      Sheet.Cells.Item[i + 1, 2].Value := Names[i];
      Sheet.Cells.Item[i + 1, 3].Value := Sales[i];
      if Sales[i] > 60000 then
        Sheet.Cells.Item[i + 1, 4].Value := 'ดีมาก'
      else if Sales[i] > 45000 then
        Sheet.Cells.Item[i + 1, 4].Value := 'ดี'
      else
        Sheet.Cells.Item[i + 1, 4].Value := 'ปรับปรุง';
    end;
    
    // เพิ่ม SUM formula
    Sheet.Cells.Item[7, 2].Value := 'รวม';
    Sheet.Cells.Item[7, 3].Formula := '=SUM(C2:C6)';
    
    // บันทึก
    TempFile := GetTempDir + 'Report.xlsx';
    Workbook.SaveAs(TempFile, 51); // 51 = xlOpenXMLWorkbook
    WriteLn('Excel file saved: ', TempFile);
    
    Workbook.Close;
    ExcelApp.Quit;
    
    WriteLn('Excel automation completed!');
    
    if FileExists(TempFile) then
      DeleteFile(TempFile);
      
  except
    on E: EOleSysError do
      WriteLn('COM Error: ', E.Message);
    on E: Exception do
      WriteLn('Error: ', E.Message);
  end;
end;

begin
  WriteLn('=== COM/OLE Automation Demo (Windows) ===');
  WriteLn('');
  WriteLn('1. Word Automation');
  AutomateWord;
  WriteLn('');
  WriteLn('2. Excel Automation');
  AutomateExcel;
end.

{$ELSE}
begin
  WriteLn('COM/OLE Automation is Windows-only.');
  WriteLn('For cross-platform document generation, consider:');
  WriteLn('  - FPSpreadsheet for Excel files');
  WriteLn('  - DocX library for Word documents');
  WriteLn('  - PDF libraries for PDF output');
end.
{$ENDIF}
```

---

## 44.8 Complete Example: Process Monitor

```pascal
program ProcessMonitor;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Process
  {$IFDEF WINDOWS}, Windows, TlHelp32{$ENDIF}
  {$IFDEF UNIX}, BaseUnix{$ENDIF};

type
  TProcessInfo = record
    PID: DWORD;
    Name: string;
    Status: string;
    CPUUsage: double;
    MemoryMB: double;
    ParentPID: DWORD;
  end;
  
  TProcessList = array of TProcessInfo;

  TProcessMonitor = class
  private
    FInterval: integer;
    FRunning: boolean;
    FProcesses: TProcessList;
    FLock: TCriticalSection;
    
    {$IFDEF WINDOWS}
    procedure CollectWindowsProcesses;
    {$ENDIF}
    {$IFDEF UNIX}
    procedure CollectUnixProcesses;
    {$ENDIF}
    
  public
    constructor Create(IntervalMs: integer = 2000);
    destructor Destroy; override;
    
    procedure Start;
    procedure Stop;
    procedure Refresh;
    procedure Display;
    
    function FindByName(const Name: string): TProcessInfo;
    function FindByPID(PID: DWORD): TProcessInfo;
    function GetCount: integer;
  end;

constructor TProcessMonitor.Create(IntervalMs: integer);
begin
  inherited Create;
  FInterval := IntervalMs;
  FRunning := False;
  SetLength(FProcesses, 0);
  FLock := TCriticalSection.Create;
end;

destructor TProcessMonitor.Destroy;
begin
  Stop;
  FLock.Free;
  inherited Destroy;
end;

{$IFDEF WINDOWS}
procedure TProcessMonitor.CollectWindowsProcesses;
var
  SnapShot: HANDLE;
  Entry: TProcessEntry32;
  NewList: TProcessList;
  Idx: integer;
begin
  SetLength(NewList, 0);
  Idx := 0;
  
  SnapShot := CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
  if SnapShot = INVALID_HANDLE_VALUE then Exit;
  
  try
    Entry.dwSize := SizeOf(TProcessEntry32);
    
    if Process32First(SnapShot, Entry) then
    begin
      repeat
        SetLength(NewList, Idx + 1);
        NewList[Idx].PID := Entry.th32ProcessID;
        NewList[Idx].Name := string(Entry.szExeFile);
        NewList[Idx].ParentPID := Entry.th32ParentProcessID;
        NewList[Idx].Status := 'Running';
        NewList[Idx].CPUUsage := 0;
        NewList[Idx].MemoryMB := 0;
        Inc(Idx);
      until not Process32Next(SnapShot, Entry);
    end;
  finally
    CloseHandle(SnapShot);
  end;
  
  FLock.Acquire;
  try
    FProcesses := NewList;
  finally
    FLock.Release;
  end;
end;
{$ENDIF}

{$IFDEF UNIX}
procedure TProcessMonitor.CollectUnixProcesses;
var
  PS: TProcess;
  Output: TStringList;
  Line: string;
  Parts: TStringList;
  NewList: TProcessList;
  Idx: integer;
begin
  SetLength(NewList, 0);
  Idx := 0;
  
  PS := TProcess.Create(nil);
  Output := TStringList.Create;
  Parts := TStringList.Create;
  
  try
    PS.Executable := 'ps';
    PS.Parameters.Add('-e');
    PS.Parameters.Add('-o');
    PS.Parameters.Add('pid,ppid,comm,stat,pcpu,rss');
    PS.Parameters.Add('--no-header');
    PS.Options := PS.Options + [poUsePipes, poWaitOnExit];
    
    try
      PS.Execute;
      
      // อ่าน output
      var Buffer: string := '';
      var Bytes: array[0..4095] of byte;
      var BytesRead: integer;
      
      repeat
        BytesRead := PS.Output.Read(Bytes, SizeOf(Bytes));
        if BytesRead > 0 then
        begin
          SetLength(Buffer, Length(Buffer) + BytesRead);
          Move(Bytes[0], Buffer[Length(Buffer) - BytesRead + 1], BytesRead);
        end;
      until BytesRead = 0;
      
      Output.Text := Buffer;
      
      for Line in Output do
      begin
        var TrimLine := Trim(Line);
        if TrimLine = '' then Continue;
        
        // Parse ps output
        Parts.DelimitedText := TrimLine;
        
        if Parts.Count >= 6 then
        begin
          SetLength(NewList, Idx + 1);
          try
            NewList[Idx].PID := StrToIntDef(Trim(Parts[0]), 0);
            NewList[Idx].ParentPID := StrToIntDef(Trim(Parts[1]), 0);
            NewList[Idx].Name := Trim(Parts[2]);
            NewList[Idx].Status := Trim(Parts[3]);
            NewList[Idx].CPUUsage := StrToFloatDef(Trim(Parts[4]), 0);
            NewList[Idx].MemoryMB := StrToIntDef(Trim(Parts[5]), 0) / 1024; // KB to MB
            Inc(Idx);
          except
            // ignore parse errors
          end;
        end;
      end;
    except
      on E: Exception do
        WriteLn('PS error: ', E.Message);
    end;
    
  finally
    PS.Free;
    Output.Free;
    Parts.Free;
  end;
  
  FLock.Acquire;
  try
    FProcesses := NewList;
  finally
    FLock.Release;
  end;
end;
{$ENDIF}

procedure TProcessMonitor.Refresh;
begin
  {$IFDEF WINDOWS}
  CollectWindowsProcesses;
  {$ENDIF}
  {$IFDEF UNIX}
  CollectUnixProcesses;
  {$ENDIF}
end;

procedure TProcessMonitor.Start;
begin
  FRunning := True;
  Refresh;
end;

procedure TProcessMonitor.Stop;
begin
  FRunning := False;
end;

procedure TProcessMonitor.Display;
var
  i: integer;
  Procs: TProcessList;
begin
  FLock.Acquire;
  try
    Procs := FProcesses;
  finally
    FLock.Release;
  end;
  
  WriteLn(Format('%-8s %-8s %-25s %-10s %8s %8s', [
    'PID', 'PPID', 'Name', 'Status', 'CPU%', 'Mem(MB)'
  ]));
  WriteLn(StringOfChar('-', 80));
  
  for i := 0 to Min(19, Length(Procs) - 1) do // แสดง 20 อันแรก
  begin
    WriteLn(Format('%-8d %-8d %-25s %-10s %8.1f %8.1f', [
      Procs[i].PID,
      Procs[i].ParentPID,
      Copy(Procs[i].Name, 1, 25),
      Copy(Procs[i].Status, 1, 10),
      Procs[i].CPUUsage,
      Procs[i].MemoryMB
    ]));
  end;
  
  WriteLn('');
  WriteLn('Total processes: ', Length(Procs));
end;

function TProcessMonitor.FindByName(const Name: string): TProcessInfo;
var
  i: integer;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  FLock.Acquire;
  try
    for i := 0 to High(FProcesses) do
      if SameText(FProcesses[i].Name, Name) then
      begin
        Result := FProcesses[i];
        Break;
      end;
  finally
    FLock.Release;
  end;
end;

function TProcessMonitor.FindByPID(PID: DWORD): TProcessInfo;
var
  i: integer;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  FLock.Acquire;
  try
    for i := 0 to High(FProcesses) do
      if FProcesses[i].PID = PID then
      begin
        Result := FProcesses[i];
        Break;
      end;
  finally
    FLock.Release;
  end;
end;

function TProcessMonitor.GetCount: integer;
begin
  FLock.Acquire;
  try
    Result := Length(FProcesses);
  finally
    FLock.Release;
  end;
end;

var
  Monitor: TProcessMonitor;
  Found: TProcessInfo;
begin
  WriteLn('=== Process Monitor Demo ===');
  WriteLn('');
  
  Monitor := TProcessMonitor.Create(2000);
  try
    WriteLn('Collecting process list...');
    Monitor.Start;
    WriteLn('');
    
    Monitor.Display;
    
    WriteLn('');
    {$IFDEF UNIX}
    // ค้นหา process ที่รู้จัก
    Found := Monitor.FindByName('bash');
    if Found.PID > 0 then
      WriteLn('Found bash at PID: ', Found.PID)
    else
      WriteLn('bash not found in process list');
    {$ENDIF}
    {$IFDEF WINDOWS}
    Found := Monitor.FindByName('explorer.exe');
    if Found.PID > 0 then
      WriteLn('Found Explorer at PID: ', Found.PID)
    else
      WriteLn('Explorer not found');
    {$ENDIF}
    
  finally
    Monitor.Free;
  end;
end.
```

---

## แบบฝึกหัด 5 ข้อ

### ข้อ 1: Chat IPC
สร้างระบบ chat แบบ local IPC:
- ใช้ named pipes สำหรับ communication
- รองรับ broadcast messages
- หลาย clients พร้อมกัน

### ข้อ 2: File Watcher IPC
สร้าง file watcher service:
- Monitor directory changes
- แจ้ง subscribed processes เมื่อมีการเปลี่ยนแปลง
- ใช้ shared memory สำหรับ state

### ข้อ 3: Command Executor
สร้าง command executor service:
- รับ commands ผ่าน message queue
- Execute และ return results
- Handle timeouts

### ข้อ 4: Distributed Counter
สร้าง distributed counter:
- หลาย processes increment counter พร้อมกัน
- ใช้ shared memory + semaphore
- ไม่เกิด race condition

### ข้อ 5: Process Communication Framework
สร้าง mini IPC framework:
- Abstract interface สำหรับ IPC
- Implementations: Pipes, Shared Memory
- Message serialization/deserialization
