// =====================================================
// ESP32 RETRO GAME CONSOLE
// PART 1 OF 12
// CORE SYSTEM
// =====================================================

#include <Arduino.h>
#include <Wire.h>
#include <Preferences.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_ADDR 0x3C

#define BTN32 32
#define BTN33 33

Adafruit_SSD1306 display(
  SCREEN_WIDTH,
  SCREEN_HEIGHT,
  &Wire,
  -1
);

Preferences prefs;

// =====================================================
// STATES
// =====================================================

enum State
{
  SPLASH,
  MENU,

  PONG_INFO,
  PONG_GAME,
  PONG_OVER,

  DINO_INFO,
  DINO_GAME,
  DINO_OVER,

  FLAPPY_INFO,
  FLAPPY_GAME,
  FLAPPY_OVER,

  CHICKEN_INFO,
  CHICKEN_GAME,
  CHICKEN_OVER,

  STACK_INFO,
  STACK_GAME,
  STACK_OVER
};

State state = SPLASH;

// =====================================================
// BUTTON SYSTEM
// =====================================================

bool btn32 = false;
bool btn33 = false;

bool old32 = false;
bool old33 = false;

bool press32 = false;
bool press33 = false;

void readButtons()
{
  old32 = btn32;
  old33 = btn33;

  btn32 = !digitalRead(BTN32);
  btn33 = !digitalRead(BTN33);

  press32 = btn32 && !old32;
  press33 = btn33 && !old33;
}

// =====================================================
// HIGH SCORES
// =====================================================

int pongHigh = 0;
int dinoHigh = 0;
int flappyHigh = 0;
int chickenHigh = 0;
int stackHigh = 0;

// =====================================================
// SPLASH TIMER
// =====================================================

unsigned long splashStart;

// =====================================================
// FORWARD DECLARATIONS
// =====================================================

// Menu
void drawSplash();
void drawMenu();
void updateMenu();

// High Scores
void loadScores();
void saveScores();

// Pong
void initPong();
void updatePong();
void drawPong();

// Dino
void initDino();
void updateDino();
void drawDino();

// Flappy
void initFlappy();
void updateFlappy();
void drawFlappy();

// Chicken
void initChicken();
void updateChicken();
void drawChicken();

// Stack
void initStack();
void updateStack();
void drawStack();
// =====================================================
// PART 2 OF 12
// SPLASH SCREEN + MAIN MENU
// =====================================================

// -------------------------
// MENU ITEMS
// -------------------------

const char* menuItems[] =
{
  "PONG",
  "DINO RUN",
  "FLAPPY BIRD",
  "CHICKEN ROAD",
  "STACK BOX"
};

int menuIndex = 0;

// -------------------------
// SPLASH SCREEN
// -------------------------

void drawSplash()
{
  display.clearDisplay();

  // Pixel Gamepad

  display.fillRoundRect(
    42, 10,
    44, 22,
    4,
    SSD1306_WHITE
  );

  // D-Pad

  display.fillRect(50,17,10,2,SSD1306_BLACK);
  display.fillRect(54,13,2,10,SSD1306_BLACK);

  // Buttons

  display.fillCircle(72,17,2,SSD1306_BLACK);
  display.fillCircle(78,21,2,SSD1306_BLACK);

  display.setTextSize(1);

  display.setCursor(12,45);
  display.print("ESP32 GAME CONSOLE");

  display.display();
}

// -------------------------
// MENU UPDATE
// -------------------------

void updateMenu()
{
  // GPIO32 = NEXT ITEM

  if(press32)
  {
    menuIndex++;

    if(menuIndex > 4)
      menuIndex = 0;
  }

  // GPIO33 = SELECT

  if(press33)
  {
    switch(menuIndex)
    {
      case 0:
        state = PONG_INFO;
        break;

      case 1:
        state = DINO_INFO;
        break;

      case 2:
        state = FLAPPY_INFO;
        break;

      case 3:
        state = CHICKEN_INFO;
        break;

      case 4:
        state = STACK_INFO;
        break;
    }
  }
}

// -------------------------
// DRAW MENU
// -------------------------

void drawMenu()
{
  display.clearDisplay();

  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(18,0);
  display.print("RETRO CONSOLE");

  for(int i=0;i<5;i++)
  {
    display.setCursor(
      8,
      14 + (i * 10)
    );

    if(i == menuIndex)
      display.print("> ");
    else
      display.print("  ");

    display.print(menuItems[i]);
  }

  display.display();
}
// =====================================================
// PART 3 OF 12
// HIGH SCORE SYSTEM
// =====================================================

