# -I2C-Game-Loop-
ESP32 with led with botton

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

#define BTN_LEFT  32
#define BTN_RIGHT 33
#define BTN_SHOOT 25

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int SHIP_Y = 53;
const unsigned long FRAME_DELAY = 35;
const unsigned long SHOT_DELAY = 180;

int gameState = 1; // เริ่มเกมทันที
int score = 0;
int lives = 3;
int level = 1;

float shipX = 64;

unsigned long lastFrame = 0;
unsigned long lastShot = 0;
unsigned long lastSpawn = 0;
unsigned long lastBossShot = 0;

bool bossActive = false;
int bossX = 64;
int bossDir = 1;
int bossHP = 8;

struct Bullet {
  int x, y;
  bool active;
};

struct Enemy {
  float x, y;
  int type;
  bool active;
};

struct Star {
  int x, y, speed;
};

Bullet shots[6];
Bullet enemyShots[5];
Enemy enemies[7];
Star stars[18];

// อ่านปุ่มแบบ INPUT_PULLUP
bool pressed(int pin) {
  return digitalRead(pin) == LOW;
}

// เริ่มเกมใหม่
void resetGame() {
  score = 0;
  lives = 3;
  level = 1;
  shipX = 64;

  bossActive = false;
  bossHP = 8;
  bossX = 64;
  bossDir = 1;

  for (int i = 0; i < 6; i++)
    shots[i].active = false;

  for (int i = 0; i < 5; i++)
    enemyShots[i].active = false;

  for (int i = 0; i < 7; i++)
    enemies[i].active = false;

  lastShot = millis();
  lastSpawn = millis();
  lastBossShot = millis();
  lastFrame = millis();
  gameState = 1;
}

// วาดจรวด
void drawShip() {
  int x = (int)shipX;

  display.fillTriangle(
    x, SHIP_Y - 5,
    x - 5, SHIP_Y + 4,
    x + 5, SHIP_Y + 4,
    SSD1306_WHITE
  );

  display.drawPixel(x, SHIP_Y, SSD1306_BLACK);
  display.drawLine(x - 3, SHIP_Y + 2,
                   x - 7, SHIP_Y + 5, SSD1306_WHITE);
  display.drawLine(x + 3, SHIP_Y + 2,
                   x + 7, SHIP_Y + 5, SSD1306_WHITE);

  if (millis() % 200 < 100) {
    display.drawPixel(x, SHIP_Y + 7, SSD1306_WHITE);
    display.drawPixel(x - 1, SHIP_Y + 6, SSD1306_WHITE);
    display.drawPixel(x + 1, SHIP_Y + 6, SSD1306_WHITE);
  }
}

// คะแนน เลเวล และชีวิต
void drawHUD() {
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(0, 0);
  display.print("S:");
  display.print(score);

  display.setCursor(48, 0);
  display.print("LV:");
  display.print(level);

  display.setCursor(88, 0);
  display.print("HP:");
  for (int i = 0; i < lives; i++)
    display.print("<");

  display.drawLine(0, 10, 127, 10, SSD1306_WHITE);
}

// ดาวในอวกาศ
void updateStars() {
  for (int i = 0; i < 18; i++) {
    stars[i].y += stars[i].speed;

    if (stars[i].y > 63) {
      stars[i].y = 11;
      stars[i].x = random(0, 128);
    }

    display.drawPixel(stars[i].x, stars[i].y, SSD1306_WHITE);
  }
}

// ยิงกระสุน
void fireBullet() {
  if (millis() - lastShot < SHOT_DELAY) return;

  for (int i = 0; i < 6; i++) {
    if (!shots[i].active) {
      shots[i].x = (int)shipX;
      shots[i].y = SHIP_Y - 6;
      shots[i].active = true;
      lastShot = millis();
      break;
    }
  }
}

// อัปเดตกระสุน
void updateBullets() {
  for (int i = 0; i < 6; i++) {
    if (!shots[i].active) continue;

    shots[i].y -= 4;

    display.drawPixel(shots[i].x, shots[i].y, SSD1306_WHITE);
    display.drawPixel(shots[i].x, shots[i].y + 1, SSD1306_WHITE);

    if (shots[i].y < 11)
      shots[i].active = false;
  }

  for (int i = 0; i < 5; i++) {
    if (!enemyShots[i].active) continue;

    enemyShots[i].y += 2 + level / 3;

    display.drawPixel(
      enemyShots[i].x, enemyShots[i].y, SSD1306_WHITE
    );

    if (enemyShots[i].y > 63) {
      enemyShots[i].active = false;
      continue;
    }

    if (abs(enemyShots[i].x - (int)shipX) < 5 &&
        abs(enemyShots[i].y - SHIP_Y) < 5) {
      enemyShots[i].active = false;
      lives--;

      if (lives <= 0)
        gameState = 2;
    }
  }
}

// สร้างศัตรู
void spawnEnemy() {
  for (int i = 0; i < 7; i++) {
    if (!enemies[i].active) {
      enemies[i].x = random(5, 123);
      enemies[i].y = 14;
      enemies[i].type = random(0, 3);
      enemies[i].active = true;
      break;
    }
  }
}

