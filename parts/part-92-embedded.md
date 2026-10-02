# ตอนที่ 92: Embedded Systems กับ Free Pascal

## บทนำ: FPC สำหรับ Embedded Systems

Free Pascal รองรับการ Compile สำหรับ ARM, AVR และ Embedded Targets ต่างๆ

## 1. FPC สำหรับ AVR (Arduino)

```pascal
// AVR/blink.pas - LED Blink บน ATmega328P
program Blink;

{$mode objfpc}

uses
  AVR;

const
  LED_PIN = 5; // PB5 = Arduino Digital Pin 13

procedure delay_ms(ms: Word);
var
  I, J: Word;
begin
  for I := 0 to ms - 1 do
    for J := 0 to 1999 do
      asm nop end;
end;

begin
  // Set PB5 as output
  DDRB := DDRB or (1 shl LED_PIN);

  repeat
    // Turn LED on
    PORTB := PORTB or (1 shl LED_PIN);
    delay_ms(500);

    // Turn LED off
    PORTB := PORTB and not (1 shl LED_PIN);
    delay_ms(500);
  until False;
end.
```

## 2. AVR UART Communication

```pascal
// AVR/uart.pas - UART สำหรับ ATmega328P
unit UART;

{$mode objfpc}

interface

procedure UART_Init(BaudRate: LongWord);
procedure UART_SendChar(C: Char);
procedure UART_SendStr(const S: string);
function UART_RecvChar: Char;
function UART_Available: Boolean;

implementation

uses AVR;

const
  F_CPU = 16000000; // 16 MHz

procedure UART_Init(BaudRate: LongWord);
var
  UBRRVal: Word;
begin
  UBRRVal := F_CPU div (16 * BaudRate) - 1;

  UBRR0H := Hi(UBRRVal);
  UBRR0L := Lo(UBRRVal);

  // Enable TX and RX
  UCSR0B := (1 shl RXEN0) or (1 shl TXEN0);

  // 8 data bits, 1 stop bit, no parity
  UCSR0C := (1 shl UCSZ01) or (1 shl UCSZ00);
end;

procedure UART_SendChar(C: Char);
begin
  // Wait for empty transmit buffer
  while (UCSR0A and (1 shl UDRE0)) = 0 do;
  UDR0 := Byte(C);
end;

procedure UART_SendStr(const S: string);
var
  I: Integer;
begin
  for I := 1 to Length(S) do
    UART_SendChar(S[I]);
end;

function UART_RecvChar: Char;
begin
  // Wait for data
  while (UCSR0A and (1 shl RXC0)) = 0 do;
  Result := Char(UDR0);
end;

function UART_Available: Boolean;
begin
  Result := (UCSR0A and (1 shl RXC0)) <> 0;
end;

end.
```

## 3. AVR I2C (TWI) Driver

