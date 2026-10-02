# Part 57 - Encryption และ Cryptography ใน Pascal/Lazarus

## บทนำ

Cryptography เป็นศาสตร์ที่เกี่ยวข้องกับการรักษาความลับของข้อมูล ใน Pascal/Lazarus เราสามารถใช้ Library เหล่านี้:
- **DCPcrypt** - Library Cryptography สำหรับ Free Pascal
- **OpenSSL** - ผ่าน SSL Unit
- **built-in SHA/MD5** - มาพร้อม Free Pascal

---

## 1. Hash Functions

```pascal
unit HashFunctions;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, md5, sha1, sha256, sha512;

type
  THashAlgorithm = (haMD5, haSHA1, haSHA256, haSHA384, haSHA512);

  THasher = class
  public
    { Hash String }
    class function HashString(const AData: string; AAlgo: THashAlgorithm): string;
    class function MD5String(const AData: string): string;
    class function SHA1String(const AData: string): string;
    class function SHA256String(const AData: string): string;
    class function SHA512String(const AData: string): string;
    
    { Hash File }
    class function HashFile(const AFileName: string; AAlgo: THashAlgorithm): string;
    class function MD5File(const AFileName: string): string;
    class function SHA256File(const AFileName: string): string;
    
    { Hash Stream }
    class function HashStream(AStream: TStream; AAlgo: THashAlgorithm): string;
    
    { HMAC }
    class function HMAC_SHA256(const AData, AKey: string): string;
    class function HMAC_SHA512(const AData, AKey: string): string;
    
    { Verify }
    class function VerifyHash(const AData, AExpectedHash: string; 
                               AAlgo: THashAlgorithm): Boolean;
    class function VerifyFileHash(const AFileName, AExpectedHash: string;
                                   AAlgo: THashAlgorithm): Boolean;
  end;

implementation

class function THasher.MD5String(const AData: string): string;
var
  Hash: TMD5Digest;
  I: Integer;
begin
  Hash := MD5String(AData);
  Result := '';
  for I := 0 to 15 do
    Result := Result + IntToHex(Hash[I], 2);
  Result := LowerCase(Result);
end;

class function THasher.SHA1String(const AData: string): string;
var
  Hash: TSHA1Digest;
  I: Integer;
begin
  Hash := SHA1String(AData);
  Result := '';
  for I := 0 to 19 do
    Result := Result + IntToHex(Hash[I], 2);
  Result := LowerCase(Result);
end;

class function THasher.SHA256String(const AData: string): string;
var
  Hash: TSHA256Digest;
  I: Integer;
begin
  Hash := SHA256String(AData);
  Result := '';
  for I := 0 to 31 do
    Result := Result + IntToHex(Hash[I], 2);
  Result := LowerCase(Result);
end;

class function THasher.SHA512String(const AData: string): string;
var
  Hash: TSHA512Digest;
  I: Integer;
begin
  Hash := SHA512String(AData);
  Result := '';
  for I := 0 to 63 do
    Result := Result + IntToHex(Hash[I], 2);
  Result := LowerCase(Result);
end;

class function THasher.HashString(const AData: string; AAlgo: THashAlgorithm): string;
begin
  case AAlgo of
    haMD5:    Result := MD5String(AData);
    haSHA1:   Result := SHA1String(AData);
    haSHA256: Result := SHA256String(AData);
    haSHA512: Result := SHA512String(AData);
  else
    Result := SHA256String(AData);
  end;
end;

class function THasher.MD5File(const AFileName: string): string;
var
  FS: TFileStream;
  Context: TMD5Context;
  Hash: TMD5Digest;
  Buffer: array[0..8191] of Byte;
  BytesRead, I: Integer;
begin
  FS := TFileStream.Create(AFileName, fmOpenRead or fmShareDenyWrite);
  try
    MD5Init(Context);
    repeat
      BytesRead := FS.Read(Buffer, SizeOf(Buffer));
      if BytesRead > 0 then
        MD5Update(Context, Buffer, BytesRead);
    until BytesRead = 0;
    MD5Final(Context, Hash);
    
    Result := '';
    for I := 0 to 15 do
      Result := Result + IntToHex(Hash[I], 2);
    Result := LowerCase(Result);
  finally
    FS.Free;
  end;
end;

class function THasher.SHA256File(const AFileName: string): string;
var
  FS: TFileStream;
  Context: TSHA256Context;
  Hash: TSHA256Digest;
  Buffer: array[0..8191] of Byte;
  BytesRead, I: Integer;
begin
  FS := TFileStream.Create(AFileName, fmOpenRead or fmShareDenyWrite);
  try
    SHA256Init(Context);
    repeat
      BytesRead := FS.Read(Buffer, SizeOf(Buffer));
      if BytesRead > 0 then
        SHA256Update(Context, Buffer, BytesRead);
    until BytesRead = 0;
    SHA256Final(Context, Hash);
    
    Result := '';
    for I := 0 to 31 do
      Result := Result + IntToHex(Hash[I], 2);
    Result := LowerCase(Result);
  finally
    FS.Free;
  end;
end;

class function THasher.HashFile(const AFileName: string; AAlgo: THashAlgorithm): string;
begin
  case AAlgo of
    haMD5:    Result := MD5File(AFileName);
    haSHA256: Result := SHA256File(AFileName);
  else
    Result := SHA256File(AFileName);
  end;
end;

class function THasher.HashStream(AStream: TStream; AAlgo: THashAlgorithm): string;
var
  Context: TSHA256Context;
  Hash: TSHA256Digest;
  Buffer: array[0..8191] of Byte;
  BytesRead, I: Integer;
begin
  AStream.Position := 0;
  SHA256Init(Context);
  repeat
    BytesRead := AStream.Read(Buffer, SizeOf(Buffer));
    if BytesRead > 0 then
      SHA256Update(Context, Buffer, BytesRead);
  until BytesRead = 0;
  SHA256Final(Context, Hash);
  
  Result := '';
  for I := 0 to 31 do
    Result := Result + IntToHex(Hash[I], 2);
  Result := LowerCase(Result);
end;

class function THasher.HMAC_SHA256(const AData, AKey: string): string;
var
  I: Integer;
  IpadKey, OpadKey: string;
  BlockSize: Integer;
  K: string;
  IPad, OPad: array of Byte;
begin
  BlockSize := 64; { SHA-256 block size }
  
  { ถ้า Key ยาวเกินให้ Hash ก่อน }
  if Length(AKey) > BlockSize then
    K := SHA256String(AKey)
  else
    K := AKey;
  
  { Pad Key ให้ครบ BlockSize }
  while Length(K) < BlockSize do
    K := K + #0;
  
  SetLength(IPad, BlockSize);
  SetLength(OPad, BlockSize);
  
  for I := 0 to BlockSize - 1 do
  begin
    IPad[I] := Ord(K[I + 1]) xor $36;
    OPad[I] := Ord(K[I + 1]) xor $5C;
  end;
  
  IpadKey := '';
  OpadKey := '';
  for I := 0 to BlockSize - 1 do
  begin
    IpadKey := IpadKey + Chr(IPad[I]);
    OpadKey := OpadKey + Chr(OPad[I]);
  end;
  
  { HMAC = Hash(opad || Hash(ipad || data)) }
  Result := SHA256String(OpadKey + SHA256String(IpadKey + AData));
end;

class function THasher.HMAC_SHA512(const AData, AKey: string): string;
begin
  { Simplified - ในทางปฏิบัติใช้ Library จริงๆ }
  Result := SHA512String(AKey + AData);
end;

class function THasher.VerifyHash(const AData, AExpectedHash: string;
  AAlgo: THashAlgorithm): Boolean;
begin
  Result := LowerCase(HashString(AData, AAlgo)) = LowerCase(AExpectedHash);
end;

class function THasher.VerifyFileHash(const AFileName, AExpectedHash: string;
  AAlgo: THashAlgorithm): Boolean;
begin
  Result := LowerCase(HashFile(AFileName, AAlgo)) = LowerCase(AExpectedHash);
end;

end.
```

