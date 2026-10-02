# ตอนที่ 93: Game Development กับ Pascal/SDL2

## บทนำ: สร้างเกมด้วย Pascal และ SDL2

SDL2 (Simple DirectMedia Layer) เป็น Library สำหรับสร้างเกม 2D ที่ทำงานได้บนทุก Platform

## 1. SDL2 Setup และ Initialization

```pascal
// uSDL2Engine.pas - SDL2 Game Engine Base
unit uSDL2Engine;

{$mode objfpc}{$H+}

interface

uses
  SysUtils, Classes,
  SDL2, SDL2_image, SDL2_mixer, SDL2_ttf;

const
  SCREEN_WIDTH  = 800;
  SCREEN_HEIGHT = 600;
  FPS_TARGET    = 60;
  FRAME_TIME    = 1000 div FPS_TARGET; // ~16ms

type
  TColor = record
    R, G, B, A: Byte;
    class function Create(R, G, B: Byte; A: Byte = 255): TColor; static;
    function ToSDL: TSDL_Color;
  end;

  TVector2 = record
    X, Y: Single;
    class function Create(X, Y: Single): TVector2; static;
    class operator Add(const A, B: TVector2): TVector2;
    class operator Subtract(const A, B: TVector2): TVector2;
    class operator Multiply(const V: TVector2; S: Single): TVector2;
    function Length: Single;
    function Normalized: TVector2;
    function Dot(const Other: TVector2): Single;
  end;

  TRect = record
    X, Y, W, H: Single;
    function Contains(const P: TVector2): Boolean;
    function Intersects(const Other: TRect): Boolean;
    function ToSDL: TSDL_FRect;
  end;

  IScene = interface
    procedure Update(ADeltaTime: Single);
    procedure Render(ARenderer: PSDL_Renderer);
    procedure HandleEvent(const AEvent: TSDL_Event);
    procedure OnEnter;
    procedure OnExit;
  end;

  TGame = class
  private
    FWindow: PSDL_Window;
    FRenderer: PSDL_Renderer;
    FRunning: Boolean;
    FCurrentScene: IScene;
    FNextScene: IScene;
    FLastTime: UInt32;
    FDeltaTime: Single;
    FFPSTimer: UInt32;
    FFPSCount: Integer;
    FFPS: Integer;

    procedure Init;
    procedure Shutdown;
    procedure HandleEvents;
    procedure Update;
    procedure Render;
    procedure CapFPS;

  public
    constructor Create(const ATitle: string;
      AWidth: Integer = SCREEN_WIDTH;
      AHeight: Integer = SCREEN_HEIGHT);
    destructor Destroy; override;

    procedure Run;
    procedure Quit;
    procedure ChangeScene(const AScene: IScene);

    property Window: PSDL_Window read FWindow;
    property Renderer: PSDL_Renderer read FRenderer;
    property DeltaTime: Single read FDeltaTime;
    property FPS: Integer read FFPS;
  end;

implementation

class function TColor.Create(R, G, B: Byte; A: Byte): TColor;
begin
  Result.R := R;
  Result.G := G;
  Result.B := B;
  Result.A := A;
end;

function TColor.ToSDL: TSDL_Color;
begin
  Result.r := R;
  Result.g := G;
  Result.b := B;
  Result.a := A;
end;

class function TVector2.Create(X, Y: Single): TVector2;
begin
  Result.X := X;
  Result.Y := Y;
end;

class operator TVector2.Add(const A, B: TVector2): TVector2;
begin
  Result.X := A.X + B.X;
  Result.Y := A.Y + B.Y;
end;

class operator TVector2.Subtract(const A, B: TVector2): TVector2;
begin
  Result.X := A.X - B.X;
  Result.Y := A.Y - B.Y;
end;

class operator TVector2.Multiply(const V: TVector2; S: Single): TVector2;
begin
  Result.X := V.X * S;
  Result.Y := V.Y * S;
end;

function TVector2.Length: Single;
begin
  Result := Sqrt(X * X + Y * Y);
end;

function TVector2.Normalized: TVector2;
var
  Len: Single;
begin
  Len := Length;
  if Len > 0 then
  begin
    Result.X := X / Len;
    Result.Y := Y / Len;
  end
  else
    Result := TVector2.Create(0, 0);
end;

constructor TGame.Create(const ATitle: string; AWidth, AHeight: Integer);
begin
  inherited Create;
  FRunning := False;

  if SDL_Init(SDL_INIT_VIDEO or SDL_INIT_AUDIO or SDL_INIT_JOYSTICK) <> 0 then
    raise Exception.CreateFmt('SDL_Init failed: %s', [SDL_GetError]);

  if IMG_Init(IMG_INIT_PNG or IMG_INIT_JPG) = 0 then
    raise Exception.CreateFmt('IMG_Init failed: %s', [IMG_GetError]);

  if Mix_OpenAudio(44100, MIX_DEFAULT_FORMAT, 2, 2048) < 0 then
    raise Exception.CreateFmt('Mix_OpenAudio failed: %s', [Mix_GetError]);

  if TTF_Init <> 0 then
    raise Exception.CreateFmt('TTF_Init failed: %s', [TTF_GetError]);

  FWindow := SDL_CreateWindow(
    PChar(ATitle),
    SDL_WINDOWPOS_CENTERED,
    SDL_WINDOWPOS_CENTERED,
    AWidth, AHeight,
    SDL_WINDOW_SHOWN
  );

  if not Assigned(FWindow) then
    raise Exception.CreateFmt('SDL_CreateWindow failed: %s', [SDL_GetError]);

  FRenderer := SDL_CreateRenderer(
    FWindow, -1,
    SDL_RENDERER_ACCELERATED or SDL_RENDERER_PRESENTVSYNC
  );

  if not Assigned(FRenderer) then
    raise Exception.CreateFmt('SDL_CreateRenderer failed: %s', [SDL_GetError]);
end;

procedure TGame.Run;
begin
  FRunning := True;
  FLastTime := SDL_GetTicks;

  while FRunning do
  begin
    HandleEvents;
    Update;
    Render;
    CapFPS;

    // Scene transition
    if Assigned(FNextScene) then
    begin
      if Assigned(FCurrentScene) then
        FCurrentScene.OnExit;
      FCurrentScene := FNextScene;
      FNextScene := nil;
      FCurrentScene.OnEnter;
    end;
  end;
end;

procedure TGame.HandleEvents;
var
  Event: TSDL_Event;
begin
  while SDL_PollEvent(@Event) <> 0 do
  begin
    case Event.type_ of
      SDL_QUITEV:
        FRunning := False;

      SDL_KEYDOWN:
        if Event.key.keysym.sym = SDLK_F4 then
          FRunning := False;
    end;

    if Assigned(FCurrentScene) then
      FCurrentScene.HandleEvent(Event);
  end;
end;

procedure TGame.Update;
var
  CurrentTime: UInt32;
begin
  CurrentTime := SDL_GetTicks;
  FDeltaTime := (CurrentTime - FLastTime) / 1000.0; // seconds
  FLastTime := CurrentTime;

  // Cap delta to prevent spiral of death
  if FDeltaTime > 0.1 then FDeltaTime := 0.1;

  if Assigned(FCurrentScene) then
    FCurrentScene.Update(FDeltaTime);

  // FPS calculation
  Inc(FFPSCount);
  if SDL_GetTicks - FFPSTimer >= 1000 then
  begin
    FFPS := FFPSCount;
    FFPSCount := 0;
    FFPSTimer := SDL_GetTicks;
  end;
end;

procedure TGame.Render;
begin
  SDL_SetRenderDrawColor(FRenderer, 0, 0, 0, 255);
  SDL_RenderClear(FRenderer);

  if Assigned(FCurrentScene) then
    FCurrentScene.Render(FRenderer);

  SDL_RenderPresent(FRenderer);
end;

procedure TGame.CapFPS;
var
  FrameEnd: UInt32;
  Delay: UInt32;
begin
  FrameEnd := SDL_GetTicks;
  var FrameTime := FrameEnd - FLastTime;
  if FrameTime < FRAME_TIME then
  begin
    Delay := FRAME_TIME - FrameTime;
    SDL_Delay(Delay);
  end;
end;

end.
```

