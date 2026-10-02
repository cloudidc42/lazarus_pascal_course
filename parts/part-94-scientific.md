# ตอนที่ 94: Scientific Computing กับ Pascal

## บทนำ: Pascal สำหรับ Scientific Applications

Pascal มี precision arithmetic ที่ดีเยี่ยม เหมาะสำหรับ Numerical Methods, Signal Processing และ Statistics

## 1. Matrix Operations

```pascal
// uMatrix.pas - Matrix Operations Library
unit uMatrix;

{$mode objfpc}{$H+}

interface

uses SysUtils, Classes, Math;

type
  TMatrix = class
  private
    FData: array of array of Double;
    FRows: Integer;
    FCols: Integer;

    function GetElement(ARow, ACol: Integer): Double; inline;
    procedure SetElement(ARow, ACol: Integer; AValue: Double); inline;

  public
    constructor Create(ARows, ACols: Integer);
    constructor CreateIdentity(ASize: Integer);
    constructor CreateFrom2D(const AData: array of array of Double);
    destructor Destroy; override;

    class function Multiply(const A, B: TMatrix): TMatrix;
    class function Add(const A, B: TMatrix): TMatrix;
    class function Subtract(const A, B: TMatrix): TMatrix;

    function Transpose: TMatrix;
    function Inverse: TMatrix;
    function Determinant: Double;
    function Trace: Double;
    function Norm(AOrder: Integer = 2): Double;

    // Decompositions
    procedure LUDecompose(out L, U: TMatrix);
    procedure QRDecompose(out Q, R: TMatrix);
    procedure SVDecompose(out U, S, V: TMatrix); // Singular Value Decomp

    // Linear System solving
    function SolveLinear(const B: TMatrix): TMatrix; // Ax = b

    // Eigenvalues (Power iteration)
    procedure PowerIteration(out AEigenvalue: Double;
      out AEigenvector: TMatrix;
      AMaxIter: Integer = 1000;
      ATolerance: Double = 1e-10);

    property Rows: Integer read FRows;
    property Cols: Integer read FCols;
    property Elements[ARow, ACol: Integer]: Double
      read GetElement write SetElement; default;
  end;

implementation

constructor TMatrix.Create(ARows, ACols: Integer);
begin
  inherited Create;
  FRows := ARows;
  FCols := ACols;
  SetLength(FData, ARows, ACols);
end;

constructor TMatrix.CreateIdentity(ASize: Integer);
var
  I: Integer;
begin
  Create(ASize, ASize);
  for I := 0 to ASize - 1 do
    FData[I][I] := 1.0;
end;

class function TMatrix.Multiply(const A, B: TMatrix): TMatrix;
var
  I, J, K: Integer;
  Sum: Double;
begin
  if A.FCols <> B.FRows then
    raise Exception.CreateFmt(
      'Matrix dimensions mismatch: %dx%d * %dx%d',
      [A.FRows, A.FCols, B.FRows, B.FCols]);

  Result := TMatrix.Create(A.FRows, B.FCols);
  for I := 0 to A.FRows - 1 do
    for J := 0 to B.FCols - 1 do
    begin
      Sum := 0;
      for K := 0 to A.FCols - 1 do
        Sum := Sum + A.FData[I][K] * B.FData[K][J];
      Result.FData[I][J] := Sum;
    end;
end;

function TMatrix.Determinant: Double;
var
  L, U: TMatrix;
  I: Integer;
  Sign: Integer;
begin
  if FRows <> FCols then
    raise Exception.Create('Determinant requires square matrix');

  if FRows = 1 then
  begin
    Result := FData[0][0];
    Exit;
  end;

  if FRows = 2 then
  begin
    Result := FData[0][0] * FData[1][1] -
              FData[0][1] * FData[1][0];
    Exit;
  end;

  // LU Decomposition method
  LUDecompose(L, U);
  try
    Result := 1.0;
    for I := 0 to FRows - 1 do
      Result := Result * U.FData[I][I];
  finally
    L.Free;
    U.Free;
  end;
end;

procedure TMatrix.LUDecompose(out L, U: TMatrix);
var
  N, I, J, K: Integer;
  Sum: Double;
begin
  N := FRows;
  L := TMatrix.CreateIdentity(N);
  U := TMatrix.Create(N, N);

  // Copy to U
  for I := 0 to N - 1 do
    for J := 0 to N - 1 do
      U.FData[I][J] := FData[I][J];

  for K := 0 to N - 2 do
  begin
    for I := K + 1 to N - 1 do
    begin
      if Abs(U.FData[K][K]) < 1e-15 then
        raise Exception.Create('Matrix is singular');

      L.FData[I][K] := U.FData[I][K] / U.FData[K][K];

      for J := K to N - 1 do
        U.FData[I][J] := U.FData[I][J] -
          L.FData[I][K] * U.FData[K][J];
    end;
  end;
end;

function TMatrix.SolveLinear(const B: TMatrix): TMatrix;
var
  L, U: TMatrix;
  N, I, J: Integer;
  Y, X: TMatrix;
  Sum: Double;
begin
  N := FRows;
  LUDecompose(L, U);
  try
    // Forward substitution: Ly = b
    Y := TMatrix.Create(N, 1);
    try
      for I := 0 to N - 1 do
      begin
        Sum := B.FData[I][0];
        for J := 0 to I - 1 do
          Sum := Sum - L.FData[I][J] * Y.FData[J][0];
        Y.FData[I][0] := Sum; // L diagonal is 1
      end;

      // Backward substitution: Ux = y
      Result := TMatrix.Create(N, 1);
      for I := N - 1 downto 0 do
      begin
        Sum := Y.FData[I][0];
        for J := I + 1 to N - 1 do
          Sum := Sum - U.FData[I][J] * Result.FData[J][0];
        Result.FData[I][0] := Sum / U.FData[I][I];
      end;
    finally
      Y.Free;
    end;
  finally
    L.Free;
    U.Free;
  end;
end;

end.
```