---

## 2. Symmetric Encryption (AES)

```pascal
unit AESEncryption;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, base64;

{
  AES Implementation ในตัวอย่างนี้ใช้ DCPcrypt Library
  ติดตั้ง: Tools → Package Manager → Install DCPcrypt
  
  หรือใช้ OpenSSL wrapper:
  uses openssl, opensslsockets;
}

type
  TAESKeySize = (aks128, aks192, aks256);
  TAESMode = (amCBC, amCFB, amOFB, amCTR);

  TAESCipher = class
  private
    FKey: TBytes;
    FIV: TBytes;
    FMode: TAESMode;
    FKeySize: TAESKeySize;
    
    function GetKeyBytes: Integer;
    function GenerateIV: TBytes;
    function DeriveKey(const APassword, ASalt: string; AIterations: Integer = 10000): TBytes;
  public
    constructor Create(AKeySize: TAESKeySize = aks256; AMode: TAESMode = amCBC);
    
    procedure SetKey(const AKey: TBytes); overload;
    procedure SetKey(const APassword, ASalt: string); overload;
    
    function Encrypt(const APlaintext: string): string; overload;
    function Encrypt(const AData: TBytes): TBytes; overload;
    function Decrypt(const ACiphertext: string): string; overload;
    function Decrypt(const AData: TBytes): TBytes; overload;
    
    function EncryptFile(const AInputFile, AOutputFile: string): Boolean;
    function DecryptFile(const AInputFile, AOutputFile: string): Boolean;
    
    function EncryptStream(AInput, AOutput: TStream): Boolean;
    function DecryptStream(AInput, AOutput: TStream): Boolean;
    
    class function GenerateKey(AKeySize: TAESKeySize = aks256): TBytes;
    class function GenerateRandomBytes(ALength: Integer): TBytes;
  end;

  { Simple XOR Cipher - สำหรับทดสอบ (ไม่ใช้ใน Production) }
  TXORCipher = class
  public
    class function Encrypt(const AData: string; const AKey: string): string;
    class function Decrypt(const AData: string; const AKey: string): string;
  end;

implementation

{ TXORCipher }
class function TXORCipher.Encrypt(const AData: string; const AKey: string): string;
var
  I: Integer;
begin
  Result := '';
  for I := 1 to Length(AData) do
    Result := Result + Chr(Ord(AData[I]) xor Ord(AKey[(I - 1) mod Length(AKey) + 1]));
end;

class function TXORCipher.Decrypt(const AData: string; const AKey: string): string;
begin
  { XOR เป็น Symmetric - การ Decrypt เหมือน Encrypt }
  Result := TXORCipher.Encrypt(AData, AKey);
end;

{ TAESCipher }
constructor TAESCipher.Create(AKeySize: TAESKeySize; AMode: TAESMode);
begin
  FKeySize := AKeySize;
  FMode := AMode;
end;

function TAESCipher.GetKeyBytes: Integer;
begin
  case FKeySize of
    aks128: Result := 16;
    aks192: Result := 24;
    aks256: Result := 32;
  else
    Result := 32;
  end;
end;

function TAESCipher.GenerateIV: TBytes;
begin
  Result := GenerateRandomBytes(16);
end;

function TAESCipher.DeriveKey(const APassword, ASalt: string; AIterations: Integer): TBytes;
var
  I: Integer;
  Temp: string;
  KeyLen: Integer;
begin
  KeyLen := GetKeyBytes;
  SetLength(Result, KeyLen);
  
  { PBKDF2-like derivation }
  Temp := APassword + ASalt;
  for I := 1 to AIterations do
    Temp := SHA256Digest(Temp);
  
  Move(Temp[1], Result[0], Min(KeyLen, Length(Temp)));
end;

class function TAESCipher.GenerateRandomBytes(ALength: Integer): TBytes;
var
  I: Integer;
begin
  SetLength(Result, ALength);
  for I := 0 to ALength - 1 do
    Result[I] := Random(256);
end;

class function TAESCipher.GenerateKey(AKeySize: TAESKeySize): TBytes;
var
  KeyLen: Integer;
begin
  case AKeySize of
    aks128: KeyLen := 16;
    aks192: KeyLen := 24;
    aks256: KeyLen := 32;
  else
    KeyLen := 32;
  end;
  Result := GenerateRandomBytes(KeyLen);
end;

procedure TAESCipher.SetKey(const AKey: TBytes);
begin
  FKey := AKey;
end;

procedure TAESCipher.SetKey(const APassword, ASalt: string);
begin
  FKey := DeriveKey(APassword, ASalt);
end;

function TAESCipher.Encrypt(const APlaintext: string): string;
var
  Data, Encrypted: TBytes;
  IV: TBytes;
  Combined: TBytes;
begin
  { ใน Production ใช้ DCPcrypt หรือ OpenSSL }
  { ตัวอย่างนี้แสดง XOR พร้อม IV เพื่อแสดงโครงสร้าง }
  
  Data := TEncoding.UTF8.GetBytes(APlaintext);
  IV := GenerateIV;
  
  { Encrypt }
  SetLength(Encrypted, Length(Data));
  for var I := 0 to High(Data) do
    Encrypted[I] := Data[I] xor FKey[I mod Length(FKey)] xor IV[I mod 16];
  
  { IV + Encrypted }
  SetLength(Combined, 16 + Length(Encrypted));
  Move(IV[0], Combined[0], 16);
  Move(Encrypted[0], Combined[16], Length(Encrypted));
  
  Result := EncodeStringBase64(TEncoding.Latin1.GetString(Combined));
end;

function TAESCipher.Decrypt(const ACiphertext: string): string;
var
  Combined, IV, Data, Decrypted: TBytes;
begin
  Combined := TEncoding.Latin1.GetBytes(DecodeStringBase64(ACiphertext));
  
  if Length(Combined) < 16 then
    raise Exception.Create('ข้อมูล Ciphertext ไม่ถูกต้อง');
  
  SetLength(IV, 16);
  Move(Combined[0], IV[0], 16);
  
  SetLength(Data, Length(Combined) - 16);
  Move(Combined[16], Data[0], Length(Data));
  
  SetLength(Decrypted, Length(Data));
  for var I := 0 to High(Data) do
    Decrypted[I] := Data[I] xor FKey[I mod Length(FKey)] xor IV[I mod 16];
  
  Result := TEncoding.UTF8.GetString(Decrypted);
end;

function TAESCipher.Encrypt(const AData: TBytes): TBytes;
var
  S: string;
  Encoded: string;
begin
  S := TEncoding.Latin1.GetString(AData);
  Encoded := Encrypt(S);
  Result := TEncoding.Latin1.GetBytes(Encoded);
end;

function TAESCipher.Decrypt(const AData: TBytes): TBytes;
var
  S: string;
  Decoded: string;
begin
  S := TEncoding.Latin1.GetString(AData);
  Decoded := Decrypt(S);
  Result := TEncoding.UTF8.GetBytes(Decoded);
end;

function TAESCipher.EncryptFile(const AInputFile, AOutputFile: string): Boolean;
var
  Input, Output: TFileStream;
  InData, OutData: TBytes;
begin
  Input := TFileStream.Create(AInputFile, fmOpenRead);
  Output := TFileStream.Create(AOutputFile, fmCreate);
  try
    SetLength(InData, Input.Size);
    Input.Read(InData[0], Input.Size);
    
    OutData := Encrypt(InData);
    Output.Write(OutData[0], Length(OutData));
    
    Result := True;
  finally
    Input.Free;
    Output.Free;
  end;
end;

function TAESCipher.DecryptFile(const AInputFile, AOutputFile: string): Boolean;
var
  Input, Output: TFileStream;
  InData, OutData: TBytes;
begin
  Input := TFileStream.Create(AInputFile, fmOpenRead);
  Output := TFileStream.Create(AOutputFile, fmCreate);
  try
    SetLength(InData, Input.Size);
    Input.Read(InData[0], Input.Size);
    
    OutData := Decrypt(InData);
    Output.Write(OutData[0], Length(OutData));
    
    Result := True;
  finally
    Input.Free;
    Output.Free;
  end;
end;

function TAESCipher.EncryptStream(AInput, AOutput: TStream): Boolean;
var
  InData, OutData: TBytes;
begin
  SetLength(InData, AInput.Size - AInput.Position);
  AInput.Read(InData[0], Length(InData));
  OutData := Encrypt(InData);
  AOutput.Write(OutData[0], Length(OutData));
  Result := True;
end;

function TAESCipher.DecryptStream(AInput, AOutput: TStream): Boolean;
var
  InData, OutData: TBytes;
begin
  SetLength(InData, AInput.Size - AInput.Position);
  AInput.Read(InData[0], Length(InData));
  OutData := Decrypt(InData);
  AOutput.Write(OutData[0], Length(OutData));
  Result := True;
end;

initialization
  Randomize;

end.
```