```pascal
// AVR/i2c_driver.pas - I2C Driver สำหรับ AVR
unit I2CDriver;

{$mode objfpc}

interface

const
  TWI_FREQ = 100000; // 100kHz Standard mode
  F_CPU    = 16000000;

type
  TI2CStatus = (i2cOK, i2cArbitLost, i2cNACK, i2cTimeout);

procedure I2C_Init;
function I2C_Start(Address: Byte; ReadWrite: Byte): TI2CStatus;
procedure I2C_Stop;
function I2C_Write(Data: Byte): TI2CStatus;
function I2C_ReadACK: Byte;
function I2C_ReadNACK: Byte;

implementation

uses AVR;

const
  TWI_READ  = 1;
  TWI_WRITE = 0;

procedure I2C_Init;
var
  Prescaler: Byte;
  TWBR_Val: Byte;
begin
  // Calculate TWBR
  Prescaler := 1;
  TWBR_Val := ((F_CPU div TWI_FREQ) - 16) div (2 * Prescaler);

  TWBR := TWBR_Val;
  TWSR := 0; // Prescaler = 1
  TWCR := (1 shl TWEN); // Enable TWI
end;

function I2C_Start(Address: Byte; ReadWrite: Byte): TI2CStatus;
var
  Status: Byte;
begin
  // Send START condition
  TWCR := (1 shl TWINT) or (1 shl TWSTA) or (1 shl TWEN);

  // Wait for TWINT flag
  while (TWCR and (1 shl TWINT)) = 0 do;

  // Check START status
  Status := TWSR and $F8;
  if (Status <> $08) and (Status <> $10) then
  begin
    Result := i2cArbitLost;
    Exit;
  end;

  // Send address + R/W bit
  TWDR := (Address shl 1) or ReadWrite;
  TWCR := (1 shl TWINT) or (1 shl TWEN);

  while (TWCR and (1 shl TWINT)) = 0 do;

  Status := TWSR and $F8;
  if ReadWrite = TWI_WRITE then
  begin
    if Status <> $18 then // SLA+W ACK
    begin
      Result := i2cNACK;
      Exit;
    end;
  end
  else
  begin
    if Status <> $40 then // SLA+R ACK
    begin
      Result := i2cNACK;
      Exit;
    end;
  end;

  Result := i2cOK;
end;

procedure I2C_Stop;
begin
  TWCR := (1 shl TWINT) or (1 shl TWSTO) or (1 shl TWEN);
  while (TWCR and (1 shl TWSTO)) <> 0 do;
end;

function I2C_Write(Data: Byte): TI2CStatus;
begin
  TWDR := Data;
  TWCR := (1 shl TWINT) or (1 shl TWEN);
  while (TWCR and (1 shl TWINT)) = 0 do;

  if (TWSR and $F8) = $28 then // Data transmitted, ACK received
    Result := i2cOK
  else
    Result := i2cNACK;
end;

function I2C_ReadACK: Byte;
begin
  // Enable ACK
  TWCR := (1 shl TWINT) or (1 shl TWEN) or (1 shl TWEA);
  while (TWCR and (1 shl TWINT)) = 0 do;
  Result := TWDR;
end;

function I2C_ReadNACK: Byte;
begin
  // No ACK (last byte)
  TWCR := (1 shl TWINT) or (1 shl TWEN);
  while (TWCR and (1 shl TWINT)) = 0 do;
  Result := TWDR;
end;

end.
```

## 4. BME280 Sensor Driver (I2C)