## 2. Fast Fourier Transform (FFT)

```pascal
// uFFT.pas - Fast Fourier Transform
unit uFFT;

{$mode objfpc}{$H+}

interface

uses SysUtils, Classes, Math;

type
  TComplex = record
    Re, Im: Double;
    class function Create(ARe, AIm: Double): TComplex; static;
    class operator Add(const A, B: TComplex): TComplex;
    class operator Subtract(const A, B: TComplex): TComplex;
    class operator Multiply(const A, B: TComplex): TComplex;
    function Magnitude: Double;
    function Phase: Double;
  end;

  TComplexArray = array of TComplex;

  // FFT Functions
  procedure FFT(var AData: TComplexArray; AInverse: Boolean = False);
  procedure RFFT(const AReal: array of Double;
    out AMagnitude, APhase: array of Double);
  function PowerSpectrum(const ASignal: array of Double): TArray<Double>;
  function PeakFrequency(const ASignal: array of Double;
    ASampleRate: Double): Double;

implementation

class function TComplex.Create(ARe, AIm: Double): TComplex;
begin
  Result.Re := ARe;
  Result.Im := AIm;
end;

class operator TComplex.Add(const A, B: TComplex): TComplex;
begin
  Result.Re := A.Re + B.Re;
  Result.Im := A.Im + B.Im;
end;

class operator TComplex.Multiply(const A, B: TComplex): TComplex;
begin
  Result.Re := A.Re * B.Re - A.Im * B.Im;
  Result.Im := A.Re * B.Im + A.Im * B.Re;
end;

function TComplex.Magnitude: Double;
begin
  Result := Sqrt(Re * Re + Im * Im);
end;

// Cooley-Tukey FFT (Radix-2 DIT)
procedure FFT(var AData: TComplexArray; AInverse: Boolean);
var
  N, Steps, Step, Half: Integer;
  Temp: TComplex;
  Angle: Double;
  W, WN: TComplex;
  I, J, K: Integer;
begin
  N := Length(AData);
  if N <= 1 then Exit;

  // Bit-reversal permutation
  J := 0;
  for I := 1 to N - 1 do
  begin
    var Bit := N shr 1;
    while J and Bit <> 0 do
    begin
      J := J xor Bit;
      Bit := Bit shr 1;
    end;
    J := J xor Bit;

    if I < J then
    begin
      Temp := AData[I];
      AData[I] := AData[J];
      AData[J] := Temp;
    end;
  end;

  // Butterfly operations
  Steps := 2;
  while Steps <= N do
  begin
    Half := Steps div 2;

    if AInverse then
      Angle := 2 * Pi / Steps
    else
      Angle := -2 * Pi / Steps;

    WN := TComplex.Create(Cos(Angle), Sin(Angle));

    I := 0;
    while I < N do
    begin
      W := TComplex.Create(1, 0);

      for J := 0 to Half - 1 do
      begin
        K := I + J + Half;
        Temp := TComplex.Create(
          W.Re * AData[K].Re - W.Im * AData[K].Im,
          W.Re * AData[K].Im + W.Im * AData[K].Re
        );

        AData[K] := AData[I + J] - Temp;
        AData[I + J] := AData[I + J] + Temp;

        W := W * WN;
      end;

      Inc(I, Steps);
    end;

    Steps := Steps * 2;
  end;

  // Scale for inverse FFT
  if AInverse then
    for I := 0 to N - 1 do
    begin
      AData[I].Re := AData[I].Re / N;
      AData[I].Im := AData[I].Im / N;
    end;
end;

function PowerSpectrum(const ASignal: array of Double): TArray<Double>;
var
  N, I: Integer;
  Data: TComplexArray;
begin
  N := Length(ASignal);
  SetLength(Data, N);

  for I := 0 to N - 1 do
  begin
    Data[I].Re := ASignal[I];
    Data[I].Im := 0;
  end;

  FFT(Data);

  SetLength(Result, N div 2);
  for I := 0 to (N div 2) - 1 do
    Result[I] := Data[I].Magnitude * Data[I].Magnitude / N;
end;

function PeakFrequency(const ASignal: array of Double;
  ASampleRate: Double): Double;
var
  Spectrum: TArray<Double>;
  MaxVal: Double;
  MaxIdx: Integer;
  I: Integer;
begin
  Spectrum := PowerSpectrum(ASignal);
  MaxVal := 0;
  MaxIdx := 0;

  for I := 1 to High(Spectrum) do // Skip DC component
  begin
    if Spectrum[I] > MaxVal then
    begin
      MaxVal := Spectrum[I];
      MaxIdx := I;
    end;
  end;

  Result := MaxIdx * ASampleRate / Length(ASignal);
end;

end.
```