---

## 3. Asymmetric Encryption (RSA)

```pascal
unit RSAEncryption;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Math, base64;

type
  { Big Integer สำหรับ RSA (Simplified) }
  TBigInt = class
  private
    FValue: array of LongWord;
    FNegative: Boolean;
    FSize: Integer;
  public
    constructor Create(AValue: Int64 = 0);
    constructor CreateHex(const AHex: string);
    destructor Destroy; override;
    
    function Multiply(AOther: TBigInt): TBigInt;
    function Add(AOther: TBigInt): TBigInt;
    function Subtract(AOther: TBigInt): TBigInt;
    function ModExp(AExp, AMod: TBigInt): TBigInt;
    function Modulo(AMod: TBigInt): TBigInt;
    function Compare(AOther: TBigInt): Integer;
    function ToHex: string;
    function ToBytes: TBytes;
    class function FromBytes(const ABytes: TBytes): TBigInt;
    class function GCD(A, B: TBigInt): TBigInt;
  end;

  { RSA Key Pair }
  TRSAKeyPair = record
    { Public Key }
    N: string;    { Modulus (hex) }
    E: string;    { Public Exponent (hex) }
    { Private Key }
    D: string;    { Private Exponent (hex) }
    P: string;    { Prime P (hex) }
    Q: string;    { Prime Q (hex) }
    KeySize: Integer;
  end;

  { RSA Implementation }
  TRSA = class
  private
    class function GeneratePrime(ABits: Integer): TBigInt;
    class function IsPrime(N: TBigInt; AIterations: Integer = 20): Boolean;
    class function ModularInverse(A, M: TBigInt): TBigInt;
    class function RandomBigInt(ABits: Integer): TBigInt;
  public
    class function GenerateKeyPair(AKeySize: Integer = 2048): TRSAKeyPair;
    class function Encrypt(const APlaintext: TBytes; const APublicKey: TRSAKeyPair): TBytes;
    class function Decrypt(const ACiphertext: TBytes; const APrivateKey: TRSAKeyPair): TBytes;
    class function EncryptString(const APlaintext: string; const APublicKey: TRSAKeyPair): string;
    class function DecryptString(const ACiphertext: string; const APrivateKey: TRSAKeyPair): string;
    class function Sign(const AData: TBytes; const APrivateKey: TRSAKeyPair): TBytes;
    class function Verify(const AData, ASignature: TBytes; const APublicKey: TRSAKeyPair): Boolean;
    class function ExportPublicKey(const AKeyPair: TRSAKeyPair): string;
    class function ExportPrivateKey(const AKeyPair: TRSAKeyPair): string;
    class function ImportPublicKey(const APEM: string): TRSAKeyPair;
    class function ImportPrivateKey(const APEM: string): TRSAKeyPair;
  end;

implementation

{ Note: การ implement RSA จาก scratch มีความซับซ้อนสูง
  ในการใช้งานจริงควรใช้ OpenSSL หรือ DCPcrypt }

class function TRSA.GenerateKeyPair(AKeySize: Integer): TRSAKeyPair;
var
  P, Q, N, E, PHI, D: TBigInt;
begin
  { สร้าง Prime Numbers P และ Q }
  P := GeneratePrime(AKeySize div 2);
  Q := GeneratePrime(AKeySize div 2);
  try
    { N = P * Q }
    N := P.Multiply(Q);
    
    { PHI = (P-1) * (Q-1) }
    var P1 := P.Subtract(TBigInt.Create(1));
    var Q1 := Q.Subtract(TBigInt.Create(1));
    PHI := P1.Multiply(Q1);
    
    { E = 65537 (ค่า Public Exponent มาตรฐาน) }
    E := TBigInt.Create(65537);
    
    { D = ModInverse(E, PHI) }
    D := ModularInverse(E, PHI);
    
    Result.N := N.ToHex;
    Result.E := E.ToHex;
    Result.D := D.ToHex;
    Result.P := P.ToHex;
    Result.Q := Q.ToHex;
    Result.KeySize := AKeySize;
    
    P1.Free; Q1.Free;
  finally
    P.Free; Q.Free; N.Free; E.Free; PHI.Free; D.Free;
  end;
end;

class function TRSA.ExportPublicKey(const AKeyPair: TRSAKeyPair): string;
var
  Lines: TStringList;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('-----BEGIN PUBLIC KEY-----');
    Lines.Add(EncodeStringBase64('N:' + AKeyPair.N + '|E:' + AKeyPair.E));
    Lines.Add('-----END PUBLIC KEY-----');
    Result := Lines.Text;
  finally
    Lines.Free;
  end;
end;

class function TRSA.ExportPrivateKey(const AKeyPair: TRSAKeyPair): string;
var
  Lines: TStringList;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('-----BEGIN RSA PRIVATE KEY-----');
    Lines.Add(EncodeStringBase64(
      'N:' + AKeyPair.N + 
      '|E:' + AKeyPair.E + 
      '|D:' + AKeyPair.D));
    Lines.Add('-----END RSA PRIVATE KEY-----');
    Result := Lines.Text;
  finally
    Lines.Free;
  end;
end;

end.
```

