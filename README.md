# Trabajo-Final_Vision-Artificial_Planetarium

### Ideación de la experiencia
Esta idea surge a partir de la materia de Modelamiento Matemático, en la cual se nos pidió elegir un tema y aplicarlo dentro del contexto de nuestra carrera.
El tema seleccionado fue el sistema solar, y a partir de esto decidí integrar los conceptos aprendidos con una propuesta interactiva desarrollada en p5.js utilizando handPose de la librería ml5.js.

Me pareció sumamente interesante crear una experiencia en la que las personas puedan interactuar con el sistema solar mediante simples gestos de la mano: cambiar de planeta, visualizar una breve descripción informativa y observar la composición completa del sistema.
Este proyecto busca fusionar la tecnología, el arte digital y la educación, ofreciendo una forma intuitiva, lúdica y visualmente atractiva de explorar el universo.

___________________________________________________________________________________________________________________________________________________________________________

### Concepto Artístico
El proyecto propone una visualización educativa e inmersiva del sistema solar, controlada completamente mediante gestos manuales detectados por la cámara.
El usuario puede:

Observar el sistema solar completo en modo idle.

“Entrar” a un planeta cerrando el puño ✊.

Cambiar de planeta moviendo la mano abierta horizontalmente ✋.

Mostrar información del planeta levantando solo el dedo índice ☝️.

El gesto se convierte en un lenguaje simbólico que reemplaza el mouse o teclado, fomentando una experiencia más orgánica, lúdica y corporal.

____________________________________________________________________________________________________________________________________________________________________________

### Propuesta Estética y Narrativa
La estética combina una visualización cósmica minimalista con elementos de retro-futurismo digital, evocando interfaces espaciales holográficas.
Los colores, transparencias y efectos luminosos transmiten una sensación de exploración científica pero poética, donde el gesto humano activa el conocimiento.

____________________________________________________________________________________________________________________________________________________________________________

### Referentes
La verdad en este proyecto no me base en muchos referentes de internet pero hubo un referente que me ayudo a concretar lo que queria en mi proyecto, este referente fueron mis compañeros de taller 6 donde se encargaron de realizar una experiencia increible con el handpose, asi que gracias a ellos fui capaz de concretar este proyecto en su totalidad.

____________________________________________________________________________________________________________________________________________________________________________

### Diseño en Papel
Aunque no se realizaron bocetos ni wireframes tradicionales, se desarrolló un prototipo funcional inicial en p5.js que utilizaba la detección de pinza (pinch) como mecanismo de interacción principal.

Este primer código permitió experimentar con el control de escala y rotación del sistema solar mediante gestos de la mano, sirviendo como prueba conceptual para evaluar la viabilidad técnica y el comportamiento del modelo de visión artificial (handPose).

A partir de esta exploración inicial, se decidió evolucionar el proyecto hacia un sistema de control más natural y expresivo basado en gestos completos de la mano, reemplazando el pinch por poses más reconocibles: puño cerrado, mano abierta, y dedo índice levantado.

<img width="724" height="544" alt="image" src="https://github.com/user-attachments/assets/9ccde33f-228d-48c1-a0c3-4a7c38715871" />

