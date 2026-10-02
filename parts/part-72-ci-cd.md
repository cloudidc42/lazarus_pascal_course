# ตอนที่ 72: CI/CD Pipeline ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการตั้งค่า CI/CD pipeline สำหรับโปรเจ็กต์ Pascal/Lazarus ด้วย GitHub Actions และ GitLab CI

---

## 72.1 GitHub Actions

```yaml
# .github/workflows/build.yml
name: Build and Test Pascal App

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  FPC_VERSION: "3.2.2"
  LAZARUS_VERSION: "2.2.6"

jobs:
  # Job 1: Build on Linux
  build-linux:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Install Free Pascal Compiler
        run: |
          sudo apt-get update
          sudo apt-get install -y fpc
          fpc -iV
          
      - name: Install Lazarus (headless)
        run: |
          sudo apt-get install -y lazarus-ide lcl-utils
          
      - name: Build project
        run: |
          cd src
          fpc -O2 main.pas -o ../bin/myapp
          
      - name: Run unit tests
        run: |
          cd tests
          fpc -O2 run_tests.pas -o ../bin/run_tests
          ../bin/run_tests
          
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: linux-binary
          path: bin/myapp
          
  # Job 2: Build on Windows
  build-windows:
    runs-on: windows-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Install FPC
        run: |
          choco install freepascal -y
          
      - name: Build project
        run: |
          fpc -O2 src\main.pas -o bin\myapp.exe
          
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: windows-binary
          path: bin/myapp.exe
          
  # Job 3: Code Quality
  code-quality:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install FPC
        run: sudo apt-get install -y fpc
        
      - name: Check compilation warnings
        run: |
          fpc -O2 -Vw src/main.pas 2>&1 | tee compile.log
          
          # ตรวจสอบ warnings
          if grep -q "Warning:" compile.log; then
            echo "Found compilation warnings!"
            cat compile.log
            exit 1
          fi
          
      - name: Check code style
        run: |
          # Custom style checker
          python3 scripts/check_style.py src/
          
  # Job 4: Docker Build
  docker-build:
    runs-on: ubuntu-latest
    needs: [build-linux, code-quality]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
          
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            myorg/myapp:latest
            myorg/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          
  # Job 5: Deploy to Staging
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [docker-build]
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /app
            docker-compose pull
            docker-compose up -d
            docker-compose ps
            
      - name: Run smoke tests
        run: |
          sleep 30
          curl -f https://staging.example.com/health
          
  # Job 6: Deploy to Production
  deploy-production:
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://www.example.com
    
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /app
            docker-compose pull
            docker-compose up -d --no-downtime
            
      - name: Verify deployment
        run: |
          sleep 60
          curl -f https://www.example.com/health
          echo "Deployment successful!"
```

---

## 72.2 GitLab CI

```yaml
# .gitlab-ci.yml
image: debian:bookworm-slim

stages:
  - install
  - build
  - test
  - package
  - deploy

variables:
  FPC_PATH: /usr/bin/fpc
  BUILD_DIR: build
  BIN_DIR: bin

# Cache FPC installation
cache:
  key: fpc-deps
  paths:
    - /usr/lib/fpc/
    - /usr/bin/fpc

# Install FPC
install:
  stage: install
  script:
    - apt-get update && apt-get install -y fpc libssl-dev
    - fpc -iV
  only:
    - schedules

# Build Linux binary
build:linux:
  stage: build
  before_script:
    - apt-get update && apt-get install -y fpc
  script:
    - mkdir -p $BUILD_DIR $BIN_DIR
    - fpc -O2 -dRELEASE src/main.pas -o $BIN_DIR/myapp
    - ls -la $BIN_DIR/
  artifacts:
    paths:
      - $BIN_DIR/
    expire_in: 1 week

# Build Windows binary
build:windows:
  stage: build
  before_script:
    - apt-get update && apt-get install -y fpc mingw-w64
  script:
    - mkdir -p $BIN_DIR
    # Cross-compile for Windows
    - fpc -O2 -dRELEASE -Tcrossi386-win32 src/main.pas -o $BIN_DIR/myapp.exe
  artifacts:
    paths:
      - $BIN_DIR/myapp.exe
    expire_in: 1 week

# Unit tests
test:unit:
  stage: test
  needs: [build:linux]
  before_script:
    - apt-get update && apt-get install -y fpc
  script:
    - fpc -O2 tests/test_suite.pas -o bin/test_suite
    - ./bin/test_suite --format=junit --output=test-results.xml
  artifacts:
    reports:
      junit: test-results.xml
    when: always

# Integration tests  
test:integration:
  stage: test
  needs: [test:unit]
  services:
    - postgres:15
    - redis:7
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgres://test:testpass@postgres/testdb"
    REDIS_URL: "redis://redis:6379"
  script:
    - apt-get update && apt-get install -y fpc postgresql-client
    - ./bin/myapp --integration-test

# Code coverage
coverage:
  stage: test
  needs: [build:linux]
  script:
    - fpc -O2 -pg tests/coverage_test.pas -o bin/coverage_test
    - ./bin/coverage_test
    - gprof bin/coverage_test gmon.out > coverage_report.txt
    - cat coverage_report.txt
  artifacts:
    paths:
      - coverage_report.txt
  coverage: '/Lines executed:\s+(\d+\.\d+)%/'

# Docker image
package:docker:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  needs: [test:unit, test:integration]
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

# Deploy staging
deploy:staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - apt-get update && apt-get install -y openssh-client
    - eval $(ssh-agent -s)
    - echo "$STAGING_SSH_KEY" | tr -d '\r' | ssh-add -
    - ssh -o StrictHostKeyChecking=no deploy@staging.example.com
        "cd /app && docker-compose pull && docker-compose up -d"
    - sleep 30
    - curl -f https://staging.example.com/health
  only:
    - develop

# Deploy production
deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://www.example.com
  script:
    - apt-get update && apt-get install -y openssh-client
    - eval $(ssh-agent -s)
    - echo "$PROD_SSH_KEY" | tr -d '\r' | ssh-add -
    - ssh -o StrictHostKeyChecking=no deploy@prod.example.com
        "cd /app && ./deploy.sh $CI_COMMIT_SHA"
  when: manual
  only:
    - main
```

