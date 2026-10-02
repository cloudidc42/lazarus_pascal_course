# ตอนที่ 89: DevOps Practices กับ Pascal/Lazarus

## บทนำ: DevOps คืออะไร?

DevOps คือการรวม Development (Dev) และ Operations (Ops) เข้าด้วยกัน เพื่อให้การ Deploy และ Operate ซอฟต์แวร์เป็นไปอย่างรวดเร็ว เชื่อถือได้ และปลอดภัย

## 1. CI/CD Pipeline

```
Developer Commits
      │
      ▼
┌──────────────────────────────────────────────┐
│           CI Pipeline (GitHub Actions)       │
│                                              │
│  1. Checkout Code                            │
│  2. Build (FPC Compile)                      │
│  3. Run Unit Tests                           │
│  4. Code Quality Check                       │
│  5. Security Scan                            │
│  6. Build Docker Image                       │
│  7. Push to Registry                         │
└──────────────────────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────────────┐
│           CD Pipeline (ArgoCD/Flux)          │
│                                              │
│  Staging Deploy → Integration Tests          │
│  Performance Tests → Manual Approval         │
│  Production Deploy → Health Check            │
└──────────────────────────────────────────────┘
```

## 2. Dockerfile for Pascal Applications

```dockerfile
# Dockerfile สำหรับ Pascal/Lazarus Application

# Stage 1: Build
FROM freepascal/fpc:3.2.2-full AS builder

WORKDIR /app

# Copy source files
COPY src/ ./src/
COPY *.lpi ./
COPY *.lpr ./

# Install dependencies
RUN apt-get update && apt-get install -y \
    libssl-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Compile
RUN fpc -O2 -Xs -XX -CX \
    -Fi./src \
    -Fu./src \
    ./MyApp.lpr \
    -o./bin/myapp

# Stage 2: Runtime (minimal image)
FROM debian:bullseye-slim

WORKDIR /app

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    libssl1.1 \
    libpq5 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy binary from builder
COPY --from=builder /app/bin/myapp ./myapp

# Copy config templates
COPY config/ ./config/

# Non-root user for security
RUN useradd -r -u 1001 -g root appuser
RUN chown -R appuser:root /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080

ENTRYPOINT ["./myapp"]
```

## 3. GitHub Actions CI/CD

```yaml
# .github/workflows/ci-cd.yml
# (YAML สำหรับ Reference - ไม่ใช่ Pascal code)

# name: CI/CD Pipeline
# on:
#   push:
#     branches: [main, develop]
#   pull_request:
#     branches: [main]
#
# env:
#   REGISTRY: ghcr.io
#   IMAGE_NAME: mycompany/myapp
#
# jobs:
#   build-and-test:
#     runs-on: ubuntu-latest
#     container:
#       image: freepascal/fpc:3.2.2-full
#     steps:
#       - uses: actions/checkout@v3
#       - name: Build
#         run: fpc -O2 ./MyApp.lpr
#       - name: Run Tests
#         run: ./run_tests.sh
#
#   docker-build:
#     needs: build-and-test
#     runs-on: ubuntu-latest
#     steps:
#       - name: Build and push Docker image
#         uses: docker/build-push-action@v4
```

## 4. Deployment Scripts