```js
let handPose;
let video;
let hands = [];

let pinchThreshold = 20;
let isPinching = false;
let lastPinchState = false;

let showMiniCam = false;
let mode = "idle"; // "idle" (sistema solar) o "individual"
let transitioning = false;
let fadeAlpha = 0;
let fadeDuration = 0.6 * 60; // 0.6 s a 60 FPS
let fadeFrame = 0;

// ---- Sistema Solar ----
let sunPulse = 0;
let stars = [];
let planets = [];
let particles = [];

// ---- Planetas (modo individual) ----
let currentPlanet = 0;
let nextPlanet = 0;
let sphereX;
let targetX;
let animating = false;

function preload() {
  handPose = ml5.handPose();
}

function setup() {
  createCanvas(640, 480);
  video = createCapture(VIDEO);
  video.size(640, 480);
  video.hide();
  handPose.detectStart(video, gotHands);

  // Fondo de estrellas
  for (let i = 0; i < 150; i++) {
    stars.push({
      x: random(width),
      y: random(height),
      brightness: random(150, 255),
    });
  }

  // Definir planetas
  planets = [
    { name: "Mercurio", color: "#b1a181", dist: 60, size: 8, speed: 0.035 },
    { name: "Venus", color: "#e0c16c", dist: 90, size: 12, speed: 0.028 },
    { name: "Tierra", color: "#2e83f2", dist: 120, size: 13, speed: 0.022 },
    { name: "Marte", color: "#d14b28", dist: 150, size: 10, speed: 0.018 },
    { name: "Júpiter", color: "#d7b48a", dist: 190, size: 22, speed: 0.012 },
    { name: "Saturno", color: "#e9d19b", dist: 230, size: 20, speed: 0.009, ring: true },
    { name: "Urano", color: "#76d7ea", dist: 270, size: 15, speed: 0.007, ring: true },
    { name: "Neptuno", color: "#466df5", dist: 300, size: 14, speed: 0.005 },
  ];
  planets.forEach(p => (p.angle = random(TWO_PI)));

  // Partículas de estela
  for (let i = 0; i < 60; i++) {
    particles.push({
      angle: random(TWO_PI),
      dist: random(120, 160),
      speed: random(0.002, 0.008),
    });
  }

  sphereX = width / 2;
  targetX = sphereX;
}

function draw() {
  background(0);

  // --- Efecto espejo ---
  push();
  translate(width, 0);
  scale(-1, 1);

  // Fondo de estrellas
  drawStars();

  // Modo según estado
  if (mode === "idle") drawSolarSystem();
  else drawIndividualMode();

  if (showMiniCam) {
    push();
    translate(160, height - 120);
    image(video, 0, 0, 160, 120);
    pop();
  }

  pop(); // fin espejo

  // Fade transition
  if (transitioning) {
    fadeFrame++;
    let t = fadeFrame / fadeDuration;
    fadeAlpha = (mode === "transitionOut") ? lerp(0, 255, t) : lerp(255, 0, t);
    fill(0, fadeAlpha);
    noStroke();
    rect(0, 0, width, height);

    if (fadeFrame >= fadeDuration) {
      if (mode === "transitionOut") {
        mode = "individual";
        transitioning = false;
      } else if (mode === "transitionIn") {
        mode = "idle";
        transitioning = false;
      }
    }
  }

  // Texto informativo
  drawHUD();
}

// =======================
// 🔭 MODO IDLE (SISTEMA SOLAR)
// =======================
function drawSolarSystem() {
  translate(width / 2, height / 2);

  // 🌞 Sol pulsante
  let pulse = (sin(frameCount * 0.05) + 1) * 0.5;
  noStroke();
  fill(255, 200, 0, 180);
  ellipse(0, 0, 80 + pulse * 10, 80 + pulse * 10);
  fill(255, 230, 100);
  ellipse(0, 0, 40, 40);

  // 🪐 Órbitas + planetas
  for (let p of planets) {
    noFill();
    stroke(255, 40);
    ellipse(0, 0, p.dist * 2);

    let x = cos(p.angle) * p.dist;
    let y = sin(p.angle) * p.dist * 0.8;

    // Estela luminosa (simple transparencia)
    fill(p.color + "33");
    noStroke();
    ellipse(x - 4, y - 2, p.size * 1.8, p.size * 1.2);

    // Planeta
    fill(p.color);
    noStroke();
    ellipse(x, y, p.size * 2);

    // Movimiento orbital
    if (!transitioning && mode === "idle") p.angle += p.speed;

    // Anillo (si tiene)
    if (p.ring) {
      noFill();
      stroke(200, 120);
      strokeWeight(1);
      ellipse(x, y, p.size * 3, p.size * 1.2);
    }
  }
}

// =======================
// 🌍 MODO INDIVIDUAL
// =======================
function drawIndividualMode() {
  sphereX = lerp(sphereX, targetX, 0.1);
  let current = planets[currentPlanet];
  let next = planets[nextPlanet];

  drawPlanetDecorated(current, sphereX, height / 2, 100);

  if (animating) {
    drawPlanetDecorated(next, sphereX - width, height / 2, 100);
  }

  if (animating && abs(sphereX - targetX) < 5) {
    animating = false;
    currentPlanet = nextPlanet;
    sphereX = width / 2;
    targetX = width / 2;
  }

  // Detección de manos
  if (hands.length > 0) {
    let hand = hands[0];
    drawConnections(hand.keypoints);
    drawKeypoints(hand.keypoints);
    let thumbTip = hand.keypoints[4];
    let indexTip = hand.keypoints[8];
    if (thumbTip && indexTip) {
      let d = dist(thumbTip.x, thumbTip.y, indexTip.x, indexTip.y);
      isPinching = d < pinchThreshold;
      if (isPinching && !lastPinchState && !animating) {
        nextPlanet = (currentPlanet + 1) % planets.length;
        targetX = width * 1.5;
        animating = true;
      }
      lastPinchState = isPinching;
    }
  }

  // Nombre planeta
  fill(255);
  noStroke();
  textAlign(CENTER);
  textSize(22);
  text(planets[currentPlanet].name, width / 2, height / 2 + 150);
}

// =======================
// ✨ FUNCIONES AUXILIARES
// =======================
function drawStars() {
  noStroke();
  for (let star of stars) {
    fill(star.brightness);
    circle(star.x, star.y, random(1, 3));
    star.brightness += random(-2, 2);
    star.brightness = constrain(star.brightness, 120, 255);
  }
}

function drawPlanetDecorated(planet, x, y, r) {
  push();
  translate(x, y);

  if (planet.ring) {
    noFill();
    stroke(200, 150);
    strokeWeight(6);
    ellipse(0, 0, r * 2.2, r * 0.8);
  }

  noStroke();
  fill(planet.color);
  circle(0, 0, r * 2);

  for (let i = 0; i < 20; i++) {
    fill(0, 10);
    ellipse(r * 0.2, r * 0.2, r * 2 - i * 5, r * 2 - i * 5);
  }

  fill(255, 40);
  ellipse(-r * 0.3, -r * 0.3, r * 0.8, r * 0.5);

  noStroke();
  fill(255, 180);
  for (let p of particles) {
    let px = cos(p.angle) * p.dist;
    let py = sin(p.angle) * p.dist * 0.6;
    circle(px, py, 2);
    p.angle += p.speed;
  }

  pop();
}

function drawKeypoints(keypoints) {
  fill(0, 255, 0);
  noStroke();
  for (let kp of keypoints) circle(kp.x, kp.y, 8);
}

function drawConnections(keypoints) {
  stroke(255, 0, 255);
  strokeWeight(2);
  let fingers = [
    [0, 1, 2, 3, 4],
    [0, 5, 6, 7, 8],
    [0, 9, 10, 11, 12],
    [0, 13, 14, 15, 16],
    [0, 17, 18, 19, 20],
  ];
  for (let finger of fingers) {
    for (let i = 0; i < finger.length - 1; i++) {
      let p1 = keypoints[finger[i]];
      let p2 = keypoints[finger[i + 1]];
      line(p1.x, p1.y, p2.x, p2.y);
    }
  }
}

function gotHands(results) {
  hands = results;
}

function drawHUD() {
  fill(255);
  noStroke();
  textSize(16);
  textAlign(LEFT);
  text("Presiona 'm' para cámara", 15, height - 40);
  text("Presiona 'l' para cambiar modo", 15, height - 20);
}

function keyPressed() {
  if (key === "m" || key === "M") showMiniCam = !showMiniCam;
  if (key === "l" || key === "L") {
    if (!transitioning) {
      transitioning = true;
      fadeFrame = 0;
      fadeAlpha = 0;
      mode = (mode === "idle") ? "transitionOut" : "transitionIn";
    }
  }
}

```