void loadScores()
{
  prefs.begin("retro", true);

  pongHigh =
    prefs.getInt("pongHigh", 0);

  dinoHigh =
    prefs.getInt("dinoHigh", 0);

  flappyHigh =
    prefs.getInt("flappyHigh", 0);

  chickenHigh =
    prefs.getInt("chickenHigh", 0);

  stackHigh =
    prefs.getInt("stackHigh", 0);

  prefs.end();
}

void saveScores()
{
  prefs.begin("retro", false);

  prefs.putInt(
    "pongHigh",
    pongHigh
  );

  prefs.putInt(
    "dinoHigh",
    dinoHigh
  );

  prefs.putInt(
    "flappyHigh",
    flappyHigh
  );

  prefs.putInt(
    "chickenHigh",
    chickenHigh
  );

  prefs.putInt(
    "stackHigh",
    stackHigh
  );

  prefs.end();
}

// =====================================================
// GAME INFO SCREEN HELPERS
// =====================================================

void drawInfoScreen(
  const char* title,
  int highScore
)
{
  display.clearDisplay();

  display.setTextSize(2);
  display.setCursor(10,6);
  display.print(title);

  display.setTextSize(1);

  display.setCursor(20,34);
  display.print("HIGH SCORE:");

  display.print(highScore);

  display.setCursor(6,54);
  display.print("33=START 32=MENU");

  display.display();
}
// =====================================================
// PART 4 OF 12
// PONG GAME
// =====================================================

// --------------------
// VARIABLES
// --------------------

float pongBallX;
float pongBallY;

float pongBallVX;
float pongBallVY;

int pongPaddleX;

const int pongPaddleW = 24;
const int pongPaddleY = 60;

int pongScore = 0;

bool pongStarted = false;

// --------------------
// INIT
// --------------------

void initPong()
{
  pongPaddleX = 52;

  pongBallX = 64;
  pongBallY = 10;

  pongBallVX = random(-20, 21) / 10.0;

  if(pongBallVX == 0)
    pongBallVX = 1.2;

  pongBallVY = 1.8;

  pongScore = 0;

  pongStarted = false;
}

// --------------------
// UPDATE
// --------------------

void updatePong()
{
  // INFO SCREEN

  if(!pongStarted)
  {
    if(press32)
      state = MENU;

    if(press33)
      pongStarted = true;

    return;
  }

  // MOVE PADDLE

  if(btn32)
    pongPaddleX -= 3;

  if(btn33)
    pongPaddleX += 3;

  if(pongPaddleX < 0)
    pongPaddleX = 0;

  if(pongPaddleX > 128 - pongPaddleW)
    pongPaddleX = 128 - pongPaddleW;

  // MOVE BALL

  pongBallX += pongBallVX;
  pongBallY += pongBallVY;

  // WALL COLLISION

  if(pongBallX <= 2)
  {
    pongBallX = 2;
    pongBallVX = -pongBallVX;
  }

  if(pongBallX >= 126)
  {
    pongBallX = 126;
    pongBallVX = -pongBallVX;
  }

  if(pongBallY <= 2)
  {
    pongBallY = 2;
    pongBallVY = -pongBallVY;
  }

  // PADDLE COLLISION

  if(
      pongBallY >= pongPaddleY - 2 &&
      pongBallY <= pongPaddleY + 2 &&
      pongBallX >= pongPaddleX &&
      pongBallX <= pongPaddleX + pongPaddleW
    )
  {
    float hitPos =
      (pongBallX - pongPaddleX) /
      (float)pongPaddleW;

    // Angle bounce

    pongBallVX =
      (hitPos - 0.5f) * 7.0f;

    pongBallVY =
      -(2.5f + pongScore * 0.08f);

    pongScore++;

    if(pongScore > pongHigh)
    {
      pongHigh = pongScore;
      saveScores();
    }
  }

  // GAME OVER

  if(pongBallY > 64)
  {
    state = PONG_OVER;
  }
}

// --------------------
// DRAW
// --------------------

