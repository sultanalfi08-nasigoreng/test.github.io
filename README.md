<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>Night Forest FPS with Flashlight & Chaser</title>
<style>
  body { margin: 0; overflow: hidden; }
  canvas { display: block; }

  #info {
    position: absolute;
    top: 10px;
    left: 10px;
    color: white;
    font-family: Arial;
    background: rgba(0,0,0,0.5);
    padding: 8px;
    border-radius: 4px;
    z-index: 10;
  }

  #staminaContainer {
    position: absolute;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    width: 200px;
    height: 16px;
    background: #333;
    border: 2px solid #000;
    z-index: 10;
  }

  #staminaBar {
    height: 100%;
    width: 100%;
    background: limegreen;
  }

  #warning {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: #ff4444;
    font-family: Arial;
    font-size: 24px;
    font-weight: bold;
    text-shadow: 0 0 10px #ff0000;
    opacity: 0;
    transition: opacity 0.3s;
    pointer-events: none;
    z-index: 10;
  }

  /* Game Over Screen */
  #gameOverScreen {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: #000;
    color: white;
    display: none;
    justify-content: center;
    align-items: center;
    flex-direction: column;
    z-index: 100;
    font-family: 'Arial', sans-serif;
    text-align: center;
  }

  #gameOverTitle {
    font-size: 72px;
    color: #ff0000;
    margin-bottom: 20px;
    text-shadow: 0 0 20px #ff0000;
    animation: pulse 1.5s infinite;
  }

  #gameOverText {
    font-size: 24px;
    margin-bottom: 40px;
    color: #ff6666;
  }

  #restartButton {
    background: #ff0000;
    color: white;
    border: none;
    padding: 15px 40px;
    font-size: 20px;
    cursor: pointer;
    border-radius: 5px;
    transition: background 0.3s, transform 0.2s;
    font-weight: bold;
    letter-spacing: 2px;
  }

  #restartButton:hover {
    background: #cc0000;
    transform: scale(1.05);
  }

  @keyframes pulse {
    0% { opacity: 0.3; }
    50% { opacity: 1; }
    100% { opacity: 0.3; }
  }

  /* Jumpscare Screen */
  #jumpscareScreen {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: #000;
    display: none;
    z-index: 200;
  }

  #jumpscareImage {
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0;
  }
</style>
</head>
<body>

<div id="info">
  Click to lock mouse<br>
  WASD move | Shift sprint | Mouse look
</div>

<div id="staminaContainer">
  <div id="staminaBar"></div>
</div>

<div id="warning">!! DANGER CLOSE !!</div>

<!-- Game Over Screen -->
<div id="gameOverScreen">
  <h1 id="gameOverTitle">YOU DIED</h1>
  <p id="gameOverText">The creature caught you in the dark forest...</p>
  <button id="restartButton">TRY AGAIN</button>
</div>

<!-- Jumpscare Screen -->
<div id="jumpscareScreen">
  <!-- Will be filled dynamically -->
</div>

<!-- No audio elements in HTML - will be created in JavaScript -->

<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
<script>
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000011);
scene.fog = new THREE.Fog(0x000011, 10, 70);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// -------- GAME STATE --------
let gameActive = true;
let jumpscareActive = false;

// -------- SOUNDS --------
// Create audio elements
const jumpscareSound = new Audio('jumpscare.mp3');
const chaseSound = new Audio('chase.mp3');
const ambientSound = new Audio('ambience.mp3');

// Set volume levels
ambientSound.volume = 0.3;
chaseSound.volume = 0;
jumpscareSound.volume = 0.8;

// Set audio to loop
chaseSound.loop = true;
ambientSound.loop = true;

// Error handling for audio
jumpscareSound.onerror = () => console.warn("Failed to load jumpscare.mp3 - make sure it's in the same directory");
ambientSound.onerror = () => console.warn("Failed to load ambient sound");
chaseSound.onerror = () => console.warn("Failed to load chase sound");

// Try to start ambient sound
try {
  ambientSound.play().catch(e => {
    console.log("Audio play blocked, will retry after user interaction:", e);
    // Audio will be started after user interaction
  });
} catch(e) {
  console.log("Audio error:", e);
}

// Start audio after user interaction
document.addEventListener('click', () => {
  if (ambientSound.paused && gameActive) {
    ambientSound.play().catch(e => console.log("Ambient sound play failed:", e));
  }
}, { once: true });