____________________________________________________________________________________________________________________________________________________________________________

### Planificación Técnica
- **Técnica seleccionada:** HandPose (ml5.js), para detección de poses y gestos de la mano en tiempo real.
- **Gestos definidos:**
- 👊 Puño cerrado: alterna entre modo general e individual.
- ✋ Mano abierta horizontal: cambia de planeta (izquierda o derecha según orientación).
- ☝️ Índice levantado: muestra u oculta el panel informativo.
- **Ubicación de la cámara:** superior frontal, orientada hacia el usuario, garantizando una detección estable del rostro y las manos.
- **Área de detección:** región central del cuadro de video, donde se espera la mayor precisión de handPose.
- **Lógica de procesamiento:** detección de la pose → clasificación del gesto → actualización del modo de visualización → respuesta visual animada en el sistema solar.

____________________________________________________________________________________________________________________________________________________________________________

### Implementación del Sistema de Detección
En esta parte del proyecto empecé a conectar todo lo que había planeado. Primero configuré la cámara con p5.js y luego integré el modelo HandPose de ml5.js, que es el encargado de detectar los movimientos y posiciones de la mano en tiempo real.
Tuve que hacer varias pruebas para calibrar bien el modelo, porque al principio detectaba gestos incluso cuando no los estaba haciendo. Fui ajustando los valores hasta lograr que reconociera solo cuando realmente movía la mano de cierta forma.