## 3. Statistics Library

```pascal
// uStatistics.pas - Statistical Analysis
unit uStatistics;

{$mode objfpc}{$H+}

interface

uses SysUtils, Math;

type
  TStatResult = record
    Mean: Double;
    Median: Double;
    StdDev: Double;
    Variance: Double;
    Min, Max: Double;
    Range: Double;
    Q1, Q3: Double;  // Quartiles
    IQR: Double;     // Interquartile Range
    Skewness: Double;
    Kurtosis: Double;
    Count: Integer;
  end;

  TLinearRegression = record
    Slope: Double;
    Intercept: Double;
    RSquared: Double;
    RMSE: Double;
    Predict: function(X: Double): Double of object;
  end;

  // Descriptive Statistics
  function Describe(const AData: array of Double): TStatResult;
  function Mean(const AData: array of Double): Double;
  function Median(const AData: array of Double): Double;
  function Variance(const AData: array of Double): Double;
  function StdDev(const AData: array of Double): Double;
  function Percentile(const AData: array of Double; P: Double): Double;
  function ZScore(AValue, AMean, AStdDev: Double): Double;

  // Correlation
  function Pearson(const AX, AY: array of Double): Double;
  function Spearman(const AX, AY: array of Double): Double;

  // Regression
  function LinearRegress(const AX, AY: array of Double): TLinearRegression;

  // Hypothesis Testing
  function TTest(const AGroup1, AGroup2: array of Double;
    out APValue: Double): Double; // Returns t-statistic
  function ChiSquareTest(const AObserved,
    AExpected: array of Double): Double;

implementation

function Mean(const AData: array of Double): Double;
var
  Sum: Double;
  V: Double;
begin
  if Length(AData) = 0 then
  begin
    Result := 0;
    Exit;
  end;

  Sum := 0;
  for V in AData do
    Sum := Sum + V;
  Result := Sum / Length(AData);
end;

function Median(const AData: array of Double): Double;
var
  Sorted: TArray<Double>;
  N, Mid: Integer;
begin
  N := Length(AData);
  if N = 0 then
  begin
    Result := 0;
    Exit;
  end;

  Sorted := Copy(AData, 0, N);
  TArray.Sort<Double>(Sorted);

  Mid := N div 2;
  if N mod 2 = 0 then
    Result := (Sorted[Mid - 1] + Sorted[Mid]) / 2
  else
    Result := Sorted[Mid];
end;

function Variance(const AData: array of Double): Double;
var
  Avg: Double;
  SumSq: Double;
  V: Double;
  N: Integer;
begin
  N := Length(AData);
  if N < 2 then
  begin
    Result := 0;
    Exit;
  end;

  Avg := Mean(AData);
  SumSq := 0;
  for V in AData do
    SumSq := SumSq + Sqr(V - Avg);

  Result := SumSq / (N - 1); // Sample variance (Bessel's correction)
end;

function Describe(const AData: array of Double): TStatResult;
var
  Sorted: TArray<Double>;
  N, I: Integer;
  Avg: Double;
  SumCube, SumFour: Double;
  Diff: Double;
begin
  N := Length(AData);
  Result.Count := N;

  if N = 0 then Exit;

  Sorted := Copy(AData, 0, N);
  TArray.Sort<Double>(Sorted);

  Result.Min := Sorted[0];
  Result.Max := Sorted[N - 1];
  Result.Range := Result.Max - Result.Min;
  Result.Mean := Mean(AData);
  Result.Median := Median(AData);
  Result.Variance := Variance(AData);
  Result.StdDev := Sqrt(Result.Variance);

  // Quartiles
  Result.Q1 := Percentile(AData, 25);
  Result.Q3 := Percentile(AData, 75);
  Result.IQR := Result.Q3 - Result.Q1;

  // Skewness and Kurtosis
  Avg := Result.Mean;
  SumCube := 0;
  SumFour := 0;
  for var V in AData do
  begin
    Diff := V - Avg;
    SumCube := SumCube + Diff * Diff * Diff;
    SumFour := SumFour + Diff * Diff * Diff * Diff;
  end;

  if Result.StdDev > 0 then
  begin
    Result.Skewness := (SumCube / N) / Power(Result.StdDev, 3);
    Result.Kurtosis := (SumFour / N) / Power(Result.Variance, 2) - 3;
  end;
end;

function Pearson(const AX, AY: array of Double): Double;
var
  N, I: Integer;
  MeanX, MeanY: Double;
  Num, DenX, DenY: Double;
begin
  N := Length(AX);
  if (N <> Length(AY)) or (N < 2) then
  begin
    Result := 0;
    Exit;
  end;

  MeanX := Mean(AX);
  MeanY := Mean(AY);

  Num := 0;
  DenX := 0;
  DenY := 0;

  for I := 0 to N - 1 do
  begin
    Num := Num + (AX[I] - MeanX) * (AY[I] - MeanY);
    DenX := DenX + Sqr(AX[I] - MeanX);
    DenY := DenY + Sqr(AY[I] - MeanY);
  end;

  if (DenX = 0) or (DenY = 0) then
    Result := 0
  else
    Result := Num / Sqrt(DenX * DenY);
end;

function LinearRegress(const AX, AY: array of Double): TLinearRegression;
var
  N, I: Integer;
  MeanX, MeanY: Double;
  SumXY, SumXX: Double;
  SSRes, SSTot: Double;
begin
  N := Length(AX);
  MeanX := Mean(AX);
  MeanY := Mean(AY);

  SumXY := 0;
  SumXX := 0;
  for I := 0 to N - 1 do
  begin
    SumXY := SumXY + (AX[I] - MeanX) * (AY[I] - MeanY);
    SumXX := SumXX + Sqr(AX[I] - MeanX);
  end;

  if SumXX = 0 then
  begin
    Result.Slope := 0;
    Result.Intercept := MeanY;
  end
  else
  begin
    Result.Slope := SumXY / SumXX;
    Result.Intercept := MeanY - Result.Slope * MeanX;
  end;

  // Calculate R²
  SSRes := 0;
  SSTot := 0;
  for I := 0 to N - 1 do
  begin
    SSRes := SSRes + Sqr(AY[I] - (Result.Slope * AX[I] + Result.Intercept));
    SSTot := SSTot + Sqr(AY[I] - MeanY);
  end;

  if SSTot = 0 then
    Result.RSquared := 1
  else
    Result.RSquared := 1 - SSRes / SSTot;

  Result.RMSE := Sqrt(SSRes / N);
end;

end.
```

