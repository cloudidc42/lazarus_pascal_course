# ตอนที่ 62: Scripting Integration ใน Lazarus/Pascal

## บทนำ

การรวม Scripting Engine เข้ากับแอปพลิเคชัน Pascal ช่วยให้ผู้ใช้สามารถขยายฟังก์ชันโปรแกรมได้โดยไม่ต้องคอมไพล์ใหม่ บทนี้จะครอบคลุม Pascal Script, Python embedding, และ Lua scripting

---

## 62.1 Pascal Script (PascalScript library)

PascalScript เป็น scripting engine ที่ใช้ภาษา Pascal เป็นภาษา script ทำให้ผู้ใช้ที่คุ้นเคยกับ Pascal สามารถเขียน script ได้ทันที

### ติดตั้ง PascalScript

```bash
# ติดตั้งผ่าน OPM (Online Package Manager) ใน Lazarus
# หรือโหลดจาก: https://github.com/remobjects/pascalscript
# เพิ่มใน .lpr:
# uses PascalScript;
```

### ตัวอย่างพื้นฐาน

```pascal
program pascal_script_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  uPSCompiler,   // PascalScript Compiler
  uPSRuntime,    // PascalScript Runtime
  uPSC_std,      // Standard functions
  uPSR_std;      // Runtime std

// Script ที่ต้องการรัน
const
  SCRIPT_CODE = '''
    var i: Integer;
    var s: string;
    begin
      s := ''สวัสดี Pascal Script!'';
      WriteLn(s);
      for i := 1 to 5 do
        WriteLn(''จำนวน: '' + IntToStr(i));
    end.
  ''';

procedure RunScript(const AScript: string);
var
  Compiler: TPSPascalCompiler;
  Runtime: TPSExec;
  Data: string;
  Msg: TPSPascalCompilerMessage;
  i: Integer;
begin
  Compiler := TPSPascalCompiler.Create;
  try
    // ลงทะเบียน built-in functions
    RegisterStdLibWithInit(Compiler);
    
    // คอมไพล์ script
    if not Compiler.Compile(AScript) then
    begin
      WriteLn('ข้อผิดพลาดในการคอมไพล์:');
      for i := 0 to Compiler.MsgCount - 1 do
      begin
        Msg := Compiler.Msg[i];
        WriteLn(Format('  บรรทัด %d, คอลัมน์ %d: %s', 
          [Msg.Row, Msg.Col, Msg.ShortMessageToString]));
      end;
      Exit;
    end;
    
    // ดึง bytecode
    Compiler.GetOutput(Data);
    
    // สร้าง Runtime
    Runtime := TPSExec.Create;
    try
      RegisterStdLibRuntime(Runtime);
      
      if not Runtime.LoadData(Data) then
      begin
        WriteLn('ไม่สามารถโหลด bytecode: ', Runtime.ExceptionString);
        Exit;
      end;
      
      // รัน script
      if not Runtime.RunScript then
        WriteLn('ข้อผิดพลาดขณะรัน: ', Runtime.ExceptionString);
        
    finally
      Runtime.Free;
    end;
    
  finally
    Compiler.Free;
  end;
end;

begin
  WriteLn('=== Pascal Script Demo ===');
  RunScript(SCRIPT_CODE);
end.
```

---

## 62.2 Scripting Engine แบบ Embedded

### สร้าง Script Engine ที่รองรับหลายภาษา

```pascal
unit script_engine;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TScriptLanguage = (slPascal, slPython, slLua, slJavaScript);
  
  TScriptResult = record
    Success: Boolean;
    Output: string;
    ErrorMsg: string;
    ExecutionTime: Int64;  // milliseconds
  end;
  
  // Callback สำหรับฟังก์ชันที่ Script เรียกได้
  TScriptFunction = function(const AParams: array of Variant): Variant of object;
  
  TScriptEngine = class
  private
    FLanguage: TScriptLanguage;
    FFunctions: TStringList;  // ชื่อ -> callback
    FVariables: TStringList;  // ตัวแปร global
    FOutput: TStringList;
    FMaxExecutionTime: Integer;  // ms, 0 = ไม่จำกัด
    
  public
    constructor Create(ALanguage: TScriptLanguage);
    destructor Destroy; override;
    
    // ลงทะเบียนฟังก์ชัน
    procedure RegisterFunction(const AName: string; ACallback: TScriptFunction);
    
    // ตัวแปร global
    procedure SetVariable(const AName: string; AValue: Variant);
    function GetVariable(const AName: string): Variant;
    
    // รัน script
    function Execute(const AScript: string): TScriptResult;
    function ExecuteFile(const AFilePath: string): TScriptResult;
    
    property Language: TScriptLanguage read FLanguage;
    property Output: TStringList read FOutput;
    property MaxExecutionTime: Integer read FMaxExecutionTime write FMaxExecutionTime;
  end;

implementation

constructor TScriptEngine.Create(ALanguage: TScriptLanguage);
begin
  inherited Create;
  FLanguage := ALanguage;
  FFunctions := TStringList.Create;
  FVariables := TStringList.Create;
  FOutput := TStringList.Create;
  FMaxExecutionTime := 30000;  // 30 วินาที default
end;

destructor TScriptEngine.Destroy;
begin
  FFunctions.Free;
  FVariables.Free;
  FOutput.Free;
  inherited Destroy;
end;

procedure TScriptEngine.RegisterFunction(const AName: string; ACallback: TScriptFunction);
begin
  // บันทึก callback ใน list
  // ในระบบจริงต้องใช้ pointer หรือ method pointer
  FFunctions.Values[AName] := AName;  // simplified
end;

procedure TScriptEngine.SetVariable(const AName: string; AValue: Variant);
begin
  FVariables.Values[AName] := VarToStr(AValue);
end;

function TScriptEngine.GetVariable(const AName: string): Variant;
begin
  Result := FVariables.Values[AName];
end;

function TScriptEngine.Execute(const AScript: string): TScriptResult;
var
  StartTime: Int64;
begin
  Result.Success := False;
  Result.Output := '';
  Result.ErrorMsg := '';
  
  StartTime := GetTickCount64;
  FOutput.Clear;
  
  try
    case FLanguage of
      slPascal:
        begin
          // รัน Pascal Script
          Result.Output := 'Pascal Script execution (see unit uPSExec)';
          Result.Success := True;
        end;
      slPython:
        begin
          Result.Output := 'Python execution (see section 62.3)';
          Result.Success := True;
        end;
      slLua:
        begin
          Result.Output := 'Lua execution (see section 62.4)';
          Result.Success := True;
        end;
    end;
  except
    on E: Exception do
    begin
      Result.Success := False;
      Result.ErrorMsg := E.Message;
    end;
  end;
  
  Result.ExecutionTime := GetTickCount64 - StartTime;
  Result.Output := FOutput.Text;
end;

function TScriptEngine.ExecuteFile(const AFilePath: string): TScriptResult;
var
  Script: TStringList;
begin
  Script := TStringList.Create;
  try
    Script.LoadFromFile(AFilePath);
    Result := Execute(Script.Text);
  finally
    Script.Free;
  end;
end;

end.
```