void drawPong()
{
  display.clearDisplay();

  // START SCREEN

  if(!pongStarted)
  {
    drawInfoScreen(
      "PONG",
      pongHigh
    );

    return;
  }

  // SCORE

  display.setCursor(0,0);
  display.print("S:");
  display.print(pongScore);

  display.setCursor(84,0);
  display.print("H:");
  display.print(pongHigh);

  // BALL

  display.fillCircle(
    (int)pongBallX,
    (int)pongBallY,
    2,
    SSD1306_WHITE
  );

  // PADDLE

  display.fillRect(
    pongPaddleX,
    pongPaddleY,
    pongPaddleW,
    3,
    SSD1306_WHITE
  );

  display.display();
}
// =====================================================
// PART 5 OF 12
// DINO RUN
// =====================================================

float dinoY;
float dinoVel;

bool dinoGround;
bool dinoStarted;

int dinoScore;

float obstacleX;
int obstacleType;

float dinoSpeed;

int cloudX;

void initDino()
{
  dinoY = 48;
  dinoVel = 0;

  dinoGround = true;
  dinoStarted = false;

  dinoScore = 0;

  obstacleX = 140;
  obstacleType = random(0,4);

  dinoSpeed = 2.0;

  cloudX = 128;
}

void updateDino()
{
  if(!dinoStarted)
  {
    if(press32) state = MENU;
    if(press33) dinoStarted = true;
    return;
  }

  if(press33 && dinoGround)
  {
    dinoVel = -5.5;
    dinoGround = false;
  }

  dinoVel += 0.3;
  dinoY += dinoVel;

  if(dinoY >= 48)
  {
    dinoY = 48;
    dinoVel = 0;
    dinoGround = true;
  }

  obstacleX -= dinoSpeed;

  if(obstacleX < -20)
  {
    obstacleX = 140;

    obstacleType = random(0,4);

    dinoScore++;

    dinoSpeed += 0.08;

    if(dinoScore > dinoHigh)
    {
      dinoHigh = dinoScore;
      saveScores();
    }
  }

  cloudX--;

  if(cloudX < -20)
    cloudX = 128;

  int obsW = 8;
  int obsY = 50;

  if(obstacleType == 1)
  {
    obsW = 8;
    obsY = 44;
  }

  if(obstacleType == 2)
  {
    obsW = 16;
    obsY = 50;
  }

  if(obstacleType == 3)
  {
    obsW = 12;
    obsY = 40;
  }

  bool hit =
    obstacleX < 27 &&
    obstacleX + obsW > 15 &&
    obsY < dinoY + 12 &&
    obsY + 12 > dinoY;

  if(hit)
    state = DINO_OVER;
}

void drawDino()
{
  display.clearDisplay();

  if(!dinoStarted)
  {
    drawInfoScreen(
      "DINO",
      dinoHigh
    );
    return;
  }

  display.drawLine(
    0,62,127,62,
    SSD1306_WHITE
  );

  display.drawRoundRect(
    cloudX,
    10,
    18,
    6,
    2,
    SSD1306_WHITE
  );

  display.setCursor(0,0);
  display.print(dinoScore);

  int frame =
    (millis()/180)%2;

  int x = 15;
  int y = (int)dinoY;

  display.drawRect(x+2,y,8,5,SSD1306_WHITE);
  display.drawRect(x,y+5,10,7,SSD1306_WHITE);

  if(frame)
  {
    display.fillRect(x+1,y+12,2,3,SSD1306_WHITE);
    display.fillRect(x+7,y+12,2,3,SSD1306_WHITE);
  }
  else
  {
    display.fillRect(x+1,y+11,2,2,SSD1306_WHITE);
    display.fillRect(x+7,y+13,2,2,SSD1306_WHITE);
  }

  if(obstacleType == 0)
  {
    display.fillRect(
      obstacleX,
      50,
      6,
      12,
      SSD1306_WHITE
    );
  }
  else if(obstacleType == 1)
  {
    display.fillRect(
      obstacleX,
      44,
      6,
      18,
      SSD1306_WHITE
    );
  }
  else if(obstacleType == 2)
  {
    display.fillRect(obstacleX,50,6,12,SSD1306_WHITE);
    display.fillRect(obstacleX+8,50,6,12,SSD1306_WHITE);
  }
  else
  {
    display.drawRect(
      obstacleX,
      40,
      10,
      5,
      SSD1306_WHITE
    );
  }

  display.display();
}
// =====================================================
// PART 6 OF 12
// FLAPPY BIRD
// =====================================================

float birdY;
float birdVel;

int pipeX;
int pipeGapY;

int flappyScore;

bool flappyStarted;

void initFlappy()
{
  birdY = 32;
  birdVel = 0;

  pipeX = 128;
  pipeGapY = random(18,46);

  flappyScore = 0;

  flappyStarted = false;
}