// อัปเดตศัตรูและตรวจการชน
void updateEnemies() {
  for (int i = 0; i < 7; i++) {
    if (!enemies[i].active) continue;

    enemies[i].y += 0.7 + level * 0.22;

    int x = (int)enemies[i].x;
    int y = (int)enemies[i].y;

    if (enemies[i].type == 0) {
      display.drawCircle(x, y, 3, SSD1306_WHITE);
      display.drawPixel(x - 1, y, SSD1306_WHITE);
      display.drawPixel(x + 1, y, SSD1306_WHITE);
    } else if (enemies[i].type == 1) {
      display.drawTriangle(
        x, y - 3, x - 4, y + 3,
        x + 4, y + 3, SSD1306_WHITE
      );
    } else {
      display.drawRect(x - 3, y - 3, 7, 7, SSD1306_WHITE);
      display.drawPixel(x, y, SSD1306_WHITE);
    }

    // กระสุนผู้เล่นโดนศัตรู
    for (int j = 0; j < 6; j++) {
      if (!shots[j].active) continue;

      if (abs(shots[j].x - x) < 5 &&
          abs(shots[j].y - y) < 5) {
        shots[j].active = false;
        enemies[i].active = false;
        score += 10;
        break;
      }
    }

    // ชนจรวด
    if (enemies[i].active &&
        abs(x - (int)shipX) < 7 &&
        abs(y - SHIP_Y) < 7) {
      enemies[i].active = false;
      lives--;

      if (lives <= 0)
        gameState = 2;
    }

    // ศัตรูหลุดผ่านด้านล่าง
    if (enemies[i].active && enemies[i].y > 64) {
      enemies[i].active = false;
      lives--;

      if (lives <= 0)
        gameState = 2;
    }
  }
}

// บอส
void updateBoss() {
  if (score >= 100 && !bossActive) {
    bossActive = true;
    bossHP = 8;
  }

  if (!bossActive) return;

  bossX += bossDir;

  if (bossX > 116) bossDir = -1;
  if (bossX < 12) bossDir = 1;

  display.drawCircle(bossX, 22, 9, SSD1306_WHITE);
  display.drawCircle(bossX, 22, 5, SSD1306_WHITE);
  display.fillCircle(bossX, 22, 2, SSD1306_WHITE);

  display.setCursor(0, 12);
  display.print("BOSS:");
  display.print(bossHP);

  // ยิงบอส
  for (int i = 0; i < 6; i++) {
    if (!shots[i].active) continue;

    if (abs(shots[i].x - bossX) < 10 &&
        abs(shots[i].y - 22) < 10) {
      shots[i].active = false;
      bossHP--;

      if (bossHP <= 0) {
        bossActive = false;
        score += 50;
      }
    }
  }

  // บอสยิงกลับ
  if (millis() - lastBossShot > 800) {
    for (int i = 0; i < 5; i++) {
      if (!enemyShots[i].active) {
        enemyShots[i].x = bossX;
        enemyShots[i].y = 31;
        enemyShots[i].active = true;
        lastBossShot = millis();
        break;
      }
    }
  }
}

// หน้าจอ Game Over
void showGameOver() {
  display.clearDisplay();

  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(7, 8);
  display.println("GAME OVER");

  display.setTextSize(1);
  display.setCursor(28, 34);
  display.print("Score: ");
  display.println(score);

  display.setCursor(9, 51);
  display.println("SHOOT = RESTART");

  display.display();
}

// เริ่มต้นระบบ
void setup() {
  Serial.begin(115200);

  pinMode(BTN_LEFT, INPUT_PULLUP);
  pinMode(BTN_RIGHT, INPUT_PULLUP);
  pinMode(BTN_SHOOT, INPUT_PULLUP);

  Wire.begin(21, 22);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED not found!");
    while (true) delay(100);
  }

  display.clearDisplay();
  display.display();

  randomSeed(analogRead(34));

  for (int i = 0; i < 18; i++) {
    stars[i].x = random(0, 128);
    stars[i].y = random(11, 64);
    stars[i].speed = random(1, 3);
  }

  resetGame(); // เปิดเครื่องแล้วเล่นทันที
}

// ลูปเกมหลัก
void loop() {
  if (gameState == 2) {
    showGameOver();

    // ใช้ปุ่มยิงเริ่มใหม่
    if (pressed(BTN_SHOOT)) {
      delay(200);
      while (pressed(BTN_SHOOT)) delay(10);
      resetGame();
    }
    return;
  }

  if (millis() - lastFrame < FRAME_DELAY) return;
  lastFrame = millis();

  // เคลื่อนที่ซ้ายและขวา
  if (pressed(BTN_LEFT)) shipX -= 2.5;
  if (pressed(BTN_RIGHT)) shipX += 2.5;

  if (shipX < 8) shipX = 8;
  if (shipX > 120) shipX = 120;

  // กดยิงค้างเพื่อยิงต่อเนื่อง
  if (pressed(BTN_SHOOT)) fireBullet();

  // เพิ่มความยากทุก 50 คะแนน
  level = 1 + score / 50;
  if (level > 5) level = 5;

  unsigned long spawnDelay = 1000 - level * 120;
  if (spawnDelay < 350) spawnDelay = 350;

  if (millis() - lastSpawn > spawnDelay) {
    spawnEnemy();
    lastSpawn = millis();
  }

  display.clearDisplay();

  updateStars();
  drawHUD();
  updateBullets();
  updateEnemies();
  updateBoss();
  drawShip();

  display.display();

  Serial.print("Score: ");
  Serial.print(score);
  Serial.print(" Lives: ");
  Serial.print(lives);
  Serial.print(" Level: ");
  Serial.println(level);
}