---

## 62.3 Python Embedding

### ใช้ Python4Delphi/Python4Lazarus

```pascal
unit python_bridge;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  PythonEngine,  // จาก Python4Delphi package
  VarPyth;

type
  TPythonBridge = class
  private
    FEngine: TPythonEngine;
    FModule: TPythonModule;
    FOutput: TStringList;
    
    procedure OnOutput(Sender: TObject; const Text: String);
    procedure OnError(Sender: TObject; const Text: String);
    
    // ฟังก์ชัน Pascal ที่ให้ Python เรียก
    function PyFunc_GetVersion(PSelf, Args: PPyObject): PPyObject; cdecl;
    function PyFunc_ShowMessage(PSelf, Args: PPyObject): PPyObject; cdecl;
    function PyFunc_Calculate(PSelf, Args: PPyObject): PPyObject; cdecl;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    function Initialize: Boolean;
    procedure Finalize;
    
    function RunScript(const AScript: string): Boolean;
    function RunFile(const AFilePath: string): Boolean;
    function EvalExpression(const AExpr: string): string;
    
    procedure SetVar(const AName: string; AValue: Variant);
    function GetVar(const AName: string): Variant;
    
    property Output: TStringList read FOutput;
  end;

implementation

constructor TPythonBridge.Create;
begin
  inherited Create;
  FOutput := TStringList.Create;
end;

destructor TPythonBridge.Destroy;
begin
  Finalize;
  FOutput.Free;
  inherited Destroy;
end;

function TPythonBridge.Initialize: Boolean;
begin
  Result := False;
  try
    FEngine := TPythonEngine.Create(nil);
    FEngine.AutoLoad := True;
    FEngine.IO := nil;
    
    // ลงทะเบียน module สำหรับ Pascal functions
    FModule := TPythonModule.Create(nil);
    FModule.Engine := FEngine;
    FModule.ModuleName := 'pascal';
    
    // เพิ่มฟังก์ชัน Pascal
    FModule.AddMethod('get_version', @PyFunc_GetVersion, 
      'get_version() -> str: คืนค่า version ของโปรแกรม');
    FModule.AddMethod('show_message', @PyFunc_ShowMessage,
      'show_message(msg: str) -> None: แสดงข้อความ');
    FModule.AddMethod('calculate', @PyFunc_Calculate,
      'calculate(expr: str) -> float: คำนวณนิพจน์');
      
    FEngine.LoadDll;
    Result := True;
    
  except
    on E: Exception do
    begin
      FOutput.Add('Error initializing Python: ' + E.Message);
      Result := False;
    end;
  end;
end;

procedure TPythonBridge.Finalize;
begin
  if Assigned(FModule) then FreeAndNil(FModule);
  if Assigned(FEngine) then FreeAndNil(FEngine);
end;

procedure TPythonBridge.OnOutput(Sender: TObject; const Text: String);
begin
  FOutput.Add(Text);
end;

procedure TPythonBridge.OnError(Sender: TObject; const Text: String);
begin
  FOutput.Add('Error: ' + Text);
end;

function TPythonBridge.PyFunc_GetVersion(PSelf, Args: PPyObject): PPyObject; cdecl;
begin
  Result := FEngine.PyUnicodeFromString('MyApp 1.0.0');
end;

function TPythonBridge.PyFunc_ShowMessage(PSelf, Args: PPyObject): PPyObject; cdecl;
var
  Msg: PAnsiChar;
begin
  if FEngine.PyArg_ParseTuple(Args, 'z', [@Msg]) <> 0 then
  begin
    FOutput.Add('MESSAGE: ' + String(Msg));
  end;
  Result := FEngine.ReturnNone;
end;

function TPythonBridge.PyFunc_Calculate(PSelf, Args: PPyObject): PPyObject; cdecl;
var
  Expr: PAnsiChar;
  Val: Double;
begin
  Result := nil;
  if FEngine.PyArg_ParseTuple(Args, 'z', [@Expr]) <> 0 then
  begin
    try
      Val := StrToFloat(String(Expr));  // simplified
      Result := FEngine.PyFloat_FromDouble(Val);
    except
      Result := FEngine.PyFloat_FromDouble(0);
    end;
  end;
  if Result = nil then
    Result := FEngine.PyFloat_FromDouble(0);
end;

function TPythonBridge.RunScript(const AScript: string): Boolean;
begin
  Result := False;
  if not Assigned(FEngine) then Exit;
  
  FOutput.Clear;
  try
    FEngine.ExecString(AnsiString(AScript));
    Result := True;
  except
    on E: EPyException do
    begin
      FOutput.Add('Python Error: ' + E.Message);
      Result := False;
    end;
  end;
end;

function TPythonBridge.RunFile(const AFilePath: string): Boolean;
begin
  Result := False;
  if not Assigned(FEngine) then Exit;
  
  if not FileExists(AFilePath) then
  begin
    FOutput.Add('ไม่พบไฟล์: ' + AFilePath);
    Exit;
  end;
  
  FOutput.Clear;
  try
    FEngine.ExecFile(AnsiString(AFilePath));
    Result := True;
  except
    on E: EPyException do
    begin
      FOutput.Add('Python Error: ' + E.Message);
      Result := False;
    end;
  end;
end;

function TPythonBridge.EvalExpression(const AExpr: string): string;
var
  Obj: PPyObject;
begin
  Result := '';
  if not Assigned(FEngine) then Exit;
  
  try
    Obj := FEngine.EvalString(AnsiString(AExpr));
    if Assigned(Obj) then
    begin
      Result := FEngine.PyObjectAsString(Obj);
      FEngine.Py_DECREF(Obj);
    end;
  except
    on E: EPyException do
      Result := 'Error: ' + E.Message;
  end;
end;

procedure TPythonBridge.SetVar(const AName: string; AValue: Variant);
begin
  if not Assigned(FEngine) then Exit;
  
  case VarType(AValue) of
    varInteger, varInt64: 
      FEngine.ExecString(AnsiString(AName + ' = ' + IntToStr(AValue)));
    varDouble, varSingle:
      FEngine.ExecString(AnsiString(AName + ' = ' + FloatToStr(AValue)));
    varBoolean:
      begin
        if AValue then
          FEngine.ExecString(AnsiString(AName + ' = True'))
        else
          FEngine.ExecString(AnsiString(AName + ' = False'));
      end;
    else
      FEngine.ExecString(AnsiString(AName + ' = "' + VarToStr(AValue) + '"'));
  end;
end;

function TPythonBridge.GetVar(const AName: string): Variant;
var
  S: string;
begin
  S := EvalExpression(AName);
  Result := S;
end;

end.
```