void updateFlappy()
{
  if(!flappyStarted)
  {
    if(press32) state = MENU;
    if(press33) flappyStarted = true;
    return;
  }

  if(press33)
    birdVel = -3.5;

  birdVel += 0.20;
  birdY += birdVel;

  pipeX -= 2;

  if(pipeX < -12)
  {
    pipeX = 128;

    pipeGapY = random(16,48);

    flappyScore++;

    if(flappyScore > flappyHigh)
    {
      flappyHigh = flappyScore;
      saveScores();
    }
  }

  if(
      20 > pipeX &&
      20 < pipeX + 12
    )
  {
    if(
        birdY < pipeGapY-10 ||
        birdY > pipeGapY+10
      )
    {
      state = FLAPPY_OVER;
    }
  }

  if(birdY < 0 || birdY > 63)
    state = FLAPPY_OVER;
}

void drawFlappy()
{
  display.clearDisplay();

  if(!flappyStarted)
  {
    drawInfoScreen(
      "FLAPPY",
      flappyHigh
    );
    return;
  }

  display.fillCircle(
    20,
    (int)birdY,
    3,
    SSD1306_WHITE
  );

  display.fillRect(
    pipeX,
    0,
    12,
    pipeGapY-10,
    SSD1306_WHITE
  );

  display.fillRect(
    pipeX,
    pipeGapY+10,
    12,
    64,
    SSD1306_WHITE
  );

  display.setCursor(0,0);
  display.print(flappyScore);

  display.display();
}
// =====================================================
// PART 7 OF 12
// CHICKEN ROAD
// =====================================================

int chickenLane;
int chickenScore;
bool chickenStarted;

float carX[4];
float carSpeed[4];

void initChicken()
{
  chickenLane = 4;
  chickenScore = 0;
  chickenStarted = false;

  for(int i=0;i<4;i++)
  {
    carX[i] = random(0,128);
    carSpeed[i] = 1.5 + (i * 0.5);
  }
}

void updateChicken()
{
  if(!chickenStarted)
  {
    if(press32) state = MENU;
    if(press33) chickenStarted = true;
    return;
  }

  // UP
  if(press33 && chickenLane > 0)
  {
    chickenLane--;

    if(chickenLane == 0)
    {
      chickenScore++;

      if(chickenScore > chickenHigh)
      {
        chickenHigh = chickenScore;
        saveScores();
      }

      chickenLane = 4;

      for(int i=0;i<4;i++)
        carSpeed[i] += 0.15;
    }
  }

  // DOWN
  if(press32 && chickenLane < 4)
  {
    chickenLane++;
  }

  for(int i=0;i<4;i++)
  {
    carX[i] += carSpeed[i];

    if(carX[i] > 140)
      carX[i] = -20;

    int laneY = 14 + (i * 12);

    if(
        chickenLane == i &&
        carX[i] > 14 &&
        carX[i] < 34
      )
    {
      state = CHICKEN_OVER;
    }
  }
}

void drawChicken()
{
  display.clearDisplay();

  if(!chickenStarted)
  {
    drawInfoScreen(
      "CHICKEN",
      chickenHigh
    );
    return;
  }

  for(int i=0;i<4;i++)
  {
    int laneY = 14 + (i * 12);

    display.drawLine(
      0,laneY+6,
      127,laneY+6,
      SSD1306_WHITE
    );

    display.fillRect(
      (int)carX[i],
      laneY,
      16,
      6,
      SSD1306_WHITE
    );
  }

  int chickenY =
    14 + (chickenLane * 12);

  display.fillCircle(
    20,
    chickenY+3,
    3,
    SSD1306_WHITE
  );

  display.setCursor(0,0);
  display.print(chickenScore);

  display.display();
}
// =====================================================
// PART 8 OF 12
// STACK BOX
// =====================================================

int towerX[20];
int towerW[20];

int towerLevel;

float moveX;
float moveSpeed;
int moveDir;

bool stackStarted;

int stackScore;

void initStack()
{
  towerLevel = 0;

  towerX[0] = 44;
  towerW[0] = 40;

  moveX = 0;
  moveSpeed = 1.5;

  moveDir = 1;

  stackStarted = false;

  stackScore = 0;
}