// -------- NIGHT LIGHTING --------
scene.add(new THREE.AmbientLight(0x111111)); // very dim ambient light

// Flashlight as spotlight attached to camera
const flashlight = new THREE.SpotLight(0xffffff, 200, 500, Math.PI/7, 0.5);
flashlight.position.set(0, 0, 0);
flashlight.target.position.set(0, 0, -1);
camera.add(flashlight);
camera.add(flashlight.target);
scene.add(camera);

// -------- GROUND --------
const groundSize = 1200;
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(groundSize, groundSize),
  new THREE.MeshStandardMaterial({ color: 0x112211 })
);
ground.rotation.x = -Math.PI / 2;
scene.add(ground);

// -------- PLAYER --------
const player = new THREE.Object3D();
player.position.set(0, 1.6, 0);
scene.add(player);
player.add(camera);

// -------- CHASING ENTITY --------
let chaser = null;
let chaserSpeed = 7; // Units per second
let chaserDetectionRange = 90000000000000000; // How far it can see/hear the player
let chaserMinSpeed = 7; // Minimum speed when far away
let chaserMaxSpeed = 10; // Maximum speed when close
let chaserCloseDistance = 8; // Distance at which it reaches max speed
let chaserColliderRadius = 1.2;

function createChaser() {
  const chaserGroup = new THREE.Group();
  
  // Main body (red glowing sphere)
  const bodyGeometry = new THREE.SphereGeometry(1, 8, 8);
  const bodyMaterial = new THREE.MeshBasicMaterial({ 
    color: 0xff2222,
    emissive: 0xff0000,
    emissiveIntensity: 0.8
  });
  const body = new THREE.Mesh(bodyGeometry, bodyMaterial);
  chaserGroup.add(body);
  
  // Eyes (white glowing)
  const eyeGeometry = new THREE.SphereGeometry(0.25, 6, 6);
  const eyeMaterial = new THREE.MeshBasicMaterial({ 
    color: 0xffffff,
    emissive: 0xffffff,
    emissiveIntensity: 1.0
  });
  
  const leftEye = new THREE.Mesh(eyeGeometry, eyeMaterial);
  leftEye.position.set(-0.4, 0.3, 0.8);
  chaserGroup.add(leftEye);
  
  const rightEye = new THREE.Mesh(eyeGeometry, eyeMaterial);
  rightEye.position.set(0.4, 0.3, 0.8);
  chaserGroup.add(rightEye);
  
  // Pulsing effect
  chaserGroup.userData.pulseSpeed = 2 + Math.random();
  chaserGroup.userData.pulseTime = Math.random() * Math.PI * 2;
  chaserGroup.userData.originalScale = 1;
  
  // Start far away from player
  const angle = Math.random() * Math.PI * 2;
  const distance = 50 + Math.random() * 30;
  chaserGroup.position.set(
    Math.cos(angle) * distance,
    1.5,
    Math.sin(angle) * distance
  );
  
  scene.add(chaserGroup);
  return chaserGroup;
}

// Initialize chaser
chaser = createChaser();

// -------- TREES & BUSHES --------
const treeColliders = [];
const playerRadius = 0.4;

function createTree(x, z) {
  const tree = new THREE.Group();
  const trunkHeight = 2 + Math.random() * 3.5;
  const trunkRadius = 0.35 + Math.random() * 0.2;

  const trunk = new THREE.Mesh(
    new THREE.CylinderGeometry(trunkRadius, trunkRadius, trunkHeight),
    new THREE.MeshStandardMaterial({ color: 0x4b2e1e })
  );
  trunk.position.y = trunkHeight / 2;
  tree.add(trunk);

  const leaves = new THREE.Mesh(
    new THREE.SphereGeometry(1.2 + Math.random() * 1.2, 7, 7),
    new THREE.MeshStandardMaterial({ color: 0x0f3f1f })
  );
  leaves.position.y = trunkHeight + 1.1;
  tree.add(leaves);

  tree.position.set(x, 0, z);
  scene.add(tree);

  treeColliders.push({x, z, radius: trunkRadius + 0.6});
}

function createBush(x, z) {
  const bush = new THREE.Mesh(
    new THREE.SphereGeometry(0.35 + Math.random() * 0.6, 6, 6),
    new THREE.MeshStandardMaterial({
      color: new THREE.Color().setHSL(0.32, 0.55, 0.22 + Math.random()*0.2)
    })
  );
  bush.position.set(x, 0.3, z);
  scene.add(bush);
}