## 4. Numerical Methods

```pascal
// uNumerical.pas - Numerical Analysis Methods
unit uNumerical;

{$mode objfpc}{$H+}

interface

uses SysUtils, Math;

type
  TFunction1D = function(X: Double): Double;
  TFunction2D = function(X, Y: Double): Double;
  TDerivative = function(X: Double): Double;

  // Root Finding
  function Bisection(AFunc: TFunction1D;
    AA, AB, ATol: Double; AMaxIter: Integer = 100): Double;
  function NewtonRaphson(AFunc: TFunction1D;
    ADerivative: TDerivative;
    AX0, ATol: Double; AMaxIter: Integer = 100): Double;
  function Brent(AFunc: TFunction1D;
    AA, AB, ATol: Double): Double;

  // Integration
  function TrapezoidRule(AFunc: TFunction1D;
    AA, AB: Double; AN: Integer): Double;
  function SimpsonRule(AFunc: TFunction1D;
    AA, AB: Double; AN: Integer): Double; // N must be even
  function GaussLegendre(AFunc: TFunction1D;
    AA, AB: Double; AOrder: Integer = 5): Double;

  // Differentiation
  function ForwardDiff(AFunc: TFunction1D;
    AX, AH: Double): Double;
  function CentralDiff(AFunc: TFunction1D;
    AX, AH: Double): Double;

  // ODE Solvers
  type
    TODEFunc = function(T, Y: Double): Double; // dy/dt = f(t, y)

  function RK4(AFunc: TODEFunc; AY0, AT0, AT1, AH: Double): Double;
  function RK45Adaptive(AFunc: TODEFunc;
    AY0, AT0, AT1, ATol: Double): TArray<Double>;

implementation

function Bisection(AFunc: TFunction1D; AA, AB, ATol: Double;
  AMaxIter: Integer): Double;
var
  FA, FB, FC: Double;
  C: Double;
  I: Integer;
begin
  FA := AFunc(AA);
  FB := AFunc(AB);

  if FA * FB > 0 then
    raise Exception.Create(
      'Bisection: f(a) and f(b) must have opposite signs');

  for I := 0 to AMaxIter - 1 do
  begin
    C := (AA + AB) / 2;
    FC := AFunc(C);

    if (Abs(FC) < ATol) or ((AB - AA) / 2 < ATol) then
    begin
      Result := C;
      Exit;
    end;

    if FA * FC > 0 then
    begin
      AA := C;
      FA := FC;
    end
    else
      AB := C;
  end;

  Result := (AA + AB) / 2;
end;

function NewtonRaphson(AFunc: TFunction1D; ADerivative: TDerivative;
  AX0, ATol: Double; AMaxIter: Integer): Double;
var
  X, FX, DFX: Double;
  I: Integer;
begin
  X := AX0;

  for I := 0 to AMaxIter - 1 do
  begin
    FX := AFunc(X);
    DFX := ADerivative(X);

    if Abs(DFX) < 1e-15 then
      raise Exception.Create('Newton-Raphson: derivative is zero');

    X := X - FX / DFX;

    if Abs(FX) < ATol then Break;
  end;

  Result := X;
end;

function SimpsonRule(AFunc: TFunction1D; AA, AB: Double; AN: Integer): Double;
var
  H, X: Double;
  I: Integer;
  Sum: Double;
begin
  if AN mod 2 <> 0 then
    raise Exception.Create('Simpson rule: N must be even');

  H := (AB - AA) / AN;
  Sum := AFunc(AA) + AFunc(AB);

  for I := 1 to AN - 1 do
  begin
    X := AA + I * H;
    if I mod 2 = 0 then
      Sum := Sum + 2 * AFunc(X)
    else
      Sum := Sum + 4 * AFunc(X);
  end;

  Result := Sum * H / 3;
end;

function RK4(AFunc: TODEFunc; AY0, AT0, AT1, AH: Double): Double;
var
  T, Y: Double;
  K1, K2, K3, K4: Double;
begin
  T := AT0;
  Y := AY0;

  while T < AT1 do
  begin
    if T + AH > AT1 then
      AH := AT1 - T;

    K1 := AH * AFunc(T, Y);
    K2 := AH * AFunc(T + AH / 2, Y + K1 / 2);
    K3 := AH * AFunc(T + AH / 2, Y + K2 / 2);
    K4 := AH * AFunc(T + AH, Y + K3);

    Y := Y + (K1 + 2 * K2 + 2 * K3 + K4) / 6;
    T := T + AH;
  end;

  Result := Y;
end;

end.
```