De todos los gestos que probé, me quedé con tres principales que son los que definen la interacción:

- 👊 Puño cerrado: sirve para cambiar entre el modo general del sistema solar y el modo donde se ve un planeta individual.

- ✋ Mano abierta horizontal: si la mano apunta hacia la derecha, cambia al siguiente planeta; si apunta hacia la izquierda, vuelve al anterior.

- ☝️ Dedo índice levantado: muestra o esconde el panel con la información del planeta que estás viendo.

Con eso ya tenía la base para que el sistema entendiera los gestos y reaccionara a ellos.

____________________________________________________________________________________________________________________________________________________________________________

### Desarrollo del Contenido Generativo

Después de tener la detección funcionando, me enfoqué en la parte visual.
Programé todo el sistema solar en p5.js, haciendo que cada planeta tuviera su propio tamaño, color, velocidad y distancia con respecto al sol. La idea era que se viera bien estéticamente, pero también que tuviera cierta coherencia con la realidad.

Aquí fue donde conecté todo con los gestos:
cuando cierro el puño, el sistema pasa de mostrar todos los planetas a enfocarse solo en uno;
cuando abro la mano hacia un lado, el sistema cambia de planeta;
y cuando levanto el dedo índice, aparece una pequeña ventana con información del planeta.

Me gustó mucho cómo se siente el resultado, porque aunque es un proyecto simple, la interacción se siente natural, como si realmente estuvieras “controlando” el sistema solar con tus manos.

____________________________________________________________________________________________________________________________________________________________________________

### Optimización y Pulimiento
En esta parte me dediqué a mejorar los detalles.
Al principio el sistema cambiaba de planeta muy rápido o sin que yo hiciera nada, así que tuve que ajustar el código para que solo reaccionara cuando el gesto estuviera bien definido.
También trabajé en que las animaciones fueran más suaves y las transiciones más fluidas, para que la experiencia se sintiera más limpia y profesional.

Hice varias pruebas con diferentes personas para ver si el sistema reconocía bien sus manos y si entendían cómo interactuar con los planetas. En general funcionó bien, aunque noté que la luz y la posición de la cámara influían bastante en la detección.

Finalmente, documenté todo el proceso, los cambios que fui haciendo y el código final ya comentado, para tener un registro completo de cómo evolucionó el proyecto.

____________________________________________________________________________________________________________________________________________________________________________