### ตัวอย่างการใช้ Python Bridge

```pascal
procedure DemoPythonBridge;
var
  Python: TPythonBridge;
  Script: string;
begin
  Python := TPythonBridge.Create;
  try
    if not Python.Initialize then
    begin
      WriteLn('ไม่สามารถเริ่ม Python');
      Exit;
    end;
    
    // ส่งตัวแปรจาก Pascal ไป Python
    Python.SetVar('app_name', 'MyPascalApp');
    Python.SetVar('user_count', 100);
    Python.SetVar('price', 199.99);
    
    // รัน Python script
    Script :=
      'import pascal' + LineEnding +
      'import json' + LineEnding +
      '' + LineEnding +
      '# ใช้ตัวแปรจาก Pascal' + LineEnding +
      'print(f"แอปพลิเคชัน: {app_name}")' + LineEnding +
      'print(f"จำนวนผู้ใช้: {user_count}")' + LineEnding +
      'print(f"ราคา: {price:.2f} บาท")' + LineEnding +
      '' + LineEnding +
      '# เรียกฟังก์ชัน Pascal' + LineEnding +
      'version = pascal.get_version()' + LineEnding +
      'print(f"Version: {version}")' + LineEnding +
      '' + LineEnding +
      '# คำนวณ' + LineEnding +
      'data = [10, 20, 30, 40, 50]' + LineEnding +
      'avg = sum(data) / len(data)' + LineEnding +
      'print(f"ค่าเฉลี่ย: {avg}")' + LineEnding +
      '' + LineEnding +
      '# ส่งผลลัพธ์กลับ' + LineEnding +
      'result = json.dumps({"avg": avg, "count": len(data)})' + LineEnding +
      'pascal.show_message(result)';
      
    if Python.RunScript(Script) then
    begin
      WriteLn('=== Output ===');
      WriteLn(Python.Output.Text);
    end;
    
    // ดึงค่าตัวแปรจาก Python
    WriteLn('version from Python: ', Python.GetVar('version'));
    
  finally
    Python.Free;
  end;
end;
```

---

## 62.4 Lua Scripting

### ใช้ LuaBridge หรือ lua4pascal