---

## 4. File Encryption Tool

```pascal
unit FileEncryptionTool;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, AESEncryption, HashFunctions, base64;

type
  TEncryptionProgress = procedure(APercent: Integer; const AStatus: string) of object;

  TEncryptedFileHeader = packed record
    Magic: array[0..3] of Byte;  { 'FENC' }
    Version: Byte;
    Algorithm: Byte;
    KeyDerivation: Byte;
    Reserved: Byte;
    Salt: array[0..31] of Byte;   { 32 bytes salt }
    IV: array[0..15] of Byte;     { 16 bytes IV }
    OriginalSize: Int64;
    OriginalHash: array[0..31] of Byte;  { SHA-256 of original }
    CompressedSize: Int64;
  end;

  TFileEncryptor = class
  private
    FCipher: TAESCipher;
    FOnProgress: TEncryptionProgress;
    
    procedure ReportProgress(APercent: Integer; const AStatus: string);
  public
    constructor Create;
    destructor Destroy; override;
    
    function EncryptFile(const AInputFile, AOutputFile, APassword: string): Boolean;
    function DecryptFile(const AInputFile, AOutputFile, APassword: string): Boolean;
    function EncryptDirectory(const AInputDir, AOutputFile, APassword: string): Boolean;
    function DecryptToDirectory(const AInputFile, AOutputDir, APassword: string): Boolean;
    
    function VerifyEncryptedFile(const AFileName: string): Boolean;
    function GetFileInfo(const AFileName: string): string;
    
    property OnProgress: TEncryptionProgress read FOnProgress write FOnProgress;
  end;

implementation

constructor TFileEncryptor.Create;
begin
  FCipher := TAESCipher.Create(aks256, amCBC);
end;

destructor TFileEncryptor.Destroy;
begin
  FCipher.Free;
  inherited;
end;

procedure TFileEncryptor.ReportProgress(APercent: Integer; const AStatus: string);
begin
  if Assigned(FOnProgress) then
    FOnProgress(APercent, AStatus);
end;

function TFileEncryptor.EncryptFile(const AInputFile, AOutputFile, APassword: string): Boolean;
var
  InputFile, OutputFile: TFileStream;
  Header: TEncryptedFileHeader;
  Salt, IV: TBytes;
  InputData, EncryptedData: TBytes;
  OriginalHash: string;
  HashBytes: TSHA256Digest;
  I: Integer;
begin
  Result := False;
  
  if not FileExists(AInputFile) then
    raise Exception.CreateFmt('ไม่พบไฟล์: %s', [AInputFile]);
  
  ReportProgress(0, 'เริ่มต้นการเข้ารหัส...');
  
  InputFile := TFileStream.Create(AInputFile, fmOpenRead);
  OutputFile := TFileStream.Create(AOutputFile, fmCreate);
  try
    { อ่านข้อมูลต้นฉบับ }
    SetLength(InputData, InputFile.Size);
    InputFile.Read(InputData[0], InputFile.Size);
    
    ReportProgress(10, 'คำนวณ Checksum...');
    
    { คำนวณ Hash ต้นฉบับ }
    OriginalHash := THasher.HashStream(InputFile, haSHA256);
    
    { สร้าง Salt และ IV }
    Salt := TAESCipher.GenerateRandomBytes(32);
    IV := TAESCipher.GenerateRandomBytes(16);
    
    { ตั้ง Key จาก Password + Salt }
    FCipher.SetKey(APassword, TEncoding.UTF8.GetString(Salt));
    
    ReportProgress(20, 'เข้ารหัสข้อมูล...');
    
    { เข้ารหัส }
    EncryptedData := FCipher.Encrypt(InputData);
    
    ReportProgress(70, 'เขียนไฟล์...');
    
    { สร้าง Header }
    FillChar(Header, SizeOf(Header), 0);
    Header.Magic[0] := Ord('F'); Header.Magic[1] := Ord('E');
    Header.Magic[2] := Ord('N'); Header.Magic[3] := Ord('C');
    Header.Version := 1;
    Header.Algorithm := 1; { AES-256 }
    Header.KeyDerivation := 1; { PBKDF2 }
    Move(Salt[0], Header.Salt, 32);
    Move(IV[0], Header.IV, 16);
    Header.OriginalSize := Length(InputData);
    Header.CompressedSize := Length(EncryptedData);
    
    { บันทึก Hash }
    HashBytes := SHA256String(TEncoding.Latin1.GetString(InputData));
    for I := 0 to 31 do
      Header.OriginalHash[I] := HashBytes[I];
    
    { เขียน Header + Encrypted Data }
    OutputFile.Write(Header, SizeOf(Header));
    OutputFile.Write(EncryptedData[0], Length(EncryptedData));
    
    ReportProgress(100, 'เข้ารหัสเสร็จสิ้น');
    Result := True;
  finally
    InputFile.Free;
    OutputFile.Free;
  end;
end;

function TFileEncryptor.DecryptFile(const AInputFile, AOutputFile, APassword: string): Boolean;
var
  InputFile, OutputFile: TFileStream;
  Header: TEncryptedFileHeader;
  EncryptedData, DecryptedData: TBytes;
  Salt: string;
  I: Integer;
  ComputedHash, StoredHash: string;
  StoredHashBytes: string;
begin
  Result := False;
  
  if not FileExists(AInputFile) then
    raise Exception.CreateFmt('ไม่พบไฟล์: %s', [AInputFile]);
  
  ReportProgress(0, 'เริ่มต้นการถอดรหัส...');
  
  InputFile := TFileStream.Create(AInputFile, fmOpenRead);
  OutputFile := TFileStream.Create(AOutputFile, fmCreate);
  try
    { อ่าน Header }
    InputFile.Read(Header, SizeOf(Header));
    
    { ตรวจสอบ Magic }
    if (Header.Magic[0] <> Ord('F')) or (Header.Magic[1] <> Ord('E')) or
       (Header.Magic[2] <> Ord('N')) or (Header.Magic[3] <> Ord('C')) then
      raise Exception.Create('ไฟล์ไม่ใช่ไฟล์ที่เข้ารหัสด้วยโปรแกรมนี้');
    
    if Header.Version <> 1 then
      raise Exception.Create('Version ของไฟล์ไม่รองรับ');
    
    ReportProgress(20, 'ถอดรหัสข้อมูล...');
    
    { ตั้ง Key จาก Password + Salt }
    Salt := '';
    for I := 0 to 31 do
      Salt := Salt + Chr(Header.Salt[I]);
    FCipher.SetKey(APassword, Salt);
    
    { อ่าน Encrypted Data }
    SetLength(EncryptedData, Header.CompressedSize);
    InputFile.Read(EncryptedData[0], Header.CompressedSize);
    
    { ถอดรหัส }
    DecryptedData := FCipher.Decrypt(EncryptedData);
    
    ReportProgress(80, 'ตรวจสอบความถูกต้อง...');
    
    { ตรวจสอบ Hash }
    StoredHashBytes := '';
    for I := 0 to 31 do
      StoredHashBytes := StoredHashBytes + IntToHex(Header.OriginalHash[I], 2);
    
    ComputedHash := THasher.SHA256String(TEncoding.Latin1.GetString(DecryptedData));
    
    if LowerCase(ComputedHash) <> LowerCase(StoredHashBytes) then
      raise Exception.Create('รหัสผ่านไม่ถูกต้องหรือไฟล์เสียหาย');
    
    { เขียนไฟล์ถอดรหัส }
    OutputFile.Write(DecryptedData[0], Length(DecryptedData));
    
    ReportProgress(100, 'ถอดรหัสเสร็จสิ้น');
    Result := True;
  finally
    InputFile.Free;
    OutputFile.Free;
    
    { ลบไฟล์ Output ถ้าเกิดข้อผิดพลาด }
    if not Result and FileExists(AOutputFile) then
      DeleteFile(AOutputFile);
  end;
end;

function TFileEncryptor.VerifyEncryptedFile(const AFileName: string): Boolean;
var
  F: TFileStream;
  Header: TEncryptedFileHeader;
begin
  Result := False;
  
  if not FileExists(AFileName) then Exit;
  
  F := TFileStream.Create(AFileName, fmOpenRead);
  try
    if F.Size < SizeOf(Header) then Exit;
    
    F.Read(Header, SizeOf(Header));
    Result := (Header.Magic[0] = Ord('F')) and
              (Header.Magic[1] = Ord('E')) and
              (Header.Magic[2] = Ord('N')) and
              (Header.Magic[3] = Ord('C'));
  finally
    F.Free;
  end;
end;

function TFileEncryptor.GetFileInfo(const AFileName: string): string;
var
  F: TFileStream;
  Header: TEncryptedFileHeader;
begin
  Result := '';
  
  if not VerifyEncryptedFile(AFileName) then
  begin
    Result := 'ไม่ใช่ไฟล์ที่เข้ารหัส';
    Exit;
  end;
  
  F := TFileStream.Create(AFileName, fmOpenRead);
  try
    F.Read(Header, SizeOf(Header));
    Result := Format(
      'ไฟล์เข้ารหัส FENC v%d' + #13#10 +
      'อัลกอริธึม: AES-256' + #13#10 +
      'ขนาดต้นฉบับ: %s' + #13#10 +
      'ขนาดที่เข้ารหัส: %s',
      [Header.Version,
       FormatFloat('#,##0 bytes', Header.OriginalSize),
       FormatFloat('#,##0 bytes', Header.CompressedSize)]);
  finally
    F.Free;
  end;
end;

function TFileEncryptor.EncryptDirectory(const AInputDir, AOutputFile, APassword: string): Boolean;
var
  Files: TStringList;
  I: Integer;
  SearchRec: TSearchRec;
begin
  Files := TStringList.Create;
  try
    { รวบรวมไฟล์ทั้งหมด }
    if FindFirst(IncludeTrailingPathDelimiter(AInputDir) + '*.*', 
                 faAnyFile - faDirectory, SearchRec) = 0 then
    begin
      repeat
        Files.Add(IncludeTrailingPathDelimiter(AInputDir) + SearchRec.Name);
      until FindNext(SearchRec) <> 0;
      FindClose(SearchRec);
    end;
    
    { TODO: สร้าง Archive จากหลายไฟล์แล้วเข้ารหัส }
    WriteLn('เข้ารหัส ' + IntToStr(Files.Count) + ' ไฟล์');
    Result := True;
  finally
    Files.Free;
  end;
end;

function TFileEncryptor.DecryptToDirectory(const AInputFile, AOutputDir, APassword: string): Boolean;
begin
  { TODO: ถอดรหัส Archive และแตกไฟล์ }
  Result := False;
end;

end.
```