## 2. Sprite และ Texture System

```pascal
// uSprite.pas - Sprite System
unit uSprite;

{$mode objfpc}{$H+}

interface

uses
  SysUtils, Classes, SDL2, SDL2_image, uSDL2Engine;

type
  TTexture = class
  private
    FTexture: PSDL_Texture;
    FWidth: Integer;
    FHeight: Integer;
    FRenderer: PSDL_Renderer;

  public
    constructor Create(ARenderer: PSDL_Renderer;
      const AFilePath: string);
    destructor Destroy; override;

    procedure Draw(AX, AY: Single; AAngle: Double = 0;
      const AClip: PSDL_Rect = nil);
    procedure DrawEx(const ADest: TSDL_FRect;
      AAngle: Double = 0; AFlipH: Boolean = False;
      AFlipV: Boolean = False);
    procedure SetAlpha(AAlpha: Byte);
    procedure SetColorMod(AR, AG, AB: Byte);

    property Width: Integer read FWidth;
    property Height: Integer read FHeight;
  end;

  // Animation Frame
  TFrame = record
    Clip: TSDL_Rect;
    Duration: Single; // seconds
  end;

  TAnimation = class
  private
    FFrames: array of TFrame;
    FCurrentFrame: Integer;
    FTimer: Single;
    FLooping: Boolean;
    FFinished: Boolean;
    FOnFinished: TNotifyEvent;

  public
    constructor Create(ALooping: Boolean = True);

    procedure AddFrame(const AClip: TSDL_Rect; ADuration: Single);
    procedure Update(ADeltaTime: Single);
    function GetCurrentClip: TSDL_Rect;
    procedure Reset;

    property CurrentFrame: Integer read FCurrentFrame;
    property Finished: Boolean read FFinished;
    property OnFinished: TNotifyEvent read FOnFinished write FOnFinished;
  end;

  TAnimatedSprite = class
  private
    FTexture: TTexture;
    FAnimations: TStringList;
    FCurrentAnimation: TAnimation;
    FCurrentAnimName: string;
    FPosition: TVector2;
    FScale: TVector2;
    FAngle: Double;
    FFlipH: Boolean;

  public
    constructor Create(ATexture: TTexture);
    destructor Destroy; override;

    procedure AddAnimation(const AName: string; AAnimation: TAnimation);
    procedure PlayAnimation(const AName: string);
    procedure Update(ADeltaTime: Single);
    procedure Draw(ARenderer: PSDL_Renderer);

    property Position: TVector2 read FPosition write FPosition;
    property Scale: TVector2 read FScale write FScale;
    property Angle: Double read FAngle write FAngle;
    property FlipH: Boolean read FFlipH write FFlipH;
  end;

implementation

constructor TTexture.Create(ARenderer: PSDL_Renderer;
  const AFilePath: string);
var
  Surface: PSDL_Surface;
begin
  inherited Create;
  FRenderer := ARenderer;

  Surface := IMG_Load(PChar(AFilePath));
  if not Assigned(Surface) then
    raise Exception.CreateFmt('Cannot load image: %s', [AFilePath]);

  try
    FTexture := SDL_CreateTextureFromSurface(ARenderer, Surface);
    FWidth := Surface^.w;
    FHeight := Surface^.h;
  finally
    SDL_FreeSurface(Surface);
  end;

  if not Assigned(FTexture) then
    raise Exception.CreateFmt('Cannot create texture: %s',
      [SDL_GetError]);
end;

procedure TTexture.Draw(AX, AY: Single; AAngle: Double;
  const AClip: PSDL_Rect);
var
  DestRect: TSDL_FRect;
  W, H: Integer;
begin
  if Assigned(AClip) then
  begin
    W := AClip^.w;
    H := AClip^.h;
  end
  else
  begin
    W := FWidth;
    H := FHeight;
  end;

  DestRect.x := AX;
  DestRect.y := AY;
  DestRect.w := W;
  DestRect.h := H;

  SDL_RenderCopyExF(FRenderer, FTexture, AClip, @DestRect,
    AAngle, nil, SDL_FLIP_NONE);
end;

procedure TAnimation.Update(ADeltaTime: Single);
begin
  if FFinished then Exit;

  FTimer := FTimer + ADeltaTime;

  while FTimer >= FFrames[FCurrentFrame].Duration do
  begin
    FTimer := FTimer - FFrames[FCurrentFrame].Duration;
    Inc(FCurrentFrame);

    if FCurrentFrame >= Length(FFrames) then
    begin
      if FLooping then
        FCurrentFrame := 0
      else
      begin
        FCurrentFrame := Length(FFrames) - 1;
        FFinished := True;
        if Assigned(FOnFinished) then
          FOnFinished(Self);
        Exit;
      end;
    end;
  end;
end;

end.
```