```pascal
unit lua_engine;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  lua,    // Lua binding headers
  lauxlib,
  lualib;

type
  TLuaEngine = class
  private
    FState: Plua_State;
    FOutput: TStringList;
    
    // ฟังก์ชัน Pascal ที่ให้ Lua เรียก
    class function lua_print(L: Plua_State): Integer; cdecl; static;
    class function lua_getdata(L: Plua_State): Integer; cdecl; static;
    class function lua_setdata(L: Plua_State): Integer; cdecl; static;
    class function lua_calculate(L: Plua_State): Integer; cdecl; static;
    
    // ตัวแปร instance สำหรับ callback
    class var CurrentEngine: TLuaEngine;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    function Initialize: Boolean;
    procedure Finalize;
    
    // รัน script
    function RunString(const AScript: string): Boolean;
    function RunFile(const AFilePath: string): Boolean;
    
    // ลงทะเบียนฟังก์ชัน
    procedure RegisterFunction(const AName: string; AFunc: lua_CFunction);
    
    // ตัวแปร global
    procedure SetGlobal(const AName: string; AValue: Variant);
    function GetGlobal(const AName: string): Variant;
    
    // เรียกฟังก์ชัน Lua
    function CallFunction(const AFuncName: string; const AArgs: array of Variant): Variant;
    
    property Output: TStringList read FOutput;
  end;

implementation

{ Callback functions - must be class static for C cdecl }

class function TLuaEngine.lua_print(L: Plua_State): Integer; cdecl;
var
  i, n: Integer;
  S: string;
begin
  n := lua_gettop(L);
  S := '';
  
  for i := 1 to n do
  begin
    if i > 1 then S := S + #9;
    S := S + String(lua_tostring(L, i));
  end;
  
  if Assigned(CurrentEngine) then
    CurrentEngine.FOutput.Add(S);
    
  Result := 0;
end;

class function TLuaEngine.lua_getdata(L: Plua_State): Integer; cdecl;
var
  Key: string;
begin
  Key := String(lua_tostring(L, 1));
  
  // ส่งข้อมูลกลับไปยัง Lua
  // (ตัวอย่างง่ายๆ)
  if Key = 'date' then
    lua_pushstring(L, PAnsiChar(AnsiString(FormatDateTime('yyyy-mm-dd', Now))))
  else if Key = 'time' then
    lua_pushstring(L, PAnsiChar(AnsiString(FormatDateTime('hh:nn:ss', Now))))
  else
    lua_pushnil(L);
    
  Result := 1;
end;

class function TLuaEngine.lua_setdata(L: Plua_State): Integer; cdecl;
var
  Key, Value: string;
begin
  Key := String(lua_tostring(L, 1));
  Value := String(lua_tostring(L, 2));
  
  // บันทึกข้อมูล
  if Assigned(CurrentEngine) then
    CurrentEngine.FOutput.Add('SET: ' + Key + '=' + Value);
    
  Result := 0;
end;

class function TLuaEngine.lua_calculate(L: Plua_State): Integer; cdecl;
var
  A, B: Double;
  Op: string;
  Result2: Double;
begin
  A := lua_tonumber(L, 1);
  Op := String(lua_tostring(L, 2));
  B := lua_tonumber(L, 3);
  
  if Op = '+' then Result2 := A + B
  else if Op = '-' then Result2 := A - B
  else if Op = '*' then Result2 := A * B
  else if (Op = '/') and (B <> 0) then Result2 := A / B
  else Result2 := 0;
  
  lua_pushnumber(L, Result2);
  Result := 1;
end;

{ TLuaEngine }

constructor TLuaEngine.Create;
begin
  inherited Create;
  FOutput := TStringList.Create;
  FState := nil;
  CurrentEngine := Self;
end;

destructor TLuaEngine.Destroy;
begin
  Finalize;
  FOutput.Free;
  inherited Destroy;
end;

function TLuaEngine.Initialize: Boolean;
begin
  Result := False;
  
  try
    // สร้าง Lua state
    FState := luaL_newstate();
    
    if not Assigned(FState) then
    begin
      FOutput.Add('ไม่สามารถสร้าง Lua state');
      Exit;
    end;
    
    // โหลด standard libraries
    luaL_openlibs(FState);
    
    // ลงทะเบียนฟังก์ชัน Pascal
    lua_register(FState, 'print', @lua_print);
    lua_register(FState, 'getdata', @lua_getdata);
    lua_register(FState, 'setdata', @lua_setdata);
    lua_register(FState, 'calculate', @lua_calculate);
    
    Result := True;
    FOutput.Add('Lua engine initialized');
    
  except
    on E: Exception do
    begin
      FOutput.Add('Error: ' + E.Message);
      Result := False;
    end;
  end;
end;

procedure TLuaEngine.Finalize;
begin
  if Assigned(FState) then
  begin
    lua_close(FState);
    FState := nil;
  end;
end;

function TLuaEngine.RunString(const AScript: string): Boolean;
var
  ErrCode: Integer;
  ErrMsg: string;
begin
  Result := False;
  if not Assigned(FState) then Exit;
  
  FOutput.Clear;
  CurrentEngine := Self;
  
  ErrCode := luaL_dostring(FState, PAnsiChar(AnsiString(AScript)));
  
  if ErrCode <> 0 then
  begin
    ErrMsg := String(lua_tostring(FState, -1));
    lua_pop(FState, 1);
    FOutput.Add('Lua Error: ' + ErrMsg);
    Exit;
  end;
  
  Result := True;
end;

function TLuaEngine.RunFile(const AFilePath: string): Boolean;
var
  ErrCode: Integer;
  ErrMsg: string;
begin
  Result := False;
  if not Assigned(FState) then Exit;
  
  if not FileExists(AFilePath) then
  begin
    FOutput.Add('ไม่พบไฟล์: ' + AFilePath);
    Exit;
  end;
  
  ErrCode := luaL_dofile(FState, PAnsiChar(AnsiString(AFilePath)));
  
  if ErrCode <> 0 then
  begin
    ErrMsg := String(lua_tostring(FState, -1));
    lua_pop(FState, 1);
    FOutput.Add('Lua Error: ' + ErrMsg);
    Exit;
  end;
  
  Result := True;
end;

procedure TLuaEngine.RegisterFunction(const AName: string; AFunc: lua_CFunction);
begin
  if Assigned(FState) then
    lua_register(FState, PAnsiChar(AnsiString(AName)), AFunc);
end;

procedure TLuaEngine.SetGlobal(const AName: string; AValue: Variant);
begin
  if not Assigned(FState) then Exit;
  
  case VarType(AValue) of
    varInteger, varInt64:
      begin
        lua_pushinteger(FState, lua_Integer(Integer(AValue)));
        lua_setglobal(FState, PAnsiChar(AnsiString(AName)));
      end;
    varDouble, varSingle:
      begin
        lua_pushnumber(FState, lua_Number(Double(AValue)));
        lua_setglobal(FState, PAnsiChar(AnsiString(AName)));
      end;
    varBoolean:
      begin
        lua_pushboolean(FState, Ord(Boolean(AValue)));
        lua_setglobal(FState, PAnsiChar(AnsiString(AName)));
      end;
    else
      begin
        lua_pushstring(FState, PAnsiChar(AnsiString(VarToStr(AValue))));
        lua_setglobal(FState, PAnsiChar(AnsiString(AName)));
      end;
  end;
end;

function TLuaEngine.GetGlobal(const AName: string): Variant;
var
  LuaType: Integer;
begin
  Result := Null;
  if not Assigned(FState) then Exit;
  
  lua_getglobal(FState, PAnsiChar(AnsiString(AName)));
  LuaType := lua_type(FState, -1);
  
  case LuaType of
    LUA_TNUMBER:
      begin
        if lua_isinteger(FState, -1) <> 0 then
          Result := Integer(lua_tointeger(FState, -1))
        else
          Result := lua_tonumber(FState, -1);
      end;
    LUA_TBOOLEAN:
      Result := Boolean(lua_toboolean(FState, -1) <> 0);
    LUA_TSTRING:
      Result := String(lua_tostring(FState, -1));
    else
      Result := Null;
  end;
  
  lua_pop(FState, 1);
end;

function TLuaEngine.CallFunction(const AFuncName: string; 
  const AArgs: array of Variant): Variant;
var
  i, ArgCount: Integer;
  LuaType: Integer;
begin
  Result := Null;
  if not Assigned(FState) then Exit;
  
  lua_getglobal(FState, PAnsiChar(AnsiString(AFuncName)));
  
  if lua_type(FState, -1) <> LUA_TFUNCTION then
  begin
    lua_pop(FState, 1);
    FOutput.Add('ไม่พบฟังก์ชัน: ' + AFuncName);
    Exit;
  end;
  
  ArgCount := Length(AArgs);
  
  for i := 0 to ArgCount - 1 do
  begin
    case VarType(AArgs[i]) of
      varInteger: lua_pushinteger(FState, AArgs[i]);
      varDouble:  lua_pushnumber(FState, AArgs[i]);
      varBoolean: lua_pushboolean(FState, Ord(Boolean(AArgs[i])));
      else        lua_pushstring(FState, PAnsiChar(AnsiString(VarToStr(AArgs[i]))));
    end;
  end;
  
  if lua_pcall(FState, ArgCount, 1, 0) <> 0 then
  begin
    FOutput.Add('Error calling ' + AFuncName + ': ' + String(lua_tostring(FState, -1)));
    lua_pop(FState, 1);
    Exit;
  end;
  
  LuaType := lua_type(FState, -1);
  case LuaType of
    LUA_TNUMBER: Result := lua_tonumber(FState, -1);
    LUA_TBOOLEAN: Result := Boolean(lua_toboolean(FState, -1) <> 0);
    LUA_TSTRING:  Result := String(lua_tostring(FState, -1));
    else Result := Null;
  end;
  
  lua_pop(FState, 1);
end;

end.
```