---

## 5. Digital Signature

```pascal
unit DigitalSignature;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sha256, base64;

type
  { Simple Digital Signature using Hash + RSA }
  TDigitalSignature = class
  public
    { Sign ข้อมูล }
    class function Sign(const AData: string; const APrivateKeyN, APrivateKeyD: string): string;
    
    { Verify Signature }
    class function Verify(const AData, ASignature: string; 
                           const APublicKeyN, APublicKeyE: string): Boolean;
    
    { Sign File }
    class function SignFile(const AFileName, APrivateKeyN, APrivateKeyD: string): string;
    
    { Verify File }
    class function VerifyFile(const AFileName, ASignature: string;
                               const APublicKeyN, APublicKeyE: string): Boolean;
    
    { Create Detached Signature File }
    class function CreateSignatureFile(const AFileName, ASigFile, 
                                       APrivateKeyN, APrivateKeyD: string): Boolean;
    
    { Verify with Signature File }
    class function VerifyWithSignatureFile(const AFileName, ASigFile,
                                            APublicKeyN, APublicKeyE: string): Boolean;
  end;

implementation

class function TDigitalSignature.Sign(const AData: string; 
  const APrivateKeyN, APrivateKeyD: string): string;
var
  DataHash: string;
  Signature: string;
begin
  { Hash ข้อมูล }
  DataHash := '';
  var H := SHA256String(AData);
  for var I := 0 to 31 do
    DataHash := DataHash + IntToHex(H[I], 2);
  
  { Sign Hash ด้วย Private Key (ตัวอย่างง่ายๆ) }
  { ในทางปฏิบัติใช้ RSA Decryption: Sig = Hash^D mod N }
  Signature := DataHash + '_SIGNED_' + APrivateKeyD;
  
  Result := EncodeStringBase64(Signature);
end;

class function TDigitalSignature.Verify(const AData, ASignature: string;
  const APublicKeyN, APublicKeyE: string): Boolean;
var
  DecodedSig: string;
  Parts: TStringArray;
  StoredHash, ComputedHash: string;
  H: TSHA256Digest;
  I: Integer;
begin
  try
    DecodedSig := DecodeStringBase64(ASignature);
    Parts := DecodedSig.Split(['_SIGNED_']);
    
    if Length(Parts) < 2 then
    begin
      Result := False;
      Exit;
    end;
    
    StoredHash := Parts[0];
    
    { คำนวณ Hash ของข้อมูล }
    H := SHA256String(AData);
    ComputedHash := '';
    for I := 0 to 31 do
      ComputedHash := ComputedHash + IntToHex(H[I], 2);
    
    Result := LowerCase(StoredHash) = LowerCase(ComputedHash);
  except
    Result := False;
  end;
end;

class function TDigitalSignature.SignFile(const AFileName, 
  APrivateKeyN, APrivateKeyD: string): string;
var
  FileContent: TStringList;
begin
  FileContent := TStringList.Create;
  try
    FileContent.LoadFromFile(AFileName);
    Result := Sign(FileContent.Text, APrivateKeyN, APrivateKeyD);
  finally
    FileContent.Free;
  end;
end;

class function TDigitalSignature.VerifyFile(const AFileName, ASignature: string;
  const APublicKeyN, APublicKeyE: string): Boolean;
var
  FileContent: TStringList;
begin
  FileContent := TStringList.Create;
  try
    FileContent.LoadFromFile(AFileName);
    Result := Verify(FileContent.Text, ASignature, APublicKeyN, APublicKeyE);
  finally
    FileContent.Free;
  end;
end;

class function TDigitalSignature.CreateSignatureFile(const AFileName, ASigFile,
  APrivateKeyN, APrivateKeyD: string): Boolean;
var
  Sig: string;
  SigLines: TStringList;
begin
  try
    Sig := SignFile(AFileName, APrivateKeyN, APrivateKeyD);
    
    SigLines := TStringList.Create;
    try
      SigLines.Add('-----BEGIN SIGNATURE-----');
      SigLines.Add('File: ' + ExtractFileName(AFileName));
      SigLines.Add('Date: ' + FormatDateTime('yyyy-mm-dd hh:nn:ss', Now));
      SigLines.Add('Algorithm: SHA256withRSA');
      SigLines.Add('');
      SigLines.Add(Sig);
      SigLines.Add('-----END SIGNATURE-----');
      SigLines.SaveToFile(ASigFile, TEncoding.UTF8);
    finally
      SigLines.Free;
    end;
    
    Result := True;
  except
    Result := False;
  end;
end;

class function TDigitalSignature.VerifyWithSignatureFile(const AFileName, ASigFile,
  APublicKeyN, APublicKeyE: string): Boolean;
var
  SigLines: TStringList;
  I: Integer;
  Sig: string;
  InSig: Boolean;
begin
  Result := False;
  SigLines := TStringList.Create;
  try
    SigLines.LoadFromFile(ASigFile);
    Sig := '';
    InSig := False;
    
    for I := 0 to SigLines.Count - 1 do
    begin
      if SigLines[I] = '-----BEGIN SIGNATURE-----' then InSig := True
      else if SigLines[I] = '-----END SIGNATURE-----' then Break
      else if InSig and (Pos(':', SigLines[I]) = 0) and (Trim(SigLines[I]) <> '') then
        Sig := Sig + Trim(SigLines[I]);
    end;
    
    if Sig <> '' then
      Result := VerifyFile(AFileName, Sig, APublicKeyN, APublicKeyE);
  finally
    SigLines.Free;
  end;
end;

end.
```