// ---------------- GENERATE ULTRA-DENSE FOREST (TIGHT NIGHT VERSION) ----------------
const minDistance = 0.9;
let attempts = 0;
for (let i = 0; i < 3000; i++) {
  let placed = false;
  while (!placed && attempts < 25000) {
    attempts++;
    const x = (Math.random() - 0.5) * groundSize;
    const z = (Math.random() - 0.5) * groundSize;
    if (Math.abs(x) < 8 && Math.abs(z) < 8) continue;

    let tooClose = false;
    for (const t of treeColliders) {
      const dx = x - t.x;
      const dz = z - t.z;
      if (Math.sqrt(dx*dx + dz*dz) < minDistance + t.radius) {
        tooClose = true;
        break;
      }
    }

    if (!tooClose) {
      createTree(x, z);
      placed = true;
    }
  }
}

// Generate bushes
for (let i = 0; i < 6000; i++) {
  const x = (Math.random() - 0.5) * groundSize;
  const z = (Math.random() - 0.5) * groundSize;
  createBush(x, z);
}

// -------- INPUT & CAMERA --------
const keys = {};
document.addEventListener("keydown", e => keys[e.code] = true);
document.addEventListener("keyup", e => keys[e.code] = false);

document.body.addEventListener("click", () => {
  if (gameActive) {
    document.body.requestPointerLock();
    // Also try to start audio on first click
    if (ambientSound.paused) {
      ambientSound.play().catch(e => console.log("Ambient sound play failed:", e));
    }
  }
});

let yaw = 0, pitch = 0;
document.addEventListener("mousemove", e => {
  if (document.pointerLockElement === document.body && gameActive) {
    yaw -= e.movementX * 0.002;
    pitch -= e.movementY * 0.002;
    pitch = Math.max(-Math.PI/2, Math.min(Math.PI/2, pitch));
  }
});

// -------- STAMINA --------
let stamina = 100;
const maxStamina = 100;
const drainRate = 25;
const regenRate = 15;
const staminaBar = document.getElementById("staminaBar");

// -------- COLLISION --------
function collides(x, z, radius = playerRadius) {
  for (const t of treeColliders) {
    const dx = x - t.x;
    const dz = z - t.z;
    if (Math.sqrt(dx*dx + dz*dz) < t.radius + radius) return true;
  }
  return false;
}

// -------- JUMPSCARE & GAME OVER --------
function triggerJumpscare() {
  if (!gameActive) return;
  
  gameActive = false;
  jumpscareActive = true;
  
  // Stop all sounds
  ambientSound.pause();
  chaseSound.pause();
  
  // Hide game UI
  document.getElementById('info').style.display = 'none';
  document.getElementById('staminaContainer').style.display = 'none';
  document.getElementById('warning').style.display = 'none';
  
  // Show jumpscare screen
  const jumpscareScreen = document.getElementById('jumpscareScreen');
  
  // Create intense jumpscare effect
  jumpscareScreen.style.background = '#ff0000';
  jumpscareScreen.style.display = 'block';
  jumpscareScreen.innerHTML = `
    <div style="
      width: 100%;
      height: 100%;
      display: flex;
      justify-content: center;
      align-items: center;
      font-family: 'Arial', sans-serif;
      font-size: 120px;
      font-weight: bold;
      color: black;
      text-shadow: 0 0 30px white;
      animation: shake 0.1s infinite;
    ">
      GOT YOU!
    </div>
    <style>
      @keyframes shake {
        0% { transform: translate(1px, 1px) rotate(0deg); }
        10% { transform: translate(-1px, -2px) rotate(-1deg); }
        20% { transform: translate(-3px, 0px) rotate(1deg); }
        30% { transform: translate(3px, 2px) rotate(0deg); }
        40% { transform: translate(1px, -1px) rotate(1deg); }
        50% { transform: translate(-1px, 2px) rotate(-1deg); }
        60% { transform: translate(-3px, 1px) rotate(0deg); }
        70% { transform: translate(3px, 1px) rotate(-1deg); }
        80% { transform: translate(-1px, -1px) rotate(1deg); }
        90% { transform: translate(1px, 2px) rotate(0deg); }
        100% { transform: translate(1px, -2px) rotate(-1deg); }
      }
    </style>
  `;
  
  // Play jumpscare sound
  jumpscareSound.currentTime = 0;
  jumpscareSound.play().catch(e => console.log("Jumpscare sound failed:", e));
  
  // After jumpscare, show game over screen
  setTimeout(() => {
    jumpscareScreen.style.display = 'none';
    document.getElementById('gameOverScreen').style.display = 'flex';
  }, 2000);
}