### Script Lua ตัวอย่าง

```lua
-- inventory_script.lua
-- Script สำหรับจัดการสินค้า

local inventory = {}
local low_stock_threshold = 10

-- ฟังก์ชันเพิ่มสินค้า
function add_product(id, name, price, qty)
    inventory[id] = {
        name = name,
        price = price,
        quantity = qty,
        sales = 0
    }
    print("เพิ่มสินค้า: " .. name .. " จำนวน " .. qty)
end

-- ฟังก์ชันขายสินค้า
function sell_product(id, qty)
    local product = inventory[id]
    if not product then
        print("ไม่พบสินค้า ID: " .. id)
        return false
    end
    
    if product.quantity < qty then
        print("สินค้าไม่เพียงพอ: " .. product.name)
        return false
    end
    
    product.quantity = product.quantity - qty
    product.sales = product.sales + qty
    
    local total = product.price * qty
    print(string.format("ขาย %s x%d = %.2f บาท", product.name, qty, total))
    
    -- แจ้งเตือนสินค้าใกล้หมด
    if product.quantity <= low_stock_threshold then
        print("⚠️ สินค้าเหลือน้อย: " .. product.name .. " เหลือ " .. product.quantity)
        setdata("low_stock_" .. id, tostring(product.quantity))
    end
    
    return true
end

-- ฟังก์ชันรายงาน
function generate_report()
    print("\n=== รายงานสินค้าคงคลัง ===")
    print(string.format("%-20s %8s %10s %10s", "ชื่อสินค้า", "คงเหลือ", "ราคา", "ยอดขาย"))
    print(string.rep("-", 52))
    
    local total_value = 0
    for id, p in pairs(inventory) do
        print(string.format("%-20s %8d %10.2f %10d", 
            p.name, p.quantity, p.price, p.sales))
        total_value = total_value + (p.quantity * p.price)
    end
    print(string.rep("-", 52))
    print(string.format("มูลค่าสต็อครวม: %.2f บาท", total_value))
end

-- คำนวณกำไรโดยใช้ Pascal function
function calculate_profit(cost, selling_price, qty)
    local revenue = calculate(selling_price, "*", qty)
    local total_cost = calculate(cost, "*", qty)
    local profit = calculate(revenue, "-", total_cost)
    return profit
end

-- รัน demo
print("=== Inventory Management Script ===")
print("วันที่: " .. getdata("date"))
print("")

add_product("P001", "สินค้า A", 100.0, 50)
add_product("P002", "สินค้า B", 250.0, 30)
add_product("P003", "สินค้า C", 75.0, 8)

print("")
sell_product("P001", 5)
sell_product("P002", 3)
sell_product("P003", 2)  -- จะแจ้งเตือนสินค้าใกล้หมด

print("")
generate_report()

-- คำนวณกำไร
local profit = calculate_profit(60, 100, 10)
print(string.format("\nกำไรจากการขาย 10 ชิ้น: %.2f บาท", profit))
```