```pascal
// DeployTool/uDeployment.pas
unit uDeployment;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Process, fphttpclient, fpjson;

type
  TDeploymentConfig = record
    Environment: string;
    ServiceName: string;
    ImageTag: string;
    Replicas: Integer;
    K8sNamespace: string;
    RollingUpdate: Boolean;
    HealthCheckUrl: string;
    WaitTimeoutSeconds: Integer;
  end;

  TDeploymentResult = record
    Success: Boolean;
    Message: string;
    DeployedAt: TDateTime;
    RolloutDuration: Integer; // seconds
  end;

  TDeployer = class
  private
    FConfig: TDeploymentConfig;
    FLogger: ILogger;
    FKubeconfigPath: string;

    function RunCommand(const ACommand: string;
      out AOutput: string): Integer;
    function WaitForHealthy(ATimeoutSeconds: Integer): Boolean;
    function GetCurrentReplicas: Integer;
    procedure TakeSnapshotForRollback;

  public
    constructor Create(const AConfig: TDeploymentConfig;
      const AKubeconfigPath: string = '');

    function Deploy: TDeploymentResult;
    function Rollback(const AToVersion: string): TDeploymentResult;
    function GetDeploymentStatus: string;
    function Scale(AReplicas: Integer): Boolean;
  end;

  // Blue-Green Deployment
  TBlueGreenDeployer = class
  private
    FServiceName: string;
    FCurrentColor: string; // 'blue' or 'green'
    FNewColor: string;
    FLoadBalancer: TLoadBalancerClient;
    FLogger: ILogger;

    function GetCurrentActiveColor: string;
    procedure DeployToInactiveSlot(const AImageTag: string);
    procedure RunSmokeTests: Boolean;
    procedure SwitchTraffic;
    procedure RollbackToOldColor;

  public
    constructor Create(const AServiceName: string;
      const ALoadBalancer: TLoadBalancerClient);

    function Deploy(const AImageTag: string): TDeploymentResult;
  end;

  // Canary Deployment
  TCanaryDeployer = class
  private
    FServiceName: string;
    FBasePercentage: Integer;
    FRampUpIntervalSeconds: Integer;
    FRampUpSteps: Integer;
    FLogger: ILogger;
    FMetrics: TAppMetrics;

    function GetErrorRate(const AVersion: string): Double;
    function GetLatency(const AVersion: string): Double;
    function IsCanaryHealthy: Boolean;
    procedure IncreaseCanaryTraffic(APercentage: Integer);
    procedure RollbackCanary;
    procedure FullyPromoteCanary;

  public
    constructor Create(const AServiceName: string;
      ABasePercentage: Integer = 5;
      ARampUpIntervalSeconds: Integer = 300;
      ARampUpSteps: Integer = 5);

    function Deploy(const ACanaryImageTag: string): TDeploymentResult;
  end;

implementation

function TDeployer.RunCommand(const ACommand: string;
  out AOutput: string): Integer;
var
  AProcess: TProcess;
  OutputLines: TStringList;
begin
  AProcess := TProcess.Create(nil);
  OutputLines := TStringList.Create;
  try
    AProcess.Options := [poUsePipes, poWaitOnExit];
    AProcess.CommandLine := ACommand;
    AProcess.Execute;

    var OutputStream := TStringStream.Create;
    try
      OutputStream.CopyFrom(AProcess.Output, AProcess.Output.NumBytesAvailable);
      AOutput := OutputStream.DataString;
    finally
      OutputStream.Free;
    end;

    Result := AProcess.ExitStatus;
  finally
    OutputLines.Free;
    AProcess.Free;
  end;
end;

function TDeployer.Deploy: TDeploymentResult;
var
  Output: string;
  ExitCode: Integer;
  StartTime: TDateTime;
begin
  Result.Success := False;
  Result.DeployedAt := Now;
  StartTime := Now;

  FLogger.Info(Format('Deploying %s:%s to %s',
    [FConfig.ServiceName, FConfig.ImageTag, FConfig.Environment]));

  // 1. Take snapshot for rollback
  TakeSnapshotForRollback;

  // 2. Apply Kubernetes deployment
  var KubectlCmd := Format(
    'kubectl set image deployment/%s %s=%s:%s -n %s',
    [FConfig.ServiceName, FConfig.ServiceName,
     FConfig.ImageTag, FConfig.ImageTag,
     FConfig.K8sNamespace]);

  ExitCode := RunCommand(KubectlCmd, Output);
  if ExitCode <> 0 then
  begin
    Result.Message := 'Failed to set image: ' + Output;
    FLogger.Error('Deployment failed', nil);
    Exit;
  end;

  // 3. Wait for rollout
  var RolloutCmd := Format(
    'kubectl rollout status deployment/%s -n %s --timeout=%ds',
    [FConfig.ServiceName, FConfig.K8sNamespace,
     FConfig.WaitTimeoutSeconds]);

  ExitCode := RunCommand(RolloutCmd, Output);
  if ExitCode <> 0 then
  begin
    Result.Message := 'Rollout failed: ' + Output;
    FLogger.Error('Rollout failed, rolling back...');
    Rollback(''); // Rollback to previous
    Exit;
  end;

  // 4. Health Check
  if FConfig.HealthCheckUrl <> '' then
  begin
    if not WaitForHealthy(60) then
    begin
      Result.Message := 'Health check failed after deployment';
      Rollback('');
      Exit;
    end;
  end;

  Result.Success := True;
  Result.Message := 'Deployment successful';
  Result.RolloutDuration := Round((Now - StartTime) * SecsPerDay);

  FLogger.Info(Format('Deployment completed in %ds', [Result.RolloutDuration]));
end;

function TDeployer.WaitForHealthy(ATimeoutSeconds: Integer): Boolean;
var
  StartTime: TDateTime;
  Client: TFPHTTPClient;
  Response: TStringStream;
  ElapsedSeconds: Integer;
begin
  Result := False;
  StartTime := Now;

  FLogger.Info('Waiting for health check: ' + FConfig.HealthCheckUrl);

  Client := TFPHTTPClient.Create(nil);
  Response := TStringStream.Create;
  try
    Client.ConnectTimeout := 5000;
    Client.IOTimeout := 5000;

    repeat
      try
        Response.Clear;
        Client.Get(FConfig.HealthCheckUrl, Response);

        if Client.ResponseStatusCode = 200 then
        begin
          FLogger.Info('Health check passed');
          Result := True;
          Exit;
        end;
      except
        on E: Exception do
          FLogger.Debug('Health check not ready: ' + E.Message);
      end;

      Sleep(5000); // Wait 5 seconds before retry
      ElapsedSeconds := Round((Now - StartTime) * SecsPerDay);
    until ElapsedSeconds >= ATimeoutSeconds;

    FLogger.Warning(Format('Health check timed out after %ds', [ATimeoutSeconds]));
  finally
    Response.Free;
    Client.Free;
  end;
end;

function TDeployer.Rollback(const AToVersion: string): TDeploymentResult;
var
  Output: string;
  ExitCode: Integer;
  Command: string;
begin
  FLogger.Warning(Format('Rolling back %s...', [FConfig.ServiceName]));

  if AToVersion <> '' then
    Command := Format(
      'kubectl set image deployment/%s %s=%s:%s -n %s',
      [FConfig.ServiceName, FConfig.ServiceName,
       FConfig.ServiceName, AToVersion,
       FConfig.K8sNamespace])
  else
    Command := Format(
      'kubectl rollout undo deployment/%s -n %s',
      [FConfig.ServiceName, FConfig.K8sNamespace]);

  ExitCode := RunCommand(Command, Output);
  Result.Success := ExitCode = 0;
  Result.Message := IfThen(Result.Success, 'Rollback successful', 'Rollback failed: ' + Output);
  Result.DeployedAt := Now;

  FLogger.Info(IfThen(Result.Success, 'Rollback successful', 'Rollback failed'));
end;

// Blue-Green Deployer
function TBlueGreenDeployer.Deploy(const AImageTag: string): TDeploymentResult;
var
  StartTime: TDateTime;
begin
  Result.Success := False;
  StartTime := Now;

  // 1. Determine current and new slot
  FCurrentColor := GetCurrentActiveColor;
  FNewColor := IfThen(FCurrentColor = 'blue', 'green', 'blue');

  FLogger.Info(Format('Blue-Green: Current=%s, New=%s',
    [FCurrentColor, FNewColor]));

  // 2. Deploy to inactive slot
  DeployToInactiveSlot(AImageTag);

  // 3. Run Smoke Tests
  if not RunSmokeTests then
  begin
    Result.Message := 'Smoke tests failed on ' + FNewColor + ' slot';
    FLogger.Error(Result.Message);
    Exit;
  end;

  // 4. Switch traffic (instant!)
  SwitchTraffic;

  Result.Success := True;
  Result.Message := Format('Traffic switched to %s (tag: %s)', [FNewColor, AImageTag]);
  Result.RolloutDuration := Round((Now - StartTime) * SecsPerDay);
end;

// Canary Deployer
function TCanaryDeployer.Deploy(const ACanaryImageTag: string): TDeploymentResult;
var
  CurrentPercentage: Integer;
  StepSize: Integer;
  I: Integer;
begin
  Result.Success := False;

  FLogger.Info(Format('Canary deployment: %s at %d%%',
    [ACanaryImageTag, FBasePercentage]));

  // 1. Start with base percentage
  CurrentPercentage := FBasePercentage;
  IncreaseCanaryTraffic(CurrentPercentage);

  // 2. Ramp up gradually
  StepSize := (100 - FBasePercentage) div FRampUpSteps;

  for I := 1 to FRampUpSteps do
  begin
    // Wait between steps
    Sleep(FRampUpIntervalSeconds * 1000);

    // Check canary health
    if not IsCanaryHealthy then
    begin
      FLogger.Warning('Canary unhealthy, rolling back');
      RollbackCanary;
      Result.Message := 'Canary deployment aborted: unhealthy metrics';
      Exit;
    end;

    // Increase traffic
    CurrentPercentage := Min(100, CurrentPercentage + StepSize);
    IncreaseCanaryTraffic(CurrentPercentage);
    FLogger.Info(Format('Canary at %d%%', [CurrentPercentage]));
  end;

  // 3. Full promotion
  FullyPromoteCanary;

  Result.Success := True;
  Result.Message := 'Canary successfully promoted to 100%';
end;

function TCanaryDeployer.IsCanaryHealthy: Boolean;
var
  ErrorRate: Double;
  Latency: Double;
begin
  ErrorRate := GetErrorRate('canary');
  Latency := GetLatency('canary');

  FLogger.Debug(Format('Canary metrics: error_rate=%.2f%%, latency=%.1fms',
    [ErrorRate * 100, Latency]));

  // Compare with baseline
  var BaselineErrorRate := GetErrorRate('stable');
  var BaselineLatency := GetLatency('stable');

  // Allow up to 2x error rate increase or 50% latency increase
  Result := (ErrorRate <= BaselineErrorRate * 2.0) and
             (Latency <= BaselineLatency * 1.5);
end;

end.
```