void updateStack()
{
  if(!stackStarted)
  {
    if(press32) state = MENU;
    if(press33) stackStarted = true;
    return;
  }

  moveX += moveSpeed * moveDir;

  if(moveX < 0)
  {
    moveX = 0;
    moveDir = 1;
  }

  if(moveX > 128 - towerW[towerLevel])
  {
    moveX = 128 - towerW[towerLevel];
    moveDir = -1;
  }

  if(press33)
  {
    int prevX = towerX[towerLevel];
    int prevW = towerW[towerLevel];

    int overlapStart =
      max((int)moveX, prevX);

    int overlapEnd =
      min((int)moveX + prevW,
          prevX + prevW);

    int newWidth =
      overlapEnd - overlapStart;

    if(newWidth <= 0)
    {
      state = STACK_OVER;
      return;
    }

    towerLevel++;

    if(towerLevel > 18)
      towerLevel = 18;

    towerX[towerLevel] = overlapStart;
    towerW[towerLevel] = newWidth;

    moveX = overlapStart;

    stackScore++;

    moveSpeed += 0.15;

    if(stackScore > stackHigh)
    {
      stackHigh = stackScore;
      saveScores();
    }
  }
}

void drawStack()
{
  display.clearDisplay();

  if(!stackStarted)
  {
    drawInfoScreen(
      "STACK",
      stackHigh
    );
    return;
  }

  for(int i=0;i<=towerLevel;i++)
  {
    int y =
      62 - (i * 3);

    display.fillRect(
      towerX[i],
      y,
      towerW[i],
      3,
      SSD1306_WHITE
    );
  }

  int moveY =
    62 - ((towerLevel + 1) * 3);

  display.drawRect(
    (int)moveX,
    moveY,
    towerW[towerLevel],
    3,
    SSD1306_WHITE
  );

  display.setCursor(0,0);
  display.print(stackScore);

  display.display();
}
// =====================================================
// PART 9 OF 12
// GAME OVER SCREENS
// =====================================================

void drawGameOver(
  const char* title,
  int score,
  int high
)
{
  display.clearDisplay();

  display.setTextSize(1);

  display.setCursor(28,6);
  display.print(title);

  display.setCursor(26,20);
  display.print("GAME OVER");

  display.setCursor(18,36);
  display.print("SCORE:");
  display.print(score);

  display.setCursor(18,48);
  display.print("HIGH:");
  display.print(high);

  display.setCursor(0,58);
  display.print("32=MENU 33=RETRY");

  display.display();
}
// =====================================================
// PART 10 OF 12
// PIXEL SPRITES
// =====================================================

// --------------------
// DINO SPRITE
// --------------------

void drawDinoSprite(
  int x,
  int y,
  bool frame
)
{
  display.fillRect(x+2,y,8,5,SSD1306_WHITE);

  display.fillRect(x,y+5,10,7,SSD1306_WHITE);

  if(frame)
  {
    display.fillRect(
      x+1,y+12,
      2,3,
      SSD1306_WHITE
    );

    display.fillRect(
      x+7,y+12,
      2,3,
      SSD1306_WHITE
    );
  }
  else
  {
    display.fillRect(
      x+1,y+11,
      2,2,
      SSD1306_WHITE
    );

    display.fillRect(
      x+7,y+13,
      2,2,
      SSD1306_WHITE
    );
  }
}

// --------------------
// BIRD SPRITE
// --------------------

void drawBirdSprite(
  int x,
  int y
)
{
  display.drawRect(
    x,
    y,
    10,
    5,
    SSD1306_WHITE
  );

  display.drawLine(
    x,
    y,
    x-3,
    y-2,
    SSD1306_WHITE
  );

  display.drawLine(
    x+10,
    y,
    x+13,
    y-2,
    SSD1306_WHITE
  );
}

// --------------------
// CHICKEN SPRITE
// --------------------

void drawChickenSprite(
  int x,
  int y
)
{
  display.fillCircle(
    x,
    y,
    3,
    SSD1306_WHITE
  );

  display.drawPixel(
    x+3,
    y,
    SSD1306_WHITE
  );

  display.drawPixel(
    x-1,
    y+4,
    SSD1306_WHITE
  );

  display.drawPixel(
    x+1,
    y+4,
    SSD1306_WHITE
  );
}

// --------------------
// CAR SPRITE
// --------------------

void drawCarSprite(
  int x,
  int y
)
{
  display.fillRect(
    x,
    y,
    16,
    6,
    SSD1306_WHITE
  );

  display.drawPixel(
    x+2,
    y+6,
    SSD1306_WHITE
  );

  display.drawPixel(
    x+13,
    y+6,
    SSD1306_WHITE
  );
}

// --------------------
// CLOUD
// --------------------