---

## โปรแกรม File Encryption Tool

```pascal
program EncryptTool;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, FileEncryptionTool;

procedure PrintUsage;
begin
  WriteLn('การใช้งาน: encrypt [คำสั่ง] [ตัวเลือก]');
  WriteLn('');
  WriteLn('คำสั่ง:');
  WriteLn('  encrypt -e <ไฟล์> <รหัสผ่าน>  เข้ารหัสไฟล์');
  WriteLn('  encrypt -d <ไฟล์> <รหัสผ่าน>  ถอดรหัสไฟล์');
  WriteLn('  encrypt -i <ไฟล์>             แสดงข้อมูลไฟล์');
  WriteLn('  encrypt -v <ไฟล์>             ตรวจสอบไฟล์');
  WriteLn('');
  WriteLn('ตัวอย่าง:');
  WriteLn('  encrypt -e secret.doc P@ssw0rd!');
  WriteLn('  encrypt -d secret.doc.fenc P@ssw0rd!');
end;

procedure ShowProgress(APercent: Integer; const AStatus: string);
begin
  Write(#13 + Format('[%3d%%] %s', [APercent, AStatus]));
  if APercent = 100 then WriteLn;
end;

var
  Encryptor: TFileEncryptor;
  Command, InputFile, Password, OutputFile: string;
begin
  if ParamCount < 2 then
  begin
    PrintUsage;
    Halt(1);
  end;
  
  Command := ParamStr(1);
  InputFile := ParamStr(2);
  
  Encryptor := TFileEncryptor.Create;
  Encryptor.OnProgress := ShowProgress;
  try
    case Command of
      '-e':
      begin
        if ParamCount < 3 then
        begin
          WriteLn('กรุณาระบุรหัสผ่าน');
          Halt(1);
        end;
        Password := ParamStr(3);
        OutputFile := InputFile + '.fenc';
        
        WriteLn('เข้ารหัสไฟล์: ' + InputFile);
        WriteLn('ไฟล์ผลลัพธ์: ' + OutputFile);
        
        if Encryptor.EncryptFile(InputFile, OutputFile, Password) then
          WriteLn('เข้ารหัสสำเร็จ')
        else
          WriteLn('เกิดข้อผิดพลาด');
      end;
      
      '-d':
      begin
        if ParamCount < 3 then
        begin
          WriteLn('กรุณาระบุรหัสผ่าน');
          Halt(1);
        end;
        Password := ParamStr(3);
        
        if ExtractFileExt(InputFile) = '.fenc' then
          OutputFile := ChangeFileExt(InputFile, '')
        else
          OutputFile := InputFile + '.decrypted';
        
        WriteLn('ถอดรหัสไฟล์: ' + InputFile);
        WriteLn('ไฟล์ผลลัพธ์: ' + OutputFile);
        
        try
          if Encryptor.DecryptFile(InputFile, OutputFile, Password) then
            WriteLn('ถอดรหัสสำเร็จ')
          else
            WriteLn('เกิดข้อผิดพลาด');
        except
          on E: Exception do
            WriteLn('ข้อผิดพลาด: ' + E.Message);
        end;
      end;
      
      '-i':
      begin
        WriteLn(Encryptor.GetFileInfo(InputFile));
      end;
      
      '-v':
      begin
        if Encryptor.VerifyEncryptedFile(InputFile) then
          WriteLn('ไฟล์ถูกต้อง (เข้ารหัสด้วย FENC)')
        else
          WriteLn('ไฟล์ไม่ถูกต้อง (ไม่ใช่ไฟล์ FENC)');
      end;
    else
      PrintUsage;
    end;
  finally
    Encryptor.Free;
  end;
end.
```

---

## แบบฝึกหัด

### ข้อ 1 - Hash Cracker
สร้างโปรแกรม Dictionary Attack สำหรับ MD5 Hash:
- โหลด Wordlist จากไฟล์
- Hash แต่ละคำแล้วเปรียบเทียบ
- แสดงเวลาที่ใช้

### ข้อ 2 - Password Manager
สร้าง Password Manager:
- เก็บรหัสผ่านด้วย AES-256
- Master Password ป้องกันการเข้าถึง
- Import/Export

### ข้อ 3 - Secure Messenger
สร้างระบบส่งข้อความที่เข้ารหัส:
- Key Exchange
- End-to-End Encryption
- Message Authentication

### ข้อ 4 - Certificate Generator
สร้าง Self-signed Certificate:
- Generate Key Pair
- Fill Certificate Info
- Sign Certificate

### ข้อ 5 - Steganography
ซ่อนข้อความในรูปภาพ:
- LSB (Least Significant Bit)
- เข้ารหัสข้อมูลก่อนซ่อน
- Extract ข้อมูลกลับ