function restartGame() {
  // Reset game state
  gameActive = true;
  jumpscareActive = false;
  
  // Reset player position
  player.position.set(0, 1.6, 0);
  
  // Reset chaser position
  const angle = Math.random() * Math.PI * 2;
  const distance = 50 + Math.random() * 30;
  chaser.position.set(
    Math.cos(angle) * distance,
    1.5,
    Math.sin(angle) * distance
  );
  
  // Reset stamina
  stamina = 100;
  staminaBar.style.width = '100%';
  
  // Reset camera rotation
  yaw = 0;
  pitch = 0;
  player.rotation.y = 0;
  camera.rotation.x = 0;
  
  // Reset UI
  document.getElementById('info').style.display = 'block';
  document.getElementById('staminaContainer').style.display = 'block';
  document.getElementById('warning').style.display = 'block';
  document.getElementById('gameOverScreen').style.display = 'none';
  document.body.style.backgroundColor = '';
  
  // Restart ambient sound
  ambientSound.currentTime = 0;
  chaseSound.currentTime = 0;
  chaseSound.volume = 0;
  ambientSound.play().catch(e => console.log("Audio play failed:", e));
  
  // Request pointer lock if not already locked
  if (document.pointerLockElement === document.body) {
    document.exitPointerLock();
  }
}

// Restart button
document.getElementById('restartButton').addEventListener('click', restartGame);

// -------- CHASER AI --------
function updateChaser(delta) {
  if (!chaser || !gameActive || jumpscareActive) return;
  
  const warningElement = document.getElementById('warning');
  
  // Calculate distance to player
  const dx = player.position.x - chaser.position.x;
  const dz = player.position.z - chaser.position.z;
  const distanceToPlayer = Math.sqrt(dx*dx + dz*dz);
  
  // Update chase sound volume based on distance
  if (distanceToPlayer < chaserDetectionRange) {
    const chaseVolume = Math.max(0, 0.6 * (1 - distanceToPlayer / chaserDetectionRange));
    chaseSound.volume = chaseVolume;
    
    // Play chase sound if not already playing
    if (chaseSound.paused) {
      chaseSound.play().catch(e => console.log("Chase sound failed:", e));
    }
  } else {
    chaseSound.volume = 0;
  }
  
  // Update warning display based on distance
  if (distanceToPlayer < 15) {
    warningElement.style.opacity = Math.max(0.3, 1 - (distanceToPlayer / 15));
    if (distanceToPlayer < 8) {
      warningElement.textContent = "!!! EXTREME DANGER !!!";
    } else {
      warningElement.textContent = "!! DANGER CLOSE !!";
    }
  } else {
    warningElement.style.opacity = 0;
  }
  
  // Check if player is within detection range
  if (distanceToPlayer > chaserDetectionRange) {
    // Wander randomly if player is too far
    chaser.userData.wanderTime = (chaser.userData.wanderTime || 0) + delta;
    if (chaser.userData.wanderTime > 2 || !chaser.userData.wanderDirection) {
      chaser.userData.wanderTime = 0;
      const angle = Math.random() * Math.PI * 2;
      chaser.userData.wanderDirection = new THREE.Vector3(
        Math.cos(angle),
        0,
        Math.sin(angle)
      );
    }
    
    // Move in wander direction
    const wanderSpeed = 1.5;
    const newX = chaser.position.x + chaser.userData.wanderDirection.x * wanderSpeed * delta;
    const newZ = chaser.position.z + chaser.userData.wanderDirection.z * wanderSpeed * delta;
    
    // Simple collision avoidance for wandering
    if (!collides(newX, newZ, chaserColliderRadius) && 
        Math.abs(newX) < groundSize/2 - 10 && 
        Math.abs(newZ) < groundSize/2 - 10) {
      chaser.position.x = newX;
      chaser.position.z = newZ;
    }
    
    // Look in movement direction
    if (chaser.userData.wanderDirection.length() > 0.1) {
      chaser.lookAt(
        chaser.position.x + chaser.userData.wanderDirection.x,
        chaser.position.y,
        chaser.position.z + chaser.userData.wanderDirection.z
      );
    }
    
    return;
  }
  
  // CHASE THE PLAYER!
  // Calculate direction to player (normalized)
  const direction = new THREE.Vector3(dx, 0, dz).normalize();
  
  // Adjust speed based on distance (faster when closer)
  let speedMultiplier = 1.0;
  if (distanceToPlayer < chaserCloseDistance) {
    speedMultiplier = chaserMaxSpeed / chaserSpeed;
  } else {
    // Linear interpolation between min and max speed
    const t = 1 - Math.min(1, distanceToPlayer / chaserDetectionRange);
    speedMultiplier = (chaserMinSpeed + t * (chaserMaxSpeed - chaserMinSpeed)) / chaserSpeed;
  }
  
  const adjustedSpeed = chaserSpeed * speedMultiplier;
  
  // Calculate movement
  const moveX = direction.x * adjustedSpeed * delta;
  const moveZ = direction.z * adjustedSpeed * delta;
  
  // Check collision before moving
  const newX = chaser.position.x + moveX;
  const newZ = chaser.position.z + moveZ;
  
  // Simple obstacle avoidance - try to move around trees
  if (!collides(newX, newZ, chaserColliderRadius)) {
    chaser.position.x = newX;
    chaser.position.z = newZ;
  } else {
    // Try perpendicular movement
    const perpX = -direction.z * adjustedSpeed * delta;
    const perpZ = direction.x * adjustedSpeed * delta;
    
    if (!collides(chaser.position.x + perpX, chaser.position.z + perpZ, chaserColliderRadius)) {
      chaser.position.x += perpX;
      chaser.position.z += perpZ;
    }
  }
  
  // Make chaser look at player (but only rotate on Y axis)
  chaser.lookAt(player.position.x, chaser.position.y, player.position.z);
  
  // Pulsing effect
  chaser.userData.pulseTime += delta * chaser.userData.pulseSpeed;
  const pulseScale = 1 + 0.1 * Math.sin(chaser.userData.pulseTime);
  chaser.scale.setScalar(pulseScale);
  
  // Check if caught player (trigger jumpscare)
  if (distanceToPlayer < 2.5) {
    triggerJumpscare();
  }
}