### Codigo Final
- Html
```js
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="utf-8" />
    <title>Sistema Solar Interactivo</title>
    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.10/lib/p5.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.10/lib/addons/p5.sound.min.js"></script>
   <script src="https://unpkg.com/ml5@1/dist/ml5.js"></script>
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <main></main>
    <script src="sketch.js"></script>
  </body>
</html>

```

- Sketch
```js
let handPose, video, hands = [];
let showMiniCam = false;

let mode = "idle"; 
let transitioning = false;
let fadeAlpha = 0, fadeDuration = 0.6 * 60, fadeFrame = 0;

let stars = [];
let planets = [];
let currentPlanet = 0, nextPlanet = 0;
let sphereX, targetX, animating = false;
let infoPanel, planetInfo = {};

let lastGesture = "";
let gestureCooldown = 30; 
let gestureTimer = 0;
let infoVisible = false;

//-------------------------------------------
// CLASE PLANETA
//-------------------------------------------
class Planet {
  constructor(name, color, dist, size, speed, ring = false) {
    this.name = name;
    this.color = color;
    this.dist = dist;     
    this.size = size;
    this.speed = speed;   
    this.ring = ring;
    this.angle = random(TWO_PI);
    this.k = this.speed * pow(this.dist, 1.5);
    this.initialDist = dist;
    this.initialAngle = this.angle;
  }

  update() {
    if (mode === "idle" && !transitioning) {
      let omega = this.k / pow(this.dist, 1.5);
      this.angle += omega;
    }
  }

  reset() {
    this.dist = this.initialDist;
    this.angle = this.initialAngle;
  }

  display() {
    noFill();
    stroke(255, 40);
    ellipse(0, 0, this.dist * 2, this.dist * 1.6);
    let x = cos(this.angle) * this.dist;
    let y = sin(this.angle) * this.dist * 0.8;

    fill(this.color + "33");
    noStroke();
    ellipse(x - 4, y - 2, this.size * 1.8, this.size * 1.2);

    fill(this.color);
    noStroke();
    circle(x, y, this.size * 2);

    if (this.ring) {
      noFill();
      stroke(200, 120);
      ellipse(x, y, this.size * 3, this.size * 1.2);
    }
  }
}

//-------------------------------------------
// SETUP
//-------------------------------------------
function preload() { handPose = ml5.handPose(); }

function setup() {
  createCanvas(640, 480);
  video = createCapture(VIDEO);
  video.size(640, 480);
  video.hide();
  handPose.detectStart(video, gotHands);

  for (let i = 0; i < 120; i++) {
    stars.push({ x: random(width), y: random(height), brightness: random(150, 255) });
  }

  planets = [
    new Planet("Mercurio", "#b1a181", 60, 8, 0.035),
    new Planet("Venus", "#e0c16c", 90, 12, 0.028),
    new Planet("Tierra", "#2e83f2", 120, 13, 0.022),
    new Planet("Marte", "#d14b28", 150, 10, 0.018),
    new Planet("Júpiter", "#d7b48a", 190, 22, 0.012),
    new Planet("Saturno", "#e9d19b", 230, 20, 0.009, true),
    new Planet("Urano", "#76d7ea", 270, 15, 0.007, true),
    new Planet("Neptuno", "#466df5", 300, 14, 0.005)
  ];

  planetInfo = {
    Mercurio: { fact: "El más cercano al Sol." },
    Venus: { fact: "Su atmósfera es la más caliente." },
    Tierra: { fact: "Nuestro hogar azul." },
    Marte: { fact: "El planeta rojo." },
    Júpiter: { fact: "El gigante gaseoso." },
    Saturno: { fact: "Famoso por sus anillos." },
    Urano: { fact: "Gira de lado." },
    Neptuno: { fact: "El más lejano y azul." }
  };

  sphereX = width / 2;
  targetX = sphereX;
  createInfoPanel();
}

//-------------------------------------------
// DRAW
//-------------------------------------------
function draw() {
  background(0);
  push();
  translate(width, 0);
  scale(-1, 1);
  drawStars();

  if (mode === "idle") drawSolarSystem();
  else drawIndividualMode();

  drawHand();
  if (showMiniCam) {
    push();
    translate(width - 170, height - 130);
    image(video, 0, 0, 160, 120);
    pop();
  }
  pop();

  fadeTransition();
  drawHUD();

  if (gestureTimer > 0) gestureTimer--;
  detectGestures();
}

//-------------------------------------------
// VISUAL
//-------------------------------------------
function drawSolarSystem() {
  translate(width / 2, height / 2);
  noStroke();
  for (let r = 60; r > 0; r -= 10) {
    fill(255, 200, 0, map(r, 60, 0, 0, 180));
    circle(0, 0, r * 2);
  }
  for (let p of planets) {
    p.update();
    p.display();
  }
}

function drawIndividualMode() {
  sphereX = lerp(sphereX, targetX, 0.1);
  let current = planets[currentPlanet];
  let next = planets[nextPlanet];
  drawPlanetDecorated(current, sphereX, height / 2, 100);
  if (infoVisible) updateInfoPanel(current.name);
  if (animating) drawPlanetDecorated(next, sphereX - width, height / 2, 100);
  if (animating && abs(sphereX - targetX) < 5) {
    animating = false;
    currentPlanet = nextPlanet;
    sphereX = width / 2;
    targetX = width / 2;
  }
  drawPlanetName(planets[currentPlanet].name);
}

function drawPlanetDecorated(p, x, y, r) {
  push();
  translate(x, y);
  if (p.ring) {
    noFill();
    stroke(150, 200);
    strokeWeight(6);
    ellipse(0, 0, r * 2.2, r * 0.8);
  }
  noStroke();
  fill(p.color);
  circle(0, 0, r * 2);
  fill(255, 40);
  ellipse(-r * 0.3, -r * 0.3, r * 0.8, r * 0.5);
  pop();
}

function drawStars() {
  noStroke();
  for (let s of stars) {
    fill(s.brightness);
    circle(s.x, s.y, random(1, 3));
    s.brightness = constrain(s.brightness + random(-2, 2), 120, 255);
  }
}

function fadeTransition() {
  if (transitioning) {
    fadeFrame++;
    let t = fadeFrame / fadeDuration;
    fadeAlpha = mode === "transitionOut" ? lerp(0, 255, t) : lerp(255, 0, t);
    fill(0, fadeAlpha);
    rect(0, 0, width, height);
    if (fadeFrame >= fadeDuration) {
      if (mode === "transitionOut") {
        mode = "individual";
        transitioning = false;
      } else if (mode === "transitionIn") {
        mode = "idle";
        transitioning = false;
      }
    }
  }
}

function drawHUD() {
  fill(255);
  textSize(13);
  textAlign(LEFT);
  text("✊: cambiar modo | ✋ mano horizontal: cambiar planeta | ☝️ índice arriba: info", 15, height - 20);
}

function drawPlanetName(name) {
  push();
  scale(-1, 1);
  fill(255);
  textAlign(CENTER);
  textSize(22);
  text(name, -width / 2, height / 2 + 150);
  pop();
}

//-------------------------------------------
// HAND DETECTION
//-------------------------------------------
function gotHands(results) { hands = results; }

function drawHand() {
  if (hands.length > 0) {
    let hand = hands[0];
    let kp = hand.keypoints;
    stroke(255, 0, 255);
    strokeWeight(2);
    noFill();
    for (let i = 0; i < kp.length; i++) circle(kp[i].x, kp[i].y, 6);
  }
}

//-------------------------------------------
// GESTURE DETECTION
//-------------------------------------------
function detectGestures() {
  if (hands.length === 0 || gestureTimer > 0) return;
  let hand = hands[0];
  let kp = hand.keypoints;
  let palm = kp[0]; 
  let thumbTip = kp[4];
  let indexTip = kp[8];
  let middleTip = kp[12];
  let ringTip = kp[16];
  let pinkyTip = kp[20];

  let tips = [indexTip, middleTip, ringTip, pinkyTip];
  let avgDist = tips.map(t => dist(t.x, t.y, palm.x, palm.y));
  let avg = avgDist.reduce((a,b)=>a+b)/avgDist.length;

  let gesture = "";

  // ✊ PUÑO
  if (avg < 50) gesture = "fist";
  // ✋ MANO ABIERTA
  else if (avg > 120) gesture = "open";
  // ☝️ SOLO ÍNDICE ARRIBA
  else if (isIndexOnlyUp(kp)) gesture = "indexUp";

  if (gesture !== lastGesture) {
    gestureTimer = gestureCooldown;
    lastGesture = gesture;

    if (gesture === "fist") toggleMode();
    else if (gesture === "open") detectHorizontalMovement(hand);
    else if (gesture === "indexUp" && mode === "individual") toggleInfoPanel();
  }
}

function isIndexOnlyUp(kp) {
  let palm = kp[0];
  let indexTip = kp[8];
  let middleTip = kp[12];
  let ringTip = kp[16];
  let pinkyTip = kp[20];

  let indexDist = dist(indexTip.x, indexTip.y, palm.x, palm.y);
  let others = [middleTip, ringTip, pinkyTip].map(t => dist(t.x, t.y, palm.x, palm.y));
  let avgOthers = others.reduce((a,b)=>a+b)/others.length;
  return indexDist > 100 && avgOthers < 70;
}

function toggleMode() {
  if (!transitioning) {
    transitioning = true;
    fadeFrame = 0;
    fadeAlpha = 0;
    mode = mode === "idle" ? "transitionOut" : "transitionIn";
    if (mode === "transitionIn") infoVisible = false;
  }
}

function detectHorizontalMovement(hand) {
  let wrist = hand.keypoints[0];
  let indexBase = hand.keypoints[5];
  let dir = indexBase.x - wrist.x;
  if (mode !== "individual" || animating) return;
  if (dir > 40) nextPlanetRight();
  else if (dir < -40) nextPlanetLeft();
}

function nextPlanetRight() {
  nextPlanet = (currentPlanet + 1) % planets.length;
  targetX = width * 1.5;
  animating = true;
}

function nextPlanetLeft() {
  nextPlanet = (currentPlanet - 1 + planets.length) % planets.length;
  targetX = width * 1.5;
  animating = true;
}

function toggleInfoPanel() {
  infoVisible = !infoVisible;
  showInfoPanel(infoVisible);
}

//-------------------------------------------
// PANEL INFO
//-------------------------------------------
function createInfoPanel() {
  infoPanel = createDiv("");
  infoPanel.style("position", "absolute");
  infoPanel.style("top", "20px");
  infoPanel.style("right", "20px");
  infoPanel.style("padding", "15px 18px");
  infoPanel.style("border", "2px solid #00ffffaa");
  infoPanel.style("border-radius", "12px");
  infoPanel.style("color", "#ccfaff");
  infoPanel.style("font-family", "'Share Tech Mono', monospace");
  infoPanel.style("font-size", "14px");
  infoPanel.style("background", "rgba(10,20,40,0.55)");
  infoPanel.style("box-shadow", "0 0 20px #00ffff88, inset 0 0 10px #00ffff44");
  infoPanel.style("backdrop-filter", "blur(4px)");
  infoPanel.style("opacity", "0");
  infoPanel.style("transition", "all 0.6s ease");
  infoPanel.hide();
}

function updateInfoPanel(name) {
  let d = planetInfo[name];
  if (!d) return;
  let html = `
    <div style='color:#00ffff;font-weight:bold;font-size:16px;'>${name}</div>
    <hr style='border:1px solid #00ffff44;margin:4px 0;'>
    🌐 <i>${d.fact}</i>
  `;
  infoPanel.html(html);
}

function showInfoPanel(show) {
  if (show) {
    infoPanel.show();
    setTimeout(() => infoPanel.style("opacity", "1"), 100);
  } else {
    infoPanel.style("opacity", "0");
    setTimeout(() => infoPanel.hide(), 500);
  }
}

```






























