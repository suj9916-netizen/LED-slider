# p5.js LED PWM Slider

## 1. 프로젝트 개요

p5.js의 슬라이더를 이용하여 Arduino에 연결된 LED의 밝기를 조절하는 프로젝트이다.

웹 화면에서 슬라이더 값을 `0 ~ 255` 사이로 조절하면 해당 값이 Serial 통신을 통해 Arduino로 전달되고, Arduino는 PWM 출력을 이용하여 LED의 밝기를 변경한다.

또한 Arduino의 A0 핀에 연결된 가변저항 값을 `0 ~ 1023` 범위로 읽어 p5.js 화면에 실시간으로 표시하도록 구현하였다.

---

## 2. 동작 방식

### LED 밝기 조절

p5.js의 슬라이더 값은 `0 ~ 255` 범위로 설정하였다.

```text
0   → LED OFF
128 → 중간 정도 밝기
255 → 최대 밝기
```

슬라이더를 움직이면 p5.js에서 다음과 같은 문자열을 Arduino로 전송한다.

```text
PWM:128
```

Arduino는 `PWM:` 뒤의 값을 읽고 `analogWrite()`를 이용하여 LED의 밝기를 조절한다.

---

### 가변저항 값 표시

가변저항은 Arduino의 `A0` 핀에 연결하였다.

Arduino에서 `analogRead()`를 이용하여 `0 ~ 1023` 사이의 값을 읽고 다음 형식으로 p5.js에 전송한다.

```text
POT:523
```

p5.js에서는 받은 값을 화면의 막대와 숫자로 표시한다.

---

## 3. 회로 연결

### LED

```text
Arduino D9 → 저항 → LED (+)
LED (-) → GND


```
<img width="719" height="430" alt="111111" src="https://github.com/user-attachments/assets/1c4c1d59-08f8-485f-b807-d694ffe5aaae" />
D9는 PWM 출력이 가능한 핀이므로 LED의 밝기를 조절할 수 있다.

### 가변저항

```text
한쪽 핀   → 5V
가운데 핀 → A0
반대쪽 핀 → GND
```

---

## 4. Arduino 코드

```cpp
const int LED = 9;
const int POT = A0;

int lastPot = -10;

void setup() {
  Serial.begin(9600);
  Serial.setTimeout(10);

  pinMode(LED, OUTPUT);
}

void loop() {
  int potValue = analogRead(POT);

  // 값이 너무 조금 흔들리는 것은 무시
  if (abs(potValue - lastPot) >= 3) {
    Serial.print("POT:");
    Serial.println(potValue);

    lastPot = potValue;
  }

  // p5.js에서 PWM 값 받기
  if (Serial.available()) {
    String msg = Serial.readStringUntil('\n');
    msg.trim();

    if (msg.startsWith("PWM:")) {
      int value = msg.substring(4).toInt();

      value = constrain(value, 0, 255);

      analogWrite(LED, value);
    }
  }

  delay(10);
}
```

---

## 5. p5.js 코드

```javascript
let serial, connectButton;
let brightnessSlider;

let potValue = 0;
let brightness = 0;

let statusText = 'Arduino를 연결해 주세요.';

function setup() {
  createCanvas(600, 400);
  textFont('sans-serif');

  serial = new SerialHelper(
    9600,
    receiveLine,
    message => statusText = message
  );

  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(() => serial.toggle());

  brightnessSlider = createSlider(0, 255, 0, 1);
  brightnessSlider.position(190, 270);
  brightnessSlider.size(220);

  brightnessSlider.input(async () => {
    brightness = brightnessSlider.value();

    await serial.send('PWM:' + brightness);
  });
}

function draw() {
  background(242, 246, 250);

  fill(25, 40, 60);
  noStroke();
  textAlign(CENTER);

  textSize(24);
  text('Arduino LED Brightness', width / 2, 40);

  textSize(14);
  text(statusText, width / 2, 70);

  // 가변저항 값
  textSize(16);
  text('가변저항 A0', width / 2, 110);

  fill('#dbe3ed');
  rect(190, 130, 220, 40, 10);

  fill('#3b82f6');
  rect(
    190,
    130,
    map(potValue, 0, 1023, 0, 220),
    40,
    10
  );

  fill(25, 40, 60);
  textSize(20);
  text(potValue + ' / 1023', width / 2, 205);

  // LED 밝기
  textSize(16);
  text('LED 밝기 조절', width / 2, 250);

  textSize(18);
  text(brightness + ' / 255', width / 2, 325);

  textSize(13);
  text('0 = OFF     255 = 최대 밝기', width / 2, 355);
}

function receiveLine(line) {
  const message = line.trim().toUpperCase();

  if (message.startsWith('POT:')) {
    let value = Number(message.substring(4));

    if (!isNaN(value)) {
      potValue = constrain(value, 0, 1023);
    }
  }
}
```

---

## 6. 코드 설명

p5.js에서 슬라이더 값이 바뀌면 다음 코드를 이용하여 Arduino로 값을 전달한다.

```javascript
await serial.send('PWM:' + brightness);
```

예를 들어 슬라이더 값이 `150`이면

```text
PWM:150
```

이 Arduino로 전송된다.

Arduino에서는 다음 코드를 이용하여 `PWM:` 뒤의 숫자만 가져온다.

```cpp
int value = msg.substring(4).toInt();
```

이 값을 `analogWrite()`에 사용하여 LED의 밝기를 변경한다.

```cpp
analogWrite(LED, value);
```

가변저항은 `analogRead(A0)`를 이용하여 읽고 p5.js에 전달한다.

```cpp
Serial.print("POT:");
Serial.println(potValue);
```

---

## 7. 결과



https://github.com/user-attachments/assets/75ae3bd6-1661-49f0-9f7e-34f79af74b5e





## 8. 고찰

처음에는 p5.js에서 슬라이더 값만 보내면 LED 밝기가 바로 변할 것이라고 생각했지만, Arduino에서도 전달받은 값을 처리해서 PWM 출력으로 바꿔주는 코드가 필요했다.

처음에는 `delay()` 값을 크게 설정하여 슬라이더와 가변저항의 반응이 느리게 느껴졌다. 이후 `delay(10)`으로 줄여서 화면과 LED가 더 빠르게 반응하도록 수정하였다.

이번 실습을 통해 p5.js와 Arduino가 각각 따로 동작하는 것이 아니라 Serial 통신을 통해 데이터를 주고받으면서 하나의 시스템처럼 동작한다는 것을 이해할 수 있었다.