// -------- LOOP --------
let lastTime = performance.now();
function animate(time) {
  requestAnimationFrame(animate);
  const delta = (time - lastTime)/1000;
  lastTime = time;

  // Only update game if active
  if (gameActive && !jumpscareActive) {
    let speed = 0.1;
    
    // Check if Shift is pressed AND stamina is greater than 0
    if (keys["ShiftLeft"] && stamina > 0) {
      speed = 0.25;
      stamina -= drainRate * delta;
    } else {
      stamina += regenRate * delta;
    }

    stamina = Math.max(0, Math.min(maxStamina, stamina));
    
    // If stamina reaches 0, force speed to be 0.1
    if (stamina <= 0) {
      speed = 0.1;
    }
    
    staminaBar.style.width = (stamina/maxStamina*100) + "%";

    const forward = new THREE.Vector3(Math.sin(yaw), 0, Math.cos(yaw));
    const right = new THREE.Vector3(Math.sin(yaw+Math.PI/2), 0, Math.cos(yaw+Math.PI/2));

    let moveX = 0, moveZ = 0;
    if (keys["KeyW"]) { moveX -= forward.x*speed; moveZ -= forward.z*speed; }
    if (keys["KeyS"]) { moveX += forward.x*speed; moveZ += forward.z*speed; }
    if (keys["KeyA"]) { moveX -= right.x*speed; moveZ -= right.z*speed; }
    if (keys["KeyD"]) { moveX += right.x*speed; moveZ += right.z*speed; }

    if (!collides(player.position.x + moveX, player.position.z))
      player.position.x += moveX;
    if (!collides(player.position.x, player.position.z + moveZ))
      player.position.z += moveZ;

    player.rotation.y = yaw;
    camera.rotation.x = pitch;
    
    // Update chaser AI
    updateChaser(delta);
  }

  renderer.render(scene, camera);
}

animate(performance.now());

// Handle window resize
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