## 5. Infrastructure as Code

```pascal
// IaC/uTerraformRunner.pas - Terraform Integration
unit uTerraformRunner;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Process, fpjson;

type
  TTerraformAction = (taInit, taPlan, taApply, taDestroy, taOutput, taState);

  TTerraformConfig = record
    WorkDir: string;
    BackendConfig: TStringList;
    Variables: TStringList;
    AutoApprove: Boolean;
  end;

  TTerraformRunner = class
  private
    FConfig: TTerraformConfig;
    FLogger: ILogger;

    function RunTerraform(const AArgs: TArray<string>;
      out AOutput, AError: string): Integer;
    procedure SetupEnvironment;

  public
    constructor Create(const AConfig: TTerraformConfig);

    function Init: Boolean;
    function Plan(out APlanOutput: string): Boolean;
    function Apply: Boolean;
    function Destroy: Boolean;
    function GetOutput(const AOutputName: string): string;
    function GetState: TJSONObject;
    function Import(const AResourceAddr, AResourceId: string): Boolean;
  end;

  // Kubernetes Manifests Generator
  TK8sManifestGenerator = class
  public
    class function GenerateDeployment(
      const AServiceName, AImage, ANamespace: string;
      AReplicas: Integer;
      const AEnvVars: TStringList = nil): string;

    class function GenerateService(
      const AServiceName, ANamespace: string;
      APort, ATargetPort: Integer;
      const AServiceType: string = 'ClusterIP'): string;

    class function GenerateConfigMap(
      const AName, ANamespace: string;
      const AData: TStringList): string;

    class function GenerateHPA(
      const AServiceName, ANamespace: string;
      AMinReplicas, AMaxReplicas: Integer;
      ACpuThreshold: Integer = 70): string;
  end;

implementation

class function TK8sManifestGenerator.GenerateDeployment(
  const AServiceName, AImage, ANamespace: string;
  AReplicas: Integer; const AEnvVars: TStringList): string;
var
  Lines: TStringList;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('apiVersion: apps/v1');
    Lines.Add('kind: Deployment');
    Lines.Add('metadata:');
    Lines.Add('  name: ' + AServiceName);
    Lines.Add('  namespace: ' + ANamespace);
    Lines.Add('  labels:');
    Lines.Add('    app: ' + AServiceName);
    Lines.Add('spec:');
    Lines.Add('  replicas: ' + IntToStr(AReplicas));
    Lines.Add('  selector:');
    Lines.Add('    matchLabels:');
    Lines.Add('      app: ' + AServiceName);
    Lines.Add('  template:');
    Lines.Add('    metadata:');
    Lines.Add('      labels:');
    Lines.Add('        app: ' + AServiceName);
    Lines.Add('    spec:');
    Lines.Add('      containers:');
    Lines.Add('      - name: ' + AServiceName);
    Lines.Add('        image: ' + AImage);
    Lines.Add('        ports:');
    Lines.Add('        - containerPort: 8080');
    Lines.Add('        resources:');
    Lines.Add('          requests:');
    Lines.Add('            memory: "128Mi"');
    Lines.Add('            cpu: "100m"');
    Lines.Add('          limits:');
    Lines.Add('            memory: "512Mi"');
    Lines.Add('            cpu: "500m"');
    Lines.Add('        livenessProbe:');
    Lines.Add('          httpGet:');
    Lines.Add('            path: /health/live');
    Lines.Add('            port: 8080');
    Lines.Add('          initialDelaySeconds: 10');
    Lines.Add('          periodSeconds: 30');
    Lines.Add('        readinessProbe:');
    Lines.Add('          httpGet:');
    Lines.Add('            path: /health/ready');
    Lines.Add('            port: 8080');
    Lines.Add('          initialDelaySeconds: 5');
    Lines.Add('          periodSeconds: 10');

    if Assigned(AEnvVars) and (AEnvVars.Count > 0) then
    begin
      Lines.Add('        env:');
      for var I := 0 to AEnvVars.Count - 1 do
      begin
        Lines.Add('        - name: ' + AEnvVars.Names[I]);
        Lines.Add('          value: "' + AEnvVars.ValueFromIndex[I] + '"');
      end;
    end;

    Lines.Add('      imagePullSecrets:');
    Lines.Add('      - name: registry-secret');

    Result := Lines.Text;
  finally
    Lines.Free;
  end;
end;

class function TK8sManifestGenerator.GenerateHPA(
  const AServiceName, ANamespace: string;
  AMinReplicas, AMaxReplicas, ACpuThreshold: Integer): string;
begin
  Result :=
    'apiVersion: autoscaling/v2' + #13#10 +
    'kind: HorizontalPodAutoscaler' + #13#10 +
    'metadata:' + #13#10 +
    '  name: ' + AServiceName + '-hpa' + #13#10 +
    '  namespace: ' + ANamespace + #13#10 +
    'spec:' + #13#10 +
    '  scaleTargetRef:' + #13#10 +
    '    apiVersion: apps/v1' + #13#10 +
    '    kind: Deployment' + #13#10 +
    '    name: ' + AServiceName + #13#10 +
    '  minReplicas: ' + IntToStr(AMinReplicas) + #13#10 +
    '  maxReplicas: ' + IntToStr(AMaxReplicas) + #13#10 +
    '  metrics:' + #13#10 +
    '  - type: Resource' + #13#10 +
    '    resource:' + #13#10 +
    '      name: cpu' + #13#10 +
    '      target:' + #13#10 +
    '        type: Utilization' + #13#10 +
    '        averageUtilization: ' + IntToStr(ACpuThreshold);
end;

end.
```

