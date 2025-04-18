<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Florr.io RPG Clone</title>
  <style>
    body {
      margin: 0;
      background: #fff;
      font-family: sans-serif;
    }
    #loadingScreen {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      background: #fff;
      color: black;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      z-index: 1000;
    }
    canvas {
      display: block;
      position: absolute;
      top: 0;
      left: 0;
      background-color: #fff;
      z-index: 0;
    }
  </style>
</head>
<body>
  <div id="loadingScreen">
    <h1>Florr.io RPG</h1>
    <p>Enter your nickname:</p>
    <input id="nicknameInput" type="text" maxlength="12" placeholder="Your name" style="padding:5px;font-size:16px;" />
    <button onclick="startGame()" style="margin-top:20px;font-size:20px;padding:10px 30px;">Start Game</button>
  </div>

  <canvas id="gameCanvas"></canvas>

  <div id="shop" style="position:fixed;bottom:10px;left:10px;background:rgba(0,0,0,0.7);color:#fff;padding:10px;display:none;z-index:10;">
    <h3>Shop</h3>
    <p>Select number of petals to buy:</p>
    <input id="petalQuantity" type="number" min="1" placeholder="Enter quantity" style="padding:5px;font-size:16px;width:70px;" />
    <button onclick="buyPetals()">Buy Petals</button>

    <h3>Weapons</h3>
    <div id="weaponList"></div> <!-- Weapon list will appear here -->

    <div id="weaponSelectionMessage" style="margin-top:5px;"></div>
  </div>

  <button onclick="toggleShop()" style="position:fixed;bottom:10px;left:10px;z-index:11;">🛒 Shop</button>

  <!-- Game Over Screen -->
  <div id="gameOverScreen" style="position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.7);color:white;flex-direction:column;align-items:center;justify-content:center;display:none;z-index:1000;">
    <h1>Game Over</h1>
    <button onclick="restartGame()" style="font-size:20px;padding:10px 30px;">Continue</button>
  </div>

  <script>
    let nickname = "Player";
    let gameStarted = false;
    let player = null;
    let keys = {};
    let petals = [];
    let enemies = [];
    let coins = 0;
    let xp = 0;
    let xpNeeded = 100;
    let level = 1;
    
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    // Weapon types with damage values based on their type
    const weaponTypes = {
      common: { damage: 1, color: '#f0f', price: 5000 },
      unusual: { damage: 5, color: '#f0e68c', price: 10000 }, // More saturated yellow
      rare: { damage: 100, color: '#00f', price: 50000 },
      epic: { damage: 500, color: '#800080', price: 100000 },
      legendary: { damage: 1000, color: '#ff0000', price: 300000 },
      mythic: { damage: 2000, color: '#8a2be2', price: 500000 },
      ultra: { damage: 5000, color: '#ff1493', price: 1000000 },
      super: { damage: 10000, color: '#00ff00', price: 5000000 },
      unique: { damage: 1000000, color: '#808080', price: 10000000 }
    };

    // Monster Types and spawn intervals
    const monsterTypes = {
      common: { spawnTime: 1000, xp: 100, gold: 100, color: 'lightgreen' },
      unusual: { spawnTime: 3000, xp: 500, gold: 500, color: '#f0e68c' }, // More saturated yellow
      rare: { spawnTime: 5000, xp: 1000, gold: 1000, color: 'blue' },
      epic: { spawnTime: 7000, xp: 5000, gold: 5000, color: 'purple' },
      legendary: { spawnTime: 10000, xp: 10000, gold: 10000, color: 'red' },
      mythic: { spawnTime: 30000, xp: 30000, gold: 30000, color: 'skyblue' },
      ultra: { spawnTime: 60000, xp: 100000, gold: 100000, color: 'pink' },
      super: { spawnTime: 120000, xp: 500000, gold: 500000, color: 'mint' },
      unique: { spawnTime: 180000, xp: 1000000, gold: 1000000, color: 'gray' }
    };

    function startGame() {
      const input = document.getElementById("nicknameInput");
      if (input.value.trim()) nickname = input.value.trim();
      document.getElementById("loadingScreen").style.display = "none";
      gameStarted = true;
      initGame();
    }

    function initGame() {
      player = {
        x: canvas.width / 2, // center x
        y: canvas.height / 2, // center y
        radius: 20,
        color: "#000",
        speed: 3,
        hp: 100,
        maxHp: 100
      };
      petals = [];
      for (let i = 0; i < 3; i++) {
        petals.push({ angle: i * (Math.PI * 2 / 3), length: 40 });
      }
      enemies = [];
      spawnEnemy();
      animate();
      showRandomWeapons();
    }

    function spawnEnemy() {
      Object.keys(monsterTypes).forEach(type => {
        const monster = monsterTypes[type];
        setInterval(() => {
          enemies.push({
            x: Math.random() * canvas.width,
            y: Math.random() * canvas.height,
            radius: 15,
            color: monster.color,
            hp: 100,
            type,
            xp: monster.xp,
            gold: monster.gold
          });
        }, monster.spawnTime);
      });
    }

    function animate() {
      if (!gameStarted) return;
      if (keys['w'] || keys['ArrowUp']) player.y -= player.speed;
      if (keys['s'] || keys['ArrowDown']) player.y += player.speed;
      if (keys['a'] || keys['ArrowLeft']) player.x -= player.speed;
      if (keys['d'] || keys['ArrowRight']) player.x += player.speed;

      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "#fff";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // draw XP bar
      ctx.fillStyle = '#000';
      ctx.fillRect(20, 20, 200, 10);
      ctx.fillStyle = '#00f';
      ctx.fillRect(20, 20, 200 * (xp / xpNeeded), 10);
      ctx.fillStyle = '#000';
      ctx.font = '12px sans-serif';
      ctx.fillText(`XP: ${xp} / ${xpNeeded} (Lv.${level})`, 20, 17);
      ctx.fillText(`💰 Coins: ${coins}`, 20, 40);

      // draw player
      ctx.beginPath();
      ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
      ctx.fillStyle = player.color;
      ctx.fill();
      ctx.closePath();

      // draw player HP bar
      ctx.fillStyle = '#000';
      ctx.fillRect(player.x - 25, player.y - 40, 50, 5);
      ctx.fillStyle = '#0f0';
      ctx.fillRect(player.x - 25, player.y - 40, 50 * (player.hp / player.maxHp), 5);

      // draw petals
      petals.forEach(petal => {
        const angle = Date.now() / 300 + petal.angle;
        const px = player.x + Math.cos(angle) * petal.length;
        const py = player.y + Math.sin(angle) * petal.length;
        ctx.beginPath();
        ctx.arc(px, py, 8, 0, Math.PI * 2);
        ctx.fillStyle = petal.color;
        ctx.fill();
        ctx.closePath();
      });

      // draw enemies and HP bars
      enemies.forEach((enemy, index) => {
        const dx = player.x - enemy.x;
        const dy = player.y - enemy.y;
        const dist = Math.hypot(dx, dy);
        
        // If the weapon hits the enemy, apply damage
        petals.forEach(petal => {
          if (dist < 20) {
            enemy.hp -= petal.damage || 1;
            if (enemy.hp <= 0) {
              coins += enemy.gold;
              xp += enemy.xp;
              enemies.splice(index, 1); // Remove the enemy after collision
            }
          }
        });

        ctx.beginPath();
        ctx.arc(enemy.x, enemy.y, enemy.radius, 0, Math.PI * 2);
        ctx.fillStyle = enemy.color;
        ctx.fill();
        ctx.closePath();
      });

      requestAnimationFrame(animate);
    }

    window.addEventListener('keydown', e => keys[e.key] = true);
    window.addEventListener('keyup', e => keys[e.key] = false);

    function toggleShop() {
      const shop = document.getElementById('shop');
      shop.style.display = shop.style.display === 'block' ? 'none' : 'block';
    }

    function buyWeapon(type, weapon) {
      const message = document.getElementById('weaponSelectionMessage');
      const selectedWeapon = weaponTypes[type];
      if (selectedWeapon && coins >= selectedWeapon.price) {
        coins -= selectedWeapon.price;
        petals.push({ ...selectedWeapon, angle: Math.random() * Math.PI * 2, lastUsed: Date.now() });
        message.textContent = `Purchased: ${weapon}`;
      } else {
        message.textContent = selectedWeapon ? 'Not enough coins!' : 'Invalid weapon selection!';
      }
    }

    function buyPetals() {
      const quantity = document.getElementById('petalQuantity').value;
      if (coins >= quantity * 5000) { // Assuming each petal costs 5000 gold
        coins -= quantity * 5000;
        for (let i = 0; i < quantity; i++) {
          petals.push({ ...weaponTypes.common, angle: Math.random() * Math.PI * 2 });
        }
        alert(`Purchased ${quantity} petals!`);
      } else {
        alert('Not enough coins!');
      }
    }

    function showRandomWeapons() {
      const weaponList = document.getElementById("weaponList");
      weaponList.innerHTML = ""; // Clear existing weapon list

      for (let i = 0; i < 6; i++) {
        const randomType = Object.keys(weaponTypes)[Math.floor(Math.random() * Object.keys(weaponTypes).length)];
        const randomWeapon = weaponTypes[randomType];
        
        const button = document.createElement("button");
        button.innerText = `Buy ${randomType} Weapon (${randomWeapon.price} gold)`;
        button.onclick = () => buyWeapon(randomType, randomType);
        
        weaponList.appendChild(button);
      }
    }

    function restartGame() {
      document.getElementById('gameOverScreen').style.display = 'none';
      gameStarted = false;
      initGame();
    }
  </script>
</body>
</html>