## 5. Signal Processor Application

```pascal
// SignalProcessor/SignalProcessor.pas - Complete Signal Processing App
program SignalProcessor;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, Math,
  uFFT, uStatistics, uNumerical;

procedure AnalyzeSignal(const ASignal: TArray<Double>;
  ASampleRate: Double);
var
  Stats: TStatResult;
  Spectrum: TArray<Double>;
  PeakHz: Double;
  I: Integer;
begin
  WriteLn('=== Signal Analysis ===');
  WriteLn(Format('Samples: %d, Sample Rate: %.0f Hz',
    [Length(ASignal), ASampleRate]));

  // Statistical analysis
  Stats := Describe(ASignal);
  WriteLn(Format('Mean: %.4f', [Stats.Mean]));
  WriteLn(Format('StdDev: %.4f', [Stats.StdDev]));
  WriteLn(Format('Min: %.4f, Max: %.4f', [Stats.Min, Stats.Max]));
  WriteLn(Format('Skewness: %.4f, Kurtosis: %.4f',
    [Stats.Skewness, Stats.Kurtosis]));

  // Frequency analysis
  PeakHz := PeakFrequency(ASignal, ASampleRate);
  WriteLn(Format('Peak Frequency: %.2f Hz', [PeakHz]));

  // Power spectrum (first 10 bins)
  Spectrum := PowerSpectrum(ASignal);
  WriteLn('Power Spectrum (first 10 bins):');
  for I := 0 to Min(9, High(Spectrum)) do
  begin
    var Freq := I * ASampleRate / Length(ASignal);
    WriteLn(Format('  %.1f Hz: %.6f', [Freq, Spectrum[I]]));
  end;
end;

function GenerateTestSignal(ASamples: Integer;
  AFreq1, AFreq2, ASampleRate: Double): TArray<Double>;
var
  I: Integer;
begin
  SetLength(Result, ASamples);
  for I := 0 to ASamples - 1 do
  begin
    var T := I / ASampleRate;
    // Composite signal: 50Hz + 120Hz + noise
    Result[I] := Sin(2 * Pi * AFreq1 * T) +
                 0.5 * Sin(2 * Pi * AFreq2 * T) +
                 (Random - 0.5) * 0.1;
  end;
end;

procedure DemoNumericalMethods;
var
  Root: Double;
begin
  WriteLn(#13#10'=== Numerical Methods Demo ===');

  // Find root of x^3 - x - 2 = 0
  Root := Bisection(
    function(X: Double): Double
    begin
      Result := X * X * X - X - 2;
    end,
    1.0, 2.0, 1e-10
  );
  WriteLn(Format('Root of x³-x-2=0: x = %.10f', [Root]));

  // Integration of sin(x) from 0 to Pi = 2
  var Integral := SimpsonRule(
    function(X: Double): Double
    begin
      Result := Sin(X);
    end,
    0, Pi, 1000
  );
  WriteLn(Format('∫sin(x)dx [0,π] = %.10f (exact: 2.0)', [Integral]));

  // ODE: dy/dt = -y, y(0) = 1 → y(t) = e^(-t)
  var Y := RK4(
    function(T, Y: Double): Double
    begin
      Result := -Y;
    end,
    1.0, 0, 1.0, 0.01
  );
  WriteLn(Format('ODE y''=-y, y(1): %.6f (exact: %.6f)',
    [Y, Exp(-1)]));
end;

var
  Signal: TArray<Double>;
begin
  Randomize;

  WriteLn('Scientific Computing Demo - Pascal');
  WriteLn('===================================');

  // Generate and analyze signal
  Signal := GenerateTestSignal(1024, 50, 120, 1000);
  AnalyzeSignal(Signal, 1000);

  DemoNumericalMethods;

  WriteLn(#13#10'Press Enter to exit...');
  ReadLn;
end.
```

## 6. สรุป Scientific Computing

**Libraries ที่ควรใช้:**
1. **FPMath** - FPC Math functions ต่างๆ
2. **GSL (GNU Scientific Library)** - เรียกผ่าน binding
3. **LAPACK/BLAS** - Linear Algebra
4. **gnuplot** - Plotting via pipe
5. **HDF5** - Scientific data storage

**Use Cases:**
- Signal Processing (Audio, RF, Biomedical)
- Structural Engineering FEM
- Financial Modeling (Monte Carlo)
- Climate/Weather Simulation
- Physics Simulation
- Machine Learning (Gradient Descent)

**Performance Tips:**
1. ใช้ Extended (80-bit) precision ถ้าจำเป็น
2. SIMD instructions ผ่าน ASM สำหรับ Vector operations
3. Parallel computation ด้วย OpenMP ผ่าน Threading
4. Cache-friendly data layout (Row-major vs Column-major)