---

## 72.3 Pascal Build Script

```pascal
program build_system;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Process;

type
  TBuildTarget = (btDebug, btRelease, btTest);
  
  TBuildConfig = record
    SourceDir: string;
    OutputDir: string;
    MainFile: string;
    OutputName: string;
    Target: TBuildTarget;
    Defines: TStringList;
    IncludeDirs: TStringList;
    LibraryDirs: TStringList;
    AdditionalOptions: string;
  end;

  TBuildResult = record
    Success: Boolean;
    Output: string;
    Errors: string;
    Warnings: Integer;
    Errors_Count: Integer;
    BuildTime: TDateTime;
  end;

  TPascalBuilder = class
  private
    FConfig: TBuildConfig;
    
    function BuildCommand: string;
    function ParseOutput(const AOutput: string; out AResult: TBuildResult): Boolean;
    procedure EnsureOutputDir;
    
  public
    constructor Create(const AConfig: TBuildConfig);
    destructor Destroy; override;
    
    function Build: TBuildResult;
    function Clean: Boolean;
    function Test: TBuildResult;
    
    class function DefaultDebugConfig(const ASource, AOutput, AMain: string): TBuildConfig;
    class function DefaultReleaseConfig(const ASource, AOutput, AMain: string): TBuildConfig;
  end;

  // Test runner
  TTestRunner = class
  private
    FTestDir: string;
    FResults: TStringList;
    
  public
    constructor Create(const ATestDir: string);
    destructor Destroy; override;
    
    function RunAll: Boolean;
    function RunFile(const ATestFile: string): Boolean;
    procedure PrintReport;
    function GetJUnitXML: string;
  end;

implementation

class function TPascalBuilder.DefaultDebugConfig(const ASource, AOutput, AMain: string): TBuildConfig;
begin
  Result.SourceDir := ASource;
  Result.OutputDir := AOutput;
  Result.MainFile := AMain;
  Result.OutputName := ChangeFileExt(ExtractFileName(AMain), '');
  Result.Target := btDebug;
  Result.Defines := TStringList.Create;
  Result.Defines.Add('DEBUG');
  Result.Defines.Add('ASSERTIONS');
  Result.IncludeDirs := TStringList.Create;
  Result.LibraryDirs := TStringList.Create;
  Result.AdditionalOptions := '-g -gl -gw2';  // Debug info, line info
end;

class function TPascalBuilder.DefaultReleaseConfig(const ASource, AOutput, AMain: string): TBuildConfig;
begin
  Result.SourceDir := ASource;
  Result.OutputDir := AOutput;
  Result.MainFile := AMain;
  Result.OutputName := ChangeFileExt(ExtractFileName(AMain), '');
  Result.Target := btRelease;
  Result.Defines := TStringList.Create;
  Result.Defines.Add('RELEASE');
  Result.IncludeDirs := TStringList.Create;
  Result.LibraryDirs := TStringList.Create;
  Result.AdditionalOptions := '-O3 -Xs';  // Optimized, strip symbols
end;

constructor TPascalBuilder.Create(const AConfig: TBuildConfig);
begin
  inherited Create;
  FConfig := AConfig;
end;

destructor TPascalBuilder.Destroy;
begin
  FConfig.Defines.Free;
  FConfig.IncludeDirs.Free;
  FConfig.LibraryDirs.Free;
  inherited Destroy;
end;

procedure TPascalBuilder.EnsureOutputDir;
begin
  if not DirectoryExists(FConfig.OutputDir) then
    CreateDir(FConfig.OutputDir);
end;

function TPascalBuilder.BuildCommand: string;
var
  i: Integer;
  OutputPath: string;
begin
  OutputPath := FConfig.OutputDir + DirectorySeparator + FConfig.OutputName;
  
  {$IFDEF WINDOWS}
  OutputPath := OutputPath + '.exe';
  {$ENDIF}
  
  Result := 'fpc ';
  
  // Optimization
  case FConfig.Target of
    btDebug:   Result := Result + '-O1 ';
    btRelease: Result := Result + '-O3 ';
    btTest:    Result := Result + '-O1 ';
  end;
  
  // Mode
  Result := Result + '-Mobjfpc -Sh ';
  
  // Defines
  for i := 0 to FConfig.Defines.Count - 1 do
    Result := Result + '-d' + FConfig.Defines[i] + ' ';
    
  // Include dirs
  for i := 0 to FConfig.IncludeDirs.Count - 1 do
    Result := Result + '-I' + FConfig.IncludeDirs[i] + ' ';
    
  // Library dirs
  for i := 0 to FConfig.LibraryDirs.Count - 1 do
    Result := Result + '-Fu' + FConfig.LibraryDirs[i] + ' ';
    
  // Output
  Result := Result + '-o' + OutputPath + ' ';
  
  // Additional options
  if FConfig.AdditionalOptions <> '' then
    Result := Result + FConfig.AdditionalOptions + ' ';
    
  // Main file
  Result := Result + FConfig.SourceDir + DirectorySeparator + FConfig.MainFile;
  
  WriteLn('Build command: ', Result);
end;

function TPascalBuilder.ParseOutput(const AOutput: string; 
  out AResult: TBuildResult): Boolean;
var
  Lines: TStringList;
  i: Integer;
  Line: string;
begin
  AResult.Warnings := 0;
  AResult.Errors_Count := 0;
  
  Lines := TStringList.Create;
  try
    Lines.Text := AOutput;
    
    for i := 0 to Lines.Count - 1 do
    begin
      Line := Lines[i];
      
      if Pos('Warning:', Line) > 0 then
        Inc(AResult.Warnings)
      else if Pos('Error:', Line) > 0 then
        Inc(AResult.Errors_Count)
      else if Pos('Fatal:', Line) > 0 then
        Inc(AResult.Errors_Count);
    end;
    
    AResult.Success := AResult.Errors_Count = 0;
  finally
    Lines.Free;
  end;
  
  Result := AResult.Success;
end;

function TPascalBuilder.Build: TBuildResult;
var
  Proc: TProcess;
  Stream: TMemoryStream;
  StartTime: TDateTime;
  Cmd: string;
begin
  FillChar(Result, SizeOf(Result), 0);
  StartTime := Now;
  
  EnsureOutputDir;
  Cmd := BuildCommand;
  
  WriteLn('Starting build...');
  
  Proc := TProcess.Create(nil);
  try
    Proc.CommandLine := Cmd;
    Proc.Options := [poUsePipes, poStdErrToOutput, poWaitOnExit];
    
    Stream := TMemoryStream.Create;
    try
      Proc.Execute;
      Stream.CopyFrom(Proc.Output, Proc.Output.NumBytesAvailable);
      
      SetLength(Result.Output, Stream.Size);
      Stream.Position := 0;
      Stream.Read(Result.Output[1], Stream.Size);
      
    finally
      Stream.Free;
    end;
    
    Result.Success := Proc.ExitCode = 0;
    
  finally
    Proc.Free;
  end;
  
  ParseOutput(Result.Output, Result);
  Result.BuildTime := Now - StartTime;
  
  if Result.Success then
    WriteLn(Format('Build SUCCESS in %.2f seconds (Warnings: %d)',
      [Result.BuildTime * 86400, Result.Warnings]))
  else
    WriteLn(Format('Build FAILED! Errors: %d', [Result.Errors_Count]));
    
  WriteLn(Result.Output);
end;

function TPascalBuilder.Clean: Boolean;
var
  SR: TSearchRec;
  FileCount: Integer;
begin
  Result := True;
  FileCount := 0;
  
  // ลบ compiled files
  if FindFirst(FConfig.OutputDir + '/*', faAnyFile, SR) = 0 then
  try
    repeat
      var FullPath := FConfig.OutputDir + DirectorySeparator + SR.Name;
      var Ext := LowerCase(ExtractFileExt(SR.Name));
      
      if Ext in ['.o', '.ppu', '.a', '.so', ''] then
      begin
        DeleteFile(FullPath);
        Inc(FileCount);
      end;
    until FindNext(SR) <> 0;
  finally
    FindClose(SR);
  end;
  
  WriteLn(Format('Cleaned %d files from %s', [FileCount, FConfig.OutputDir]));
end;

function TPascalBuilder.Test: TBuildResult;
var
  TestConfig: TBuildConfig;
  TestBuilder: TPascalBuilder;
begin
  // Build test binary
  TestConfig := DefaultDebugConfig(
    FConfig.SourceDir + '/tests',
    FConfig.OutputDir,
    'test_suite.pas'
  );
  TestConfig.OutputName := 'test_runner';
  
  TestBuilder := TPascalBuilder.Create(TestConfig);
  try
    Result := TestBuilder.Build;
    
    if Result.Success then
    begin
      WriteLn('Running tests...');
      // TODO: Execute test binary and capture results
    end;
  finally
    TestBuilder.Free;
  end;
end;

{ TTestRunner }

constructor TTestRunner.Create(const ATestDir: string);
begin
  inherited Create;
  FTestDir := ATestDir;
  FResults := TStringList.Create;
end;

destructor TTestRunner.Destroy;
begin
  FResults.Free;
  inherited Destroy;
end;

function TTestRunner.RunFile(const ATestFile: string): Boolean;
var
  Proc: TProcess;
  Output: string;
  Stream: TMemoryStream;
begin
  Result := False;
  
  Proc := TProcess.Create(nil);
  try
    Proc.CommandLine := ATestFile;
    Proc.Options := [poUsePipes, poStdErrToOutput, poWaitOnExit];
    
    Stream := TMemoryStream.Create;
    try
      Proc.Execute;
      Stream.CopyFrom(Proc.Output, Proc.Output.NumBytesAvailable);
      
      SetLength(Output, Stream.Size);
      if Stream.Size > 0 then
      begin
        Stream.Position := 0;
        Stream.Read(Output[1], Stream.Size);
      end;
    finally
      Stream.Free;
    end;
    
    Result := Proc.ExitCode = 0;
    FResults.Add(Format('%s:%s', [ATestFile, IfThen(Result, 'PASS', 'FAIL')]));
    
    WriteLn(Format('[%s] %s', [IfThen(Result, 'PASS', 'FAIL'), 
      ExtractFileName(ATestFile)]));
  finally
    Proc.Free;
  end;
end;

function TTestRunner.RunAll: Boolean;
var
  SR: TSearchRec;
  TestFile: string;
  AllPassed: Boolean;
begin
  AllPassed := True;
  
  if FindFirst(FTestDir + '/*_test', faAnyFile, SR) = 0 then
  try
    repeat
      TestFile := FTestDir + DirectorySeparator + SR.Name;
      if not RunFile(TestFile) then
        AllPassed := False;
    until FindNext(SR) <> 0;
  finally
    FindClose(SR);
  end;
  
  Result := AllPassed;
end;

procedure TTestRunner.PrintReport;
var
  Passed, Failed: Integer;
  i: Integer;
begin
  Passed := 0; Failed := 0;
  
  for i := 0 to FResults.Count - 1 do
  begin
    var Status := Copy(FResults.ValueFromIndex[i], 1, 4);
    if Status = 'PASS' then Inc(Passed)
    else Inc(Failed);
  end;
  
  WriteLn('');
  WriteLn('=== Test Report ===');
  WriteLn(Format('Passed: %d', [Passed]));
  WriteLn(Format('Failed: %d', [Failed]));
  WriteLn(Format('Total:  %d', [Passed + Failed]));
  
  if Failed > 0 then
  begin
    WriteLn('');
    WriteLn('Failed tests:');
    for i := 0 to FResults.Count - 1 do
      if Pos('FAIL', FResults[i]) > 0 then
        WriteLn('  - ', FResults.Names[i]);
  end;
end;

end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **GitHub Actions** - CI/CD pipeline บน GitHub
2. **GitLab CI** - Pipeline บน GitLab
3. **Multi-stage builds** - Build, Test, Package, Deploy
4. **Pascal Build System** - สร้าง build tool ด้วย Pascal
5. **Test Runner** - ระบบรัน tests อัตโนมัติ

CI/CD ช่วยให้ทีมสามารถ deploy code ได้อย่างปลอดภัยและรวดเร็ว โดยทุก push จะถูก build, test, และ deploy อัตโนมัติ ลดความผิดพลาดจากมนุษย์