```pascal
// AVR/bme280.pas - BME280 Temperature/Humidity/Pressure Sensor
unit BME280;

{$mode objfpc}

interface

uses I2CDriver;

const
  BME280_ADDR = $76; // or $77

type
  TBME280Reading = record
    Temperature: LongInt;  // °C * 100 (avoid float on AVR)
    Humidity:    LongInt;  // % * 1024
    Pressure:    LongWord; // Pa * 256
  end;

function BME280_Init: Boolean;
function BME280_Read(out Reading: TBME280Reading): Boolean;
procedure BME280_FormatTemp(const Reading: TBME280Reading;
  out Str: string);

implementation

type
  TBME280Calib = record
    // Temperature calibration
    T1: Word;
    T2: SmallInt;
    T3: SmallInt;
    // Pressure calibration
    P1: Word;
    P2, P3, P4, P5, P6, P7, P8, P9: SmallInt;
    // Humidity calibration
    H1: Byte;
    H2: SmallInt;
    H3: Byte;
    H4, H5: SmallInt;
    H6: ShortInt;
  end;

var
  GCalib: TBME280Calib;

function BME280_ReadReg(Reg: Byte): Byte;
begin
  I2C_Start(BME280_ADDR, 0); // Write
  I2C_Write(Reg);
  I2C_Start(BME280_ADDR, 1); // Read
  Result := I2C_ReadNACK;
  I2C_Stop;
end;

procedure BME280_WriteReg(Reg, Value: Byte);
begin
  I2C_Start(BME280_ADDR, 0);
  I2C_Write(Reg);
  I2C_Write(Value);
  I2C_Stop;
end;

function BME280_Init: Boolean;
var
  ChipId: Byte;
begin
  Result := False;
  ChipId := BME280_ReadReg($D0); // Chip ID register

  if ChipId <> $60 then Exit; // Not BME280

  // Read calibration data
  I2C_Start(BME280_ADDR, 0);
  I2C_Write($88); // Start of calibration registers
  I2C_Start(BME280_ADDR, 1);

  GCalib.T1 := I2C_ReadACK or (Word(I2C_ReadACK) shl 8);
  GCalib.T2 := SmallInt(I2C_ReadACK or (Word(I2C_ReadACK) shl 8));
  GCalib.T3 := SmallInt(I2C_ReadACK or (Word(I2C_ReadACK) shl 8));
  // ... (read all 26 calibration bytes)
  I2C_ReadNACK; // Last byte without ACK
  I2C_Stop;

  // Configure: Normal mode, 1x oversampling
  BME280_WriteReg($F2, $01); // Humidity oversampling x1
  BME280_WriteReg($F4, $27); // Temp x1, Press x1, Normal mode
  BME280_WriteReg($F5, $A0); // 1000ms standby, filter off

  Result := True;
end;

function BME280_Read(out Reading: TBME280Reading): Boolean;
var
  RawData: array[0..7] of Byte;
  RawPress, RawTemp, RawHum: LongInt;
  VarT1, VarT2, TFine: LongInt;
  I: Integer;
begin
  Result := False;

  // Read raw data burst
  I2C_Start(BME280_ADDR, 0);
  I2C_Write($F7); // Start of data registers
  I2C_Start(BME280_ADDR, 1);

  for I := 0 to 6 do
    RawData[I] := I2C_ReadACK;
  RawData[7] := I2C_ReadNACK;
  I2C_Stop;

  // Parse raw values
  RawPress := (LongInt(RawData[0]) shl 12) or
              (LongInt(RawData[1]) shl 4) or
              (RawData[2] shr 4);
  RawTemp  := (LongInt(RawData[3]) shl 12) or
              (LongInt(RawData[4]) shl 4) or
              (RawData[5] shr 4);
  RawHum   := (LongInt(RawData[6]) shl 8) or RawData[7];

  // Temperature compensation (BME280 datasheet algorithm)
  VarT1 := ((RawTemp shr 3) - (LongInt(GCalib.T1) shl 1));
  VarT1 := (VarT1 * LongInt(GCalib.T2)) shr 11;

  VarT2 := ((RawTemp shr 4) - LongInt(GCalib.T1));
  VarT2 := ((VarT2 * VarT2) shr 12) * LongInt(GCalib.T3) shr 14;

  TFine := VarT1 + VarT2;
  Reading.Temperature := (TFine * 5 + 128) shr 8; // in 0.01 °C

  Result := True;
end;

procedure BME280_FormatTemp(const Reading: TBME280Reading;
  out Str: string);
var
  IntPart, FracPart: Integer;
begin
  IntPart := Reading.Temperature div 100;
  FracPart := Abs(Reading.Temperature mod 100);
  Str := IntToStr(IntPart) + '.' + Format('%.2d', [FracPart]) + ' C';
end;

end.
```

## 5. Interrupt Handling บน AVR

```pascal
// AVR/interrupts_demo.pas - Interrupt Handler
program InterruptDemo;

{$mode objfpc}

uses AVR;

// Timer1 Overflow Interrupt Handler
procedure Timer1_OVF_ISR; interrupt; public name 'TIMER1_OVF';
var
  I: Byte;
begin
  I := PORTB;
  PORTB := I xor (1 shl 5); // Toggle LED
end;

// External Interrupt INT0
procedure INT0_ISR; interrupt; public name 'INT0';
begin
  // Button pressed
  PORTB := PORTB or (1 shl 4); // Signal LED on
end;

begin
  // Configure LED pins as output
  DDRB := (1 shl 5) or (1 shl 4);

  // Configure Timer1 for 1Hz blink
  // CTC mode, prescaler 1024
  TCCR1A := 0;
  TCCR1B := (1 shl WGM12) or (1 shl CS12) or (1 shl CS10);
  OCR1A  := 15624; // (16MHz / 1024 / 1Hz) - 1

  // Enable Timer1 Output Compare A interrupt
  TIMSK1 := (1 shl OCIE1A);

  // Configure INT0 (falling edge)
  EICRA := (1 shl ISC01); // Falling edge
  EIMSK := (1 shl INT0);  // Enable INT0

  // Enable global interrupts
  asm sei end;

  repeat
    // Main loop - interrupts handle everything
    asm nop end;
  until False;
end.
```