## 6. Environment Management

```pascal
// uEnvironmentManager.pas - Environment Variables & Secrets
unit uEnvironmentManager;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson;

type
  TSecretProvider = (spEnvironment, spVaultHashicorp, spAwsSecrets,
    spAzureKeyVault, spK8sSecrets);

  ISecretManager = interface
    function GetSecret(const APath: string): string;
    function GetSecrets(const APaths: TArray<string>): TStringList;
    procedure SetSecret(const APath, AValue: string);
    function SecretExists(const APath: string): Boolean;
    procedure RotateSecret(const APath: string);
  end;

  // HashiCorp Vault Secret Manager
  TVaultSecretManager = class(TInterfacedObject, ISecretManager)
  private
    FVaultUrl: string;
    FToken: string;
    FMountPath: string;
    FHttpClient: TFPHTTPClient;
    FCachedSecrets: TDictionary<string, string>;
    FCacheTtl: Integer;
    FCacheTimestamps: TDictionary<string, TDateTime>;

    function GetHeaders: TStringList;
    function GetFromCache(const APath: string): string;
    procedure SetInCache(const APath, AValue: string);

  public
    constructor Create(const AVaultUrl, AToken: string;
      const AMountPath: string = 'secret';
      ACacheTtl: Integer = 300);
    destructor Destroy; override;

    function GetSecret(const APath: string): string;
    function GetSecrets(const APaths: TArray<string>): TStringList;
    procedure SetSecret(const APath, AValue: string);
    function SecretExists(const APath: string): Boolean;
    procedure RotateSecret(const APath: string);
  end;

  // Environment Variable Configuration
  TEnvConfig = class
  private
    FRequired: TList<string>;
    FDefaults: TStringList;
    FValidators: TDictionary<string, TFunc<string, Boolean>>;
    FLogger: ILogger;

    procedure ValidateAll;

  public
    constructor Create;
    destructor Destroy; override;

    function Require(const AKey: string): TEnvConfig;
    function WithDefault(const AKey, ADefault: string): TEnvConfig;
    function WithValidator(const AKey: string;
      const AValidator: TFunc<string, Boolean>): TEnvConfig;

    procedure Load;
    function Get(const AKey: string;
      const ADefault: string = ''): string;
    function GetInt(const AKey: string; ADefault: Integer = 0): Integer;
    function GetBool(const AKey: string; ADefault: Boolean = False): Boolean;
    function GetDuration(const AKey: string;
      ADefaultSeconds: Integer = 0): Integer;
  end;

implementation

function TVaultSecretManager.GetSecret(const APath: string): string;
var
  CachedValue: string;
  Response: TStringStream;
  Json: TJSONObject;
  DataObj: TJSONObject;
  FullPath: string;
begin
  // Check cache
  CachedValue := GetFromCache(APath);
  if CachedValue <> '' then
  begin
    Result := CachedValue;
    Exit;
  end;

  FullPath := Format('%s/v1/%s/data/%s', [FVaultUrl, FMountPath, APath]);

  Response := TStringStream.Create;
  try
    try
      FHttpClient.AddHeader('X-Vault-Token', FToken);
      FHttpClient.Get(FullPath, Response);

      Json := TJSONObject(GetJSON(Response.DataString));
      try
        DataObj := TJSONObject(Json.Find('data'));
        if Assigned(DataObj) then
        begin
          var ValueObj := TJSONObject(DataObj.Find('data'));
          if Assigned(ValueObj) then
            Result := ValueObj.Get('value', '');
        end;

        SetInCache(APath, Result);
      finally
        Json.Free;
      end;
    except
      on E: Exception do
      begin
        FLogger.Error('Failed to get secret from Vault: ' + APath, E);
        Result := '';
      end;
    end;
  finally
    Response.Free;
  end;
end;

// Environment Config
procedure TEnvConfig.Load;
begin
  ValidateAll;
end;

procedure TEnvConfig.ValidateAll;
var
  Key: string;
  Value: string;
  Validator: TFunc<string, Boolean>;
  Errors: TStringList;
begin
  Errors := TStringList.Create;
  try
    // Check required vars
    for Key in FRequired do
    begin
      Value := GetEnvironmentVariable(Key);
      if Value = '' then
        Errors.Add('Missing required environment variable: ' + Key);
    end;

    // Run validators
    for var KV in FValidators do
    begin
      Value := Get(KV.Key);
      if not KV.Value(Value) then
        Errors.Add(Format('Invalid value for %s: "%s"', [KV.Key, Value]));
    end;

    if Errors.Count > 0 then
      raise EConfigurationException.Create(
        'Configuration errors:' + #13#10 + Errors.Text);
  finally
    Errors.Free;
  end;
end;

function TEnvConfig.Get(const AKey: string; const ADefault: string): string;
begin
  Result := GetEnvironmentVariable(AKey);
  if (Result = '') and FDefaults.IndexOfName(AKey) >= 0 then
    Result := FDefaults.Values[AKey];
  if Result = '' then
    Result := ADefault;
end;

function TEnvConfig.GetInt(const AKey: string; ADefault: Integer): Integer;
begin
  Result := StrToIntDef(Get(AKey, IntToStr(ADefault)), ADefault);
end;

function TEnvConfig.GetBool(const AKey: string; ADefault: Boolean): Boolean;
var
  Val: string;
begin
  Val := LowerCase(Get(AKey, IfThen(ADefault, 'true', 'false')));
  Result := (Val = 'true') or (Val = '1') or (Val = 'yes');
end;

end.
```