## 3. Collision Detection

```pascal
// uCollision.pas - Collision Detection System
unit uCollision;

{$mode objfpc}{$H+}

interface

uses SysUtils, Classes, Generics.Collections, uSDL2Engine;

type
  TColliderType = (ctRect, ctCircle, ctPolygon);

  TCollider = class
  public
    Position: TVector2; // World position
    Offset: TVector2;   // Local offset
    IsTrigger: Boolean;
    Layer: Integer;
    Tag: string;

    function GetBounds: TRect; virtual; abstract;
    function Intersects(const Other: TCollider): Boolean; virtual; abstract;
    function Contains(const APoint: TVector2): Boolean; virtual; abstract;
  end;

  TRectCollider = class(TCollider)
  public
    Width, Height: Single;

    function GetBounds: TRect; override;
    function Intersects(const Other: TCollider): Boolean; override;
    function Contains(const APoint: TVector2): Boolean; override;
  end;

  TCircleCollider = class(TCollider)
  public
    Radius: Single;

    function GetBounds: TRect; override;
    function Intersects(const Other: TCollider): Boolean; override;
    function Contains(const APoint: TVector2): Boolean; override;
  end;

  TCollisionInfo = record
    ColliderA: TCollider;
    ColliderB: TCollider;
    Normal: TVector2;    // Collision normal
    Depth: Single;       // Penetration depth
    IsEnter: Boolean;    // True on first contact
    IsExit: Boolean;     // True when separating
  end;

  TCollisionEvent = procedure(const AInfo: TCollisionInfo) of object;

  TCollisionSystem = class
  private
    FColliders: TObjectList<TCollider>;
    FPreviousCollisions: TList<TCollider>;
    FOnCollision: TCollisionEvent;
    FOnTrigger: TCollisionEvent;

    function CheckCircleCircle(const AC: TCircleCollider;
      const BC: TCircleCollider; out AInfo: TCollisionInfo): Boolean;
    function CheckRectRect(const AR: TRectCollider;
      const BR: TRectCollider; out AInfo: TCollisionInfo): Boolean;
    function CheckCircleRect(const AC: TCircleCollider;
      const AR: TRectCollider; out AInfo: TCollisionInfo): Boolean;

  public
    constructor Create;
    destructor Destroy; override;

    procedure AddCollider(const ACollider: TCollider);
    procedure RemoveCollider(const ACollider: TCollider);
    procedure Update;

    property OnCollision: TCollisionEvent
      read FOnCollision write FOnCollision;
    property OnTrigger: TCollisionEvent
      read FOnTrigger write FOnTrigger;
  end;

implementation

function TRectCollider.GetBounds: TRect;
begin
  Result.X := Position.X + Offset.X;
  Result.Y := Position.Y + Offset.Y;
  Result.W := Width;
  Result.H := Height;
end;

function TRectCollider.Intersects(const Other: TCollider): Boolean;
var
  Info: TCollisionInfo;
  ASystem: TCollisionSystem;
begin
  if Other is TRectCollider then
  begin
    var B := TRectCollider(Other).GetBounds;
    var A := GetBounds;
    Result := (A.X < B.X + B.W) and (A.X + A.W > B.X) and
              (A.Y < B.Y + B.H) and (A.Y + A.H > B.Y);
  end
  else if Other is TCircleCollider then
  begin
    // Defer to circle-rect check
    var C := TCircleCollider(Other);
    var Bounds := GetBounds;
    var CenterX := C.Position.X + C.Offset.X;
    var CenterY := C.Position.Y + C.Offset.Y;
    var ClosestX := Max(Bounds.X, Min(CenterX, Bounds.X + Bounds.W));
    var ClosestY := Max(Bounds.Y, Min(CenterY, Bounds.Y + Bounds.H));
    var DX := CenterX - ClosestX;
    var DY := CenterY - ClosestY;
    Result := (DX * DX + DY * DY) <= (C.Radius * C.Radius);
  end
  else
    Result := False;
end;

function TCircleCollider.Intersects(const Other: TCollider): Boolean;
var
  CX, CY, OX, OY: Single;
  Dist, SumR: Single;
begin
  CX := Position.X + Offset.X;
  CY := Position.Y + Offset.Y;

  if Other is TCircleCollider then
  begin
    var OC := TCircleCollider(Other);
    OX := OC.Position.X + OC.Offset.X;
    OY := OC.Position.Y + OC.Offset.Y;
    Dist := Sqrt((CX - OX) * (CX - OX) + (CY - OY) * (CY - OY));
    SumR := Radius + OC.Radius;
    Result := Dist <= SumR;
  end
  else
    Result := Other.Intersects(Self);
end;

end.
```