## 6. ARM Cortex-M (STM32) กับ FPC

```pascal
// STM32/gpio_stm32.pas - GPIO สำหรับ STM32F103
unit GPIO_STM32;

{$mode objfpc}

interface

const
  // STM32F103 Base Addresses
  RCC_BASE   = $40021000;
  GPIOA_BASE = $40010800;
  GPIOB_BASE = $40010C00;
  GPIOC_BASE = $40011000;

  // RCC registers
  RCC_APB2ENR = RCC_BASE + $18;

  // GPIO register offsets
  GPIO_CRL  = 0;
  GPIO_CRH  = 4;
  GPIO_IDR  = 8;
  GPIO_ODR  = 12;
  GPIO_BSRR = 16;
  GPIO_BRR  = 20;

type
  TGPIOPort = (gpA, gpB, gpC);
  TGPIOMode = (gmInput = 0, gmOutput10MHz = 1,
               gmOutput2MHz = 2, gmOutput50MHz = 3);
  TGPIOConfig = (gcInputAnalog = 0, gcInputFloat = 1,
                 gcInputPullUpDown = 2,
                 gcOutputPP = 0, gcOutputOD = 1,
                 gcAFPP = 2, gcAFOD = 3);

procedure GPIO_ClockEnable(APort: TGPIOPort);
procedure GPIO_SetMode(APort: TGPIOPort; APin: Byte;
  AMode: TGPIOMode; AConfig: TGPIOConfig);
procedure GPIO_SetPin(APort: TGPIOPort; APin: Byte);
procedure GPIO_ResetPin(APort: TGPIOPort; APin: Byte);
function GPIO_ReadPin(APort: TGPIOPort; APin: Byte): Boolean;
procedure GPIO_TogglePin(APort: TGPIOPort; APin: Byte);

implementation

const
  PortBases: array[TGPIOPort] of LongWord =
    (GPIOA_BASE, GPIOB_BASE, GPIOC_BASE);

  RCC_Bits: array[TGPIOPort] of LongWord =
    (1 shl 2, 1 shl 3, 1 shl 4); // IOPAEN, IOPBEN, IOPCEN

function RegAt(ABase, AOffset: LongWord): ^LongWord;
begin
  Result := Pointer(ABase + AOffset);
end;

procedure GPIO_ClockEnable(APort: TGPIOPort);
begin
  RegAt(RCC_APB2ENR, 0)^ :=
    RegAt(RCC_APB2ENR, 0)^ or RCC_Bits[APort];
end;

procedure GPIO_SetMode(APort: TGPIOPort; APin: Byte;
  AMode: TGPIOMode; AConfig: TGPIOConfig);
var
  Base: LongWord;
  RegOffset: LongWord;
  BitOffset: Byte;
  Reg: ^LongWord;
  ModeVal: LongWord;
begin
  Base := PortBases[APort];

  if APin < 8 then
  begin
    RegOffset := GPIO_CRL;
    BitOffset := APin * 4;
  end
  else
  begin
    RegOffset := GPIO_CRH;
    BitOffset := (APin - 8) * 4;
  end;

  Reg := RegAt(Base, RegOffset);
  ModeVal := Ord(AMode) or (Ord(AConfig) shl 2);

  // Clear bits
  Reg^ := Reg^ and not (LongWord($F) shl BitOffset);
  // Set bits
  Reg^ := Reg^ or (ModeVal shl BitOffset);
end;

procedure GPIO_SetPin(APort: TGPIOPort; APin: Byte);
begin
  RegAt(PortBases[APort], GPIO_BSRR)^ := 1 shl APin;
end;

procedure GPIO_ResetPin(APort: TGPIOPort; APin: Byte);
begin
  RegAt(PortBases[APort], GPIO_BRR)^ := 1 shl APin;
end;

function GPIO_ReadPin(APort: TGPIOPort; APin: Byte): Boolean;
begin
  Result := (RegAt(PortBases[APort], GPIO_IDR)^ and (1 shl APin)) <> 0;
end;

procedure GPIO_TogglePin(APort: TGPIOPort; APin: Byte);
var
  ODR: ^LongWord;
begin
  ODR := RegAt(PortBases[APort], GPIO_ODR);
  ODR^ := ODR^ xor (1 shl APin);
end;

end.
```