void drawCloud(
  int x,
  int y
)
{
  display.drawCircle(
    x,
    y,
    3,
    SSD1306_WHITE
  );

  display.drawCircle(
    x+5,
    y-1,
    4,
    SSD1306_WHITE
  );

  display.drawCircle(
    x+10,
    y,
    3,
    SSD1306_WHITE
  );
}

// --------------------
// STAR
// --------------------

void drawStar(
  int x,
  int y
)
{
  display.drawPixel(
    x,
    y,
    SSD1306_WHITE
  );

  display.drawPixel(
    x-1,
    y,
    SSD1306_WHITE
  );

  display.drawPixel(
    x+1,
    y,
    SSD1306_WHITE
  );

  display.drawPixel(
    x,
    y-1,
    SSD1306_WHITE
  );

  display.drawPixel(
    x,
    y+1,
    SSD1306_WHITE
  );
}
// =====================================================
// PART 11 OF 12
// SETUP
// =====================================================

void setup()
{
  pinMode(BTN32, INPUT_PULLUP);
  pinMode(BTN33, INPUT_PULLUP);

  Wire.begin(21, 22);

  display.begin(
    SSD1306_SWITCHCAPVCC,
    OLED_ADDR
  );

  display.clearDisplay();
  display.display();

  randomSeed(micros());

  loadScores();

  splashStart = millis();

  state = SPLASH;
}
// =====================================================
// PART 12 OF 12
// MAIN LOOP
// =====================================================

void loop()
{
  readButtons();

  switch(state)
  {
    // ----------------
    // SPLASH
    // ----------------

    case SPLASH:
    {
      drawSplash();

      if(millis() - splashStart > 2000)
      {
        state = MENU;
      }
    }
    break;

    // ----------------
    // MENU
    // ----------------

    case MENU:
    {
      updateMenu();
      drawMenu();
    }
    break;

    // ----------------
    // PONG
    // ----------------

    case PONG_INFO:
    {
      initPong();
      state = PONG_GAME;
    }
    break;

    case PONG_GAME:
    {
      updatePong();
      drawPong();
    }
    break;

    case PONG_OVER:
    {
      drawGameOver(
        "PONG",
        pongScore,
        pongHigh
      );

      if(press32)
        state = MENU;

      if(press33)
      {
        initPong();
        state = PONG_GAME;
      }
    }
    break;

    // ----------------
    // DINO
    // ----------------

    case DINO_INFO:
    {
      initDino();
      state = DINO_GAME;
    }
    break;

    case DINO_GAME:
    {
      updateDino();
      drawDino();
    }
    break;

    case DINO_OVER:
    {
      drawGameOver(
        "DINO",
        dinoScore,
        dinoHigh
      );

      if(press32)
        state = MENU;

      if(press33)
      {
        initDino();
        state = DINO_GAME;
      }
    }
    break;

    // ----------------
    // FLAPPY
    // ----------------

    case FLAPPY_INFO:
    {
      initFlappy();
      state = FLAPPY_GAME;
    }
    break;

    case FLAPPY_GAME:
    {
      updateFlappy();
      drawFlappy();
    }
    break;

    case FLAPPY_OVER:
    {
      drawGameOver(
        "FLAPPY",
        flappyScore,
        flappyHigh
      );

      if(press32)
        state = MENU;

      if(press33)
      {
        initFlappy();
        state = FLAPPY_GAME;
      }
    }
    break;

    // ----------------
    // CHICKEN
    // ----------------

    case CHICKEN_INFO:
    {
      initChicken();
      state = CHICKEN_GAME;
    }
    break;

    case CHICKEN_GAME:
    {
      updateChicken();
      drawChicken();
    }
    break;

    case CHICKEN_OVER:
    {
      drawGameOver(
        "CHICKEN",
        chickenScore,
        chickenHigh
      );

      if(press32)
        state = MENU;

      if(press33)
      {
        initChicken();
        state = CHICKEN_GAME;
      }
    }
    break;

    // ----------------
    // STACK
    // ----------------

    case STACK_INFO:
    {
      initStack();
      state = STACK_GAME;
    }
    break;

    case STACK_GAME:
    {
      updateStack();
      drawStack();
    }
    break;

    case STACK_OVER:
    {
      drawGameOver(
        "STACK",
        stackScore,
        stackHigh
      );

      if(press32)
        state = MENU;

      if(press33)
      {
        initStack();
        state = STACK_GAME;
      }
    }
    break;
  }

  delay(16); // ~60 FPS
}