## 7. Deployment Pipeline Example

```pascal
// deploy.pas - Deployment CLI Tool
program Deploy;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes,
  uDeployment, uEnvironmentManager, uLogger;

var
  Config: TDeploymentConfig;
  Deployer: TDeployer;
  Result: TDeploymentResult;
  Logger: ILogger;
  EnvConfig: TEnvConfig;

begin
  Logger := TLogger.Create('deploy-tool');
  Logger.AddSink(TConsoleSink.Create);

  // Load and validate environment configuration
  EnvConfig := TEnvConfig.Create;
  try
    EnvConfig
      .Require('DEPLOY_ENVIRONMENT')
      .Require('DEPLOY_SERVICE')
      .Require('DEPLOY_IMAGE_TAG')
      .WithDefault('DEPLOY_REPLICAS', '3')
      .WithDefault('K8S_NAMESPACE', 'production')
      .WithDefault('DEPLOY_TIMEOUT', '300')
      .WithValidator('DEPLOY_ENVIRONMENT',
        function(AVal: string): Boolean
        begin
          Result := AVal in ['staging', 'production', 'dr'];
        end
      );
    EnvConfig.Load;
  except
    on E: Exception do
    begin
      Logger.Error('Configuration error', E);
      WriteLn('ERROR: ', E.Message);
      ExitCode := 1;
      Exit;
    end;
  end;

  // Configure deployment
  Config.Environment := EnvConfig.Get('DEPLOY_ENVIRONMENT');
  Config.ServiceName := EnvConfig.Get('DEPLOY_SERVICE');
  Config.ImageTag := EnvConfig.Get('DEPLOY_IMAGE_TAG');
  Config.Replicas := EnvConfig.GetInt('DEPLOY_REPLICAS', 3);
  Config.K8sNamespace := EnvConfig.Get('K8S_NAMESPACE');
  Config.WaitTimeoutSeconds := EnvConfig.GetInt('DEPLOY_TIMEOUT', 300);
  Config.HealthCheckUrl := Format('http://%s.%s.svc.cluster.local/health',
    [Config.ServiceName, Config.K8sNamespace]);

  Logger.Info('=== Deployment Started ===');
  Logger.Info(Format('Service: %s', [Config.ServiceName]));
  Logger.Info(Format('Image: %s', [Config.ImageTag]));
  Logger.Info(Format('Environment: %s', [Config.Environment]));
  Logger.Info(Format('Replicas: %d', [Config.Replicas]));

  // Extra confirmation for production
  if Config.Environment = 'production' then
  begin
    Write('Deploying to PRODUCTION. Type "yes" to confirm: ');
    var Confirmation: string;
    ReadLn(Confirmation);
    if Confirmation <> 'yes' then
    begin
      Logger.Info('Deployment cancelled by user');
      ExitCode := 0;
      Exit;
    end;
  end;

  // Execute deployment
  Deployer := TDeployer.Create(Config);
  try
    Result := Deployer.Deploy;

    if Result.Success then
    begin
      Logger.Info(Format('Deployment successful in %ds', [Result.RolloutDuration]));
      WriteLn('SUCCESS: ', Result.Message);
      ExitCode := 0;
    end
    else
    begin
      Logger.Error('Deployment failed: ' + Result.Message);
      WriteLn('FAILURE: ', Result.Message);
      ExitCode := 1;
    end;
  finally
    Deployer.Free;
    EnvConfig.Free;
  end;
end.
```

## 8. สรุป DevOps Practices

**DevOps Toolchain:**
```
Code → Git → CI (Build/Test) → CD (Deploy)
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
               Staging          Pre-Prod        Production
               (Auto)           (Auto)          (Manual Gate)
```

**Best Practices:**
1. **Everything as Code** - Config, Infrastructure, Tests
2. **Automated Testing** - Unit, Integration, E2E
3. **Small, Frequent Deployments** - Reduce Risk
4. **Feature Flags** - Decouple Deploy from Release
5. **Observability** - Monitor everything
6. **Immutable Infrastructure** - Never modify, always replace
7. **GitOps** - Git เป็น Single Source of Truth

**วงจร DevOps:**
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → Plan...