## 7. Building สำหรับ Embedded Targets

```bash
# Compile สำหรับ AVR (ATmega328P)
fpc -Pavr \
    -Cpavr5 \
    -Oatmega328p \
    -Tembedded \
    -Xd \
    -CX \
    -XX \
    -O3 \
    blink.pas

# Compile สำหรับ ARM Cortex-M3 (STM32F103)
fpc -Parm \
    -Tcpuarmv7m \
    -Tembedded \
    -CfVFPV3 \
    -Xd \
    -CX \
    -XX \
    -O3 \
    stm32_demo.pas

# Convert ELF to HEX สำหรับ Flash
arm-none-eabi-objcopy -O ihex stm32_demo stm32_demo.hex
avr-objcopy -O ihex blink blink.hex

# Upload ไปยัง Arduino
avrdude -p atmega328p \
        -c arduino \
        -P /dev/ttyUSB0 \
        -b 115200 \
        -U flash:w:blink.hex
```

## 8. Memory Management บน Embedded

```pascal
// AVR/memory_opt.pas - Memory Optimization
unit MemoryOpt;

{$mode objfpc}

interface

// Static Memory Pool สำหรับ Embedded (ไม่ใช้ Heap)
type
  TStaticPool = object
  private
    FBuffer: array[0..511] of Byte; // 512 bytes pool
    FAllocated: Word;
  public
    procedure Init;
    function Alloc(ASize: Word): Pointer;
    procedure Reset;
  end;

// String in Flash Memory (PROGMEM)
// บน AVR ใช้ string constants จาก program memory
const
  MSG_HELLO: string = 'Hello, Embedded World!'; // Stored in Flash

// EEPROM Storage
procedure EEPROM_Write(AAddr: Word; AData: Byte);
function EEPROM_Read(AAddr: Word): Byte;
procedure EEPROM_WriteStr(AAddr: Word; const AStr: string);

implementation

uses AVR;

procedure TStaticPool.Init;
begin
  FAllocated := 0;
end;

function TStaticPool.Alloc(ASize: Word): Pointer;
begin
  if FAllocated + ASize > SizeOf(FBuffer) then
  begin
    Result := nil; // Out of memory
    Exit;
  end;

  Result := @FBuffer[FAllocated];
  Inc(FAllocated, ASize);
end;

procedure EEPROM_Write(AAddr: Word; AData: Byte);
begin
  // Wait for completion
  while (EECR and (1 shl EEPE)) <> 0 do;

  EEAR := AAddr;
  EEDR := AData;

  // Write enable
  asm cli end;
  EECR := EECR or (1 shl EEMPE);
  EECR := EECR or (1 shl EEPE);
  asm sei end;
end;

function EEPROM_Read(AAddr: Word): Byte;
begin
  while (EECR and (1 shl EEPE)) <> 0 do;
  EEAR := AAddr;
  EECR := EECR or (1 shl EERE);
  Result := EEDR;
end;

end.
```

## 9. สรุป Embedded Pascal

**FPC Target Support:**
- `avr` - ATmega chips (Arduino)
- `arm` - ARM Cortex-M (STM32, LPC)
- `i8086` - DOS/BIOS programming
- `mipsel` - MIPS embedded

**Compiler Flags สำคัญ:**
- `-Tembedded` - Embedded target (no OS)
- `-Xd` - Disable default libraries
- `-CX` - Smart linking
- `-XX` - Use external linker
- `-O3` - Maximum optimization
- `-Cp<cpu>` - Target CPU subtype

**Embedded Best Practices:**
1. ใช้ Static allocation แทน Heap
2. หลีกเลี่ยง exceptions ใน ISR
3. ใช้ `volatile` สำหรับ hardware registers
4. Minimize stack usage
5. ใช้ `const` arrays ใน Flash memory (AVR)
6. Profile code size ด้วย `avr-size`