## 4. Complete Pong Game

```pascal
// Pong/PongGame.pas - Classic Pong เกม
program PongGame;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, SDL2, SDL2_ttf,
  uSDL2Engine;

const
  PADDLE_WIDTH  = 15;
  PADDLE_HEIGHT = 100;
  PADDLE_SPEED  = 300.0;
  BALL_SIZE     = 15;
  BALL_SPEED    = 250.0;
  BALL_MAX_SPEED = 500.0;
  WIN_SCORE     = 11;

type
  TPaddle = record
    Position: TVector2;
    Score: Integer;
    IsPlayer: Boolean; // True = player, False = AI
  end;

  TBall = record
    Position: TVector2;
    Velocity: TVector2;
    Active: Boolean;
  end;

  TPongScene = class(TInterfacedObject, IScene)
  private
    FGame: TGame;
    FFont: PTTF_Font;
    FBigFont: PTTF_Font;
    FPlayer: TPaddle;
    FCPU: TPaddle;
    FBall: TBall;
    FGameState: (gsPlaying, gsGoal, gsGameOver);
    FGoalTimer: Single;
    FLastScorer: Integer; // 1=player, 2=cpu

    procedure ResetBall(ADirection: Integer = 0);
    procedure UpdatePlayer(ADeltaTime: Single);
    procedure UpdateCPU(ADeltaTime: Single);
    procedure UpdateBall(ADeltaTime: Single);
    procedure CheckBallCollision;
    procedure DrawPaddle(ARenderer: PSDL_Renderer;
      const APaddle: TPaddle);
    procedure DrawBall(ARenderer: PSDL_Renderer);
    procedure DrawScore(ARenderer: PSDL_Renderer);
    procedure DrawNet(ARenderer: PSDL_Renderer);
    function RenderText(ARenderer: PSDL_Renderer;
      const AText: string; AFont: PTTF_Font;
      AColor: TSDL_Color; AX, AY: Integer): TSDL_Rect;

  public
    constructor Create(AGame: TGame);
    destructor Destroy; override;

    procedure Update(ADeltaTime: Single);
    procedure Render(ARenderer: PSDL_Renderer);
    procedure HandleEvent(const AEvent: TSDL_Event);
    procedure OnEnter;
    procedure OnExit;
  end;

constructor TPongScene.Create(AGame: TGame);
begin
  inherited Create;
  FGame := AGame;

  FFont := TTF_OpenFont('assets/fonts/PressStart2P.ttf', 16);
  FBigFont := TTF_OpenFont('assets/fonts/PressStart2P.ttf', 32);

  // Init Player (left paddle)
  FPlayer.Position := TVector2.Create(30,
    (SCREEN_HEIGHT - PADDLE_HEIGHT) / 2);
  FPlayer.Score := 0;
  FPlayer.IsPlayer := True;

  // Init CPU (right paddle)
  FCPU.Position := TVector2.Create(
    SCREEN_WIDTH - 30 - PADDLE_WIDTH,
    (SCREEN_HEIGHT - PADDLE_HEIGHT) / 2);
  FCPU.Score := 0;
  FCPU.IsPlayer := False;

  ResetBall;
  FGameState := gsPlaying;
end;

procedure TPongScene.ResetBall(ADirection: Integer);
begin
  FBall.Position := TVector2.Create(
    SCREEN_WIDTH / 2 - BALL_SIZE / 2,
    SCREEN_HEIGHT / 2 - BALL_SIZE / 2);

  var Angle := (Random(60) - 30) * Pi / 180;
  var Dir := ADirection;
  if Dir = 0 then
    Dir := 1 - 2 * Random(2);

  FBall.Velocity := TVector2.Create(
    Dir * BALL_SPEED * Cos(Angle),
    BALL_SPEED * Sin(Angle));
  FBall.Active := True;
end;

procedure TPongScene.UpdatePlayer(ADeltaTime: Single);
var
  Keys: PByte;
begin
  Keys := SDL_GetKeyboardState(nil);

  if Keys[SDL_SCANCODE_UP] <> 0 then
    FPlayer.Position.Y := FPlayer.Position.Y -
      PADDLE_SPEED * ADeltaTime;

  if Keys[SDL_SCANCODE_DOWN] <> 0 then
    FPlayer.Position.Y := FPlayer.Position.Y +
      PADDLE_SPEED * ADeltaTime;

  // Clamp to screen
  FPlayer.Position.Y := Max(0,
    Min(SCREEN_HEIGHT - PADDLE_HEIGHT, FPlayer.Position.Y));
end;

procedure TPongScene.UpdateCPU(ADeltaTime: Single);
var
  BallCenter, PaddleCenter: Single;
  Speed: Single;
begin
  // Simple AI - follow ball
  BallCenter := FBall.Position.Y + BALL_SIZE / 2;
  PaddleCenter := FCPU.Position.Y + PADDLE_HEIGHT / 2;

  Speed := PADDLE_SPEED * 0.8; // CPU slightly slower

  if BallCenter < PaddleCenter - 5 then
    FCPU.Position.Y := FCPU.Position.Y - Speed * ADeltaTime
  else if BallCenter > PaddleCenter + 5 then
    FCPU.Position.Y := FCPU.Position.Y + Speed * ADeltaTime;

  FCPU.Position.Y := Max(0,
    Min(SCREEN_HEIGHT - PADDLE_HEIGHT, FCPU.Position.Y));
end;

procedure TPongScene.UpdateBall(ADeltaTime: Single);
begin
  if not FBall.Active then Exit;

  FBall.Position.X := FBall.Position.X +
    FBall.Velocity.X * ADeltaTime;
  FBall.Position.Y := FBall.Position.Y +
    FBall.Velocity.Y * ADeltaTime;

  // Bounce off top/bottom
  if FBall.Position.Y <= 0 then
  begin
    FBall.Position.Y := 0;
    FBall.Velocity.Y := -FBall.Velocity.Y;
  end;

  if FBall.Position.Y + BALL_SIZE >= SCREEN_HEIGHT then
  begin
    FBall.Position.Y := SCREEN_HEIGHT - BALL_SIZE;
    FBall.Velocity.Y := -FBall.Velocity.Y;
  end;

  // Ball out of bounds = Score
  if FBall.Position.X < 0 then
  begin
    // CPU scores
    Inc(FCPU.Score);
    FLastScorer := 2;
    FGameState := gsGoal;
    FGoalTimer := 1.5;
  end
  else if FBall.Position.X + BALL_SIZE > SCREEN_WIDTH then
  begin
    // Player scores
    Inc(FPlayer.Score);
    FLastScorer := 1;
    FGameState := gsGoal;
    FGoalTimer := 1.5;
  end;

  CheckBallCollision;
end;

procedure TPongScene.CheckBallCollision;
var
  BallX, BallY: Single;
  BallRight, BallBottom: Single;
  RelativeImpact: Single;
  BounceAngle: Single;
  Speed: Single;

  procedure CheckPaddle(const APaddle: TPaddle; AIsLeft: Boolean);
  var
    PaddleRight: Single;
    PaddleBottom: Single;
  begin
    PaddleRight := APaddle.Position.X + PADDLE_WIDTH;
    PaddleBottom := APaddle.Position.Y + PADDLE_HEIGHT;

    // Check collision
    if (BallRight > APaddle.Position.X) and
       (BallX < PaddleRight) and
       (BallBottom > APaddle.Position.Y) and
       (BallY < PaddleBottom) then
    begin
      // Calculate relative impact point (-1 to 1)
      RelativeImpact := (FBall.Position.Y + BALL_SIZE / 2 -
        (APaddle.Position.Y + PADDLE_HEIGHT / 2)) /
        (PADDLE_HEIGHT / 2);
      RelativeImpact := Max(-1, Min(1, RelativeImpact));

      // Bounce angle (max 75 degrees)
      BounceAngle := RelativeImpact * 75 * Pi / 180;

      // Increase speed slightly
      Speed := FBall.Velocity.Length * 1.05;
      Speed := Min(Speed, BALL_MAX_SPEED);

      if AIsLeft then
      begin
        FBall.Velocity.X := Abs(Speed * Cos(BounceAngle));
        FBall.Position.X := PaddleRight;
      end
      else
      begin
        FBall.Velocity.X := -Abs(Speed * Cos(BounceAngle));
        FBall.Position.X := APaddle.Position.X - BALL_SIZE;
      end;

      FBall.Velocity.Y := Speed * Sin(BounceAngle);
    end;
  end;

begin
  BallX := FBall.Position.X;
  BallY := FBall.Position.Y;
  BallRight := BallX + BALL_SIZE;
  BallBottom := BallY + BALL_SIZE;

  CheckPaddle(FPlayer, True);
  CheckPaddle(FCPU, False);
end;

procedure TPongScene.Update(ADeltaTime: Single);
begin
  case FGameState of
    gsPlaying:
    begin
      UpdatePlayer(ADeltaTime);
      UpdateCPU(ADeltaTime);
      UpdateBall(ADeltaTime);

      if (FPlayer.Score >= WIN_SCORE) or
         (FCPU.Score >= WIN_SCORE) then
        FGameState := gsGameOver;
    end;

    gsGoal:
    begin
      FGoalTimer := FGoalTimer - ADeltaTime;
      if FGoalTimer <= 0 then
      begin
        ResetBall(FLastScorer * 2 - 3); // Winner serves
        FGameState := gsPlaying;
      end;
    end;
  end;
end;

procedure TPongScene.Render(ARenderer: PSDL_Renderer);
begin
  DrawNet(ARenderer);
  DrawPaddle(ARenderer, FPlayer);
  DrawPaddle(ARenderer, FCPU);
  DrawBall(ARenderer);
  DrawScore(ARenderer);

  if FGameState = gsGameOver then
  begin
    var WinText := 'GAME OVER';
    if FPlayer.Score > FCPU.Score then
      WinText := 'YOU WIN!'
    else
      WinText := 'CPU WINS!';

    var Color: TSDL_Color;
    Color.r := 255; Color.g := 255; Color.b := 0; Color.a := 255;
    RenderText(ARenderer, WinText, FBigFont, Color,
      SCREEN_WIDTH div 2, SCREEN_HEIGHT div 2);

    Color.r := 200; Color.g := 200; Color.b := 200; Color.a := 255;
    RenderText(ARenderer, 'Press R to restart', FFont, Color,
      SCREEN_WIDTH div 2, SCREEN_HEIGHT div 2 + 60);
  end;
end;

procedure TPongScene.DrawPaddle(ARenderer: PSDL_Renderer;
  const APaddle: TPaddle);
var
  Rect: TSDL_FRect;
begin
  SDL_SetRenderDrawColor(ARenderer, 255, 255, 255, 255);
  Rect.x := APaddle.Position.X;
  Rect.y := APaddle.Position.Y;
  Rect.w := PADDLE_WIDTH;
  Rect.h := PADDLE_HEIGHT;
  SDL_RenderFillRectF(ARenderer, @Rect);
end;

procedure TPongScene.HandleEvent(const AEvent: TSDL_Event);
begin
  if AEvent.type_ = SDL_KEYDOWN then
    if AEvent.key.keysym.sym = SDLK_r then
    begin
      FPlayer.Score := 0;
      FCPU.Score := 0;
      ResetBall;
      FGameState := gsPlaying;
    end;
end;

var
  Game: TGame;
  Scene: TPongScene;
begin
  Randomize;
  Game := TGame.Create('Pong - Pascal SDL2');
  try
    Scene := TPongScene.Create(Game);
    Game.ChangeScene(Scene);
    Game.Run;
  finally
    Game.Free;
  end;
end.
```

## 5. สรุป Game Development

**SDL2 Libraries:**
- `SDL2` - Core: window, events, rendering
- `SDL2_image` - Load PNG, JPG, BMP
- `SDL2_mixer` - Audio: WAV, MP3, OGG
- `SDL2_ttf` - TrueType Font rendering
- `SDL2_net` - Networking (multiplayer)

**Game Loop Pattern:**
```
while running:
  processEvents()     -- Input
  update(deltaTime)   -- Logic
  render()            -- Drawing
  capFPS()            -- 60 FPS
```

**Pascal SDL2 Bindings:** ใช้ `sdl2-pascal` package จาก OPM

**การ Compile:**
```bash
fpc -Fl/usr/lib/x86_64-linux-gnu \
    -k"-lSDL2 -lSDL2_image -lSDL2_ttf -lSDL2_mixer" \
    PongGame.pas
```