---

## 62.5 โปรแกรมตัวอย่างสมบูรณ์: Script IDE

```pascal
program script_ide;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  StdCtrls, ExtCtrls, ComCtrls, Menus, Dialogs,
  SynEdit,          // Syntax highlighting editor
  SynHighlighterPas,
  lua_engine;

type
  TScriptIDEForm = class(TForm)
    // Menu
    MainMenu: TMainMenu;
    mnuFile: TMenuItem;
    mnuNew: TMenuItem;
    mnuOpen: TMenuItem;
    mnuSave: TMenuItem;
    mnuRun: TMenuItem;
    mnuRunScript: TMenuItem;
    mnuStop: TMenuItem;
    
    // Layout
    pnlMain: TPanel;
    splMain: TSplitter;
    pnlEditor: TPanel;
    pnlOutput: TPanel;
    
    // Editor
    editorScript: TSynEdit;
    highlighterPas: TSynPasSyn;
    
    // Output
    memoOutput: TMemo;
    lblOutput: TLabel;
    
    // Toolbar
    pnlToolbar: TPanel;
    btnNew: TButton;
    btnOpen: TButton;
    btnSave: TButton;
    btnRun: TButton;
    btnClear: TButton;
    cmbLanguage: TComboBox;
    lblLanguage: TLabel;
    
    // Status
    StatusBar: TStatusBar;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnNewClick(Sender: TObject);
    procedure btnOpenClick(Sender: TObject);
    procedure btnSaveClick(Sender: TObject);
    procedure btnRunClick(Sender: TObject);
    procedure btnClearClick(Sender: TObject);
    procedure cmbLanguageChange(Sender: TObject);
    procedure editorScriptChange(Sender: TObject);
    
  private
    FLuaEngine: TLuaEngine;
    FCurrentFile: string;
    FModified: Boolean;
    
    procedure Log(const AMsg: string);
    procedure UpdateTitle;
    procedure RunLuaScript;
    procedure InsertTemplate(const ALang: string);
    
  end;

{ TScriptIDEForm }

procedure TScriptIDEForm.FormCreate(Sender: TObject);
begin
  Caption := 'Script IDE - Lazarus';
  Width := 900;
  Height := 650;
  
  // สร้าง Lua Engine
  FLuaEngine := TLuaEngine.Create;
  if FLuaEngine.Initialize then
    Log('Lua engine พร้อมใช้งาน')
  else
    Log('ไม่สามารถเริ่ม Lua engine');
    
  // ตั้งค่า language selector
  cmbLanguage.Items.Clear;
  cmbLanguage.Items.Add('Lua');
  cmbLanguage.Items.Add('Pascal Script');
  cmbLanguage.ItemIndex := 0;
  
  // ตัวอย่าง script
  InsertTemplate('Lua');
  
  UpdateTitle;
end;

procedure TScriptIDEForm.FormDestroy(Sender: TObject);
begin
  FLuaEngine.Free;
end;

procedure TScriptIDEForm.Log(const AMsg: string);
begin
  memoOutput.Lines.Add('[' + FormatDateTime('hh:nn:ss', Now) + '] ' + AMsg);
  memoOutput.SelStart := Length(memoOutput.Text);
end;

procedure TScriptIDEForm.UpdateTitle;
var
  Title: string;
begin
  if FCurrentFile <> '' then
    Title := ExtractFileName(FCurrentFile)
  else
    Title := 'Untitled';
    
  if FModified then Title := '* ' + Title;
  Caption := Title + ' - Script IDE';
end;

procedure TScriptIDEForm.btnNewClick(Sender: TObject);
begin
  if FModified then
  begin
    case MessageDlg('ยืนยัน', 'บันทึกก่อนหรือไม่?',
      mtConfirmation, [mbYes, mbNo, mbCancel], 0) of
      mrYes: btnSaveClick(nil);
      mrCancel: Exit;
    end;
  end;
  
  editorScript.Clear;
  FCurrentFile := '';
  FModified := False;
  InsertTemplate(cmbLanguage.Text);
  UpdateTitle;
end;

procedure TScriptIDEForm.btnOpenClick(Sender: TObject);
var
  OD: TOpenDialog;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.Filter := 'Lua files (*.lua)|*.lua|Pascal files (*.pas)|*.pas|All (*.*)|*.*';
    if OD.Execute then
    begin
      editorScript.Lines.LoadFromFile(OD.FileName);
      FCurrentFile := OD.FileName;
      FModified := False;
      UpdateTitle;
      Log('เปิดไฟล์: ' + OD.FileName);
    end;
  finally
    OD.Free;
  end;
end;

procedure TScriptIDEForm.btnSaveClick(Sender: TObject);
var
  SD: TSaveDialog;
begin
  if FCurrentFile = '' then
  begin
    SD := TSaveDialog.Create(nil);
    try
      SD.Filter := 'Lua files (*.lua)|*.lua|Pascal files (*.pas)|*.pas';
      if cmbLanguage.ItemIndex = 0 then
        SD.DefaultExt := 'lua'
      else
        SD.DefaultExt := 'pas';
        
      if SD.Execute then
        FCurrentFile := SD.FileName
      else
        Exit;
    finally
      SD.Free;
    end;
  end;
  
  editorScript.Lines.SaveToFile(FCurrentFile);
  FModified := False;
  UpdateTitle;
  Log('บันทึกไฟล์: ' + FCurrentFile);
end;

procedure TScriptIDEForm.btnRunClick(Sender: TObject);
var
  StartTime: TDateTime;
  ElapsedMs: Int64;
begin
  Log('กำลังรัน script...');
  memoOutput.Lines.Add('--- Output ---');
  
  StartTime := Now;
  
  case cmbLanguage.ItemIndex of
    0: RunLuaScript;  // Lua
    1: Log('Pascal Script: ยังไม่รองรับในตัวอย่างนี้');
  end;
  
  ElapsedMs := Round((Now - StartTime) * 86400 * 1000);
  Log(Format('เสร็จสิ้น ใช้เวลา %d ms', [ElapsedMs]));
  
  StatusBar.SimpleText := 'รันสำเร็จ';
end;

procedure TScriptIDEForm.RunLuaScript;
var
  Script: string;
  i: Integer;
begin
  Script := editorScript.Text;
  
  if FLuaEngine.RunString(Script) then
  begin
    for i := 0 to FLuaEngine.Output.Count - 1 do
      memoOutput.Lines.Add('  ' + FLuaEngine.Output[i]);
  end
  else
  begin
    memoOutput.Lines.Add('*** ERROR ***');
    for i := 0 to FLuaEngine.Output.Count - 1 do
      memoOutput.Lines.Add('  ' + FLuaEngine.Output[i]);
  end;
end;

procedure TScriptIDEForm.btnClearClick(Sender: TObject);
begin
  memoOutput.Clear;
end;

procedure TScriptIDEForm.cmbLanguageChange(Sender: TObject);
begin
  // เปลี่ยน syntax highlighting
  case cmbLanguage.ItemIndex of
    0: editorScript.Highlighter := nil;  // Lua (no built-in)
    1: editorScript.Highlighter := highlighterPas;  // Pascal
  end;
end;

procedure TScriptIDEForm.editorScriptChange(Sender: TObject);
begin
  FModified := True;
  UpdateTitle;
  // อัปเดต line/col ใน status bar
  StatusBar.SimpleText := Format('บรรทัด %d, คอลัมน์ %d',
    [editorScript.CaretY, editorScript.CaretX]);
end;

procedure TScriptIDEForm.InsertTemplate(const ALang: string);
begin
  editorScript.Clear;
  
  if ALang = 'Lua' then
  begin
    editorScript.Lines.Add('-- Lua Script');
    editorScript.Lines.Add('-- วันที่: ' + FormatDateTime('yyyy-mm-dd', Now));
    editorScript.Lines.Add('');
    editorScript.Lines.Add('-- ฟังก์ชัน Hello World');
    editorScript.Lines.Add('function hello(name)');
    editorScript.Lines.Add('    print("สวัสดี " .. name .. "!")');
    editorScript.Lines.Add('    return "Hello from Lua"');
    editorScript.Lines.Add('end');
    editorScript.Lines.Add('');
    editorScript.Lines.Add('-- เรียกใช้งาน');
    editorScript.Lines.Add('hello("Pascal")');
    editorScript.Lines.Add('print("วันที่: " .. getdata("date"))');
  end
  else if ALang = 'Pascal Script' then
  begin
    editorScript.Lines.Add('{ Pascal Script }');
    editorScript.Lines.Add('var');
    editorScript.Lines.Add('  i: Integer;');
    editorScript.Lines.Add('begin');
    editorScript.Lines.Add('  for i := 1 to 5 do');
    editorScript.Lines.Add('    WriteLn(''Hello '' + IntToStr(i));');
    editorScript.Lines.Add('end.');
  end;
end;

var
  Form: TScriptIDEForm;
begin
  Application.Initialize;
  Application.CreateForm(TScriptIDEForm, Form);
  Application.Run;
end.
```

---

## 62.6 การจัดการ Sandbox

```pascal
unit script_sandbox;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  lua_engine;

type
  TSandboxLevel = (slStrict, slNormal, slRelaxed);
  
  TScriptSandbox = class
  private
    FEngine: TLuaEngine;
    FLevel: TSandboxLevel;
    FAllowedFunctions: TStringList;
    FBlockedPatterns: TStringList;
    FMaxOutputSize: Integer;
    FMaxLoopIterations: Integer;
    
    procedure SetupStrict;
    procedure SetupNormal;
    procedure SetupRelaxed;
    
  public
    constructor Create(ALevel: TSandboxLevel = slNormal);
    destructor Destroy; override;
    
    function Execute(const AScript: string; out AOutput: string): Boolean;
    
    property Level: TSandboxLevel read FLevel;
    property AllowedFunctions: TStringList read FAllowedFunctions;
  end;

implementation

constructor TScriptSandbox.Create(ALevel: TSandboxLevel);
begin
  inherited Create;
  FLevel := ALevel;
  FAllowedFunctions := TStringList.Create;
  FBlockedPatterns := TStringList.Create;
  FMaxOutputSize := 64 * 1024;  // 64KB
  FMaxLoopIterations := 1000000;
  
  FEngine := TLuaEngine.Create;
  FEngine.Initialize;
  
  case ALevel of
    slStrict:  SetupStrict;
    slNormal:  SetupNormal;
    slRelaxed: SetupRelaxed;
  end;
end;

destructor TScriptSandbox.Destroy;
begin
  FEngine.Free;
  FAllowedFunctions.Free;
  FBlockedPatterns.Free;
  inherited Destroy;
end;

procedure TScriptSandbox.SetupStrict;
begin
  // อนุญาตเฉพาะฟังก์ชันพื้นฐานที่ปลอดภัย
  FAllowedFunctions.Add('print');
  FAllowedFunctions.Add('tostring');
  FAllowedFunctions.Add('tonumber');
  FAllowedFunctions.Add('type');
  FAllowedFunctions.Add('pairs');
  FAllowedFunctions.Add('ipairs');
  FAllowedFunctions.Add('next');
  FAllowedFunctions.Add('select');
  FAllowedFunctions.Add('unpack');
  
  // บล็อก pattern อันตราย
  FBlockedPatterns.Add('io.');
  FBlockedPatterns.Add('os.');
  FBlockedPatterns.Add('require');
  FBlockedPatterns.Add('dofile');
  FBlockedPatterns.Add('loadfile');
  FBlockedPatterns.Add('load(');
  FBlockedPatterns.Add('pcall');
  FBlockedPatterns.Add('rawget');
  FBlockedPatterns.Add('rawset');
end;

procedure TScriptSandbox.SetupNormal;
begin
  SetupStrict;
  // เพิ่มฟังก์ชันที่อนุญาต
  FAllowedFunctions.Add('string.*');
  FAllowedFunctions.Add('math.*');
  FAllowedFunctions.Add('table.*');
  FBlockedPatterns.Delete(FBlockedPatterns.IndexOf('pcall'));
end;

procedure TScriptSandbox.SetupRelaxed;
begin
  SetupNormal;
  FAllowedFunctions.Add('io.*');
  // อนุญาต os functions บางส่วน
  FBlockedPatterns.Add('os.execute');
  FBlockedPatterns.Add('os.remove');
end;

function TScriptSandbox.Execute(const AScript: string; out AOutput: string): Boolean;
var
  i: Integer;
  Pattern: string;
  SafeScript: string;
begin
  Result := False;
  AOutput := '';
  
  // ตรวจสอบ blocked patterns
  for i := 0 to FBlockedPatterns.Count - 1 do
  begin
    Pattern := FBlockedPatterns[i];
    if Pos(Pattern, AScript) > 0 then
    begin
      AOutput := 'Security Error: Script contains blocked pattern: ' + Pattern;
      Exit;
    end;
  end;
  
  // เพิ่ม sandbox wrapper
  SafeScript := 
    '-- Sandbox wrapper' + LineEnding +
    'local _G_backup = {}' + LineEnding +
    AScript;
    
  if FEngine.RunString(SafeScript) then
  begin
    AOutput := FEngine.Output.Text;
    // จำกัดขนาด output
    if Length(AOutput) > FMaxOutputSize then
      AOutput := Copy(AOutput, 1, FMaxOutputSize) + '...(truncated)';
    Result := True;
  end
  else
    AOutput := FEngine.Output.Text;
end;

end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Pascal Script** - ใช้ภาษา Pascal เป็น scripting language
2. **Python Embedding** - ฝัง Python interpreter ในแอปพลิเคชัน Pascal
3. **Lua Scripting** - ใช้ Lua สำหรับ lightweight scripting
4. **Script IDE** - สร้าง IDE เล็กๆ สำหรับเขียน script
5. **Sandbox Security** - จำกัดสิทธิ์ script เพื่อความปลอดภัย

การเลือก scripting engine ขึ้นอยู่กับความต้องการ:
- **Pascal Script**: เหมาะสำหรับผู้ใช้ที่รู้จัก Pascal
- **Lua**: เบา เร็ว เหมาะสำหรับ game logic หรือ config
- **Python**: มี library มากมาย เหมาะสำหรับ data processing
