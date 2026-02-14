<template>
  <div class="game-container">
    <canvas ref="gameCanvas" class="game-canvas"></canvas>
    
    <!-- 顶部状态栏 -->
    <div class="top-bar">
      <div class="hero-info">
        <span class="hero-name">{{ hero.name }}</span>
        <span class="hero-level">Lv.{{ hero.level }}</span>
        <div class="hp-bar">
          <div class="hp-fill" :style="{ width: (hero.hp / hero.maxHp * 100) + '%' }"></div>
        </div>
      </div>
      <div class="resources">
        <span>💰{{ resources.gold }}</span>
      </div>
    </div>
    
    <!-- 区域信息 -->
    <div class="area-info">
      <span>{{ currentArea?.name }} - 区域 {{ currentRegion + 1 }}/{{ regionCount }}</span>
      <span class="enemies-count">敌人: {{ enemiesRemaining }}</span>
    </div>
    
    <!-- 底部操作栏 -->
    <div class="bottom-bar">
      <button @click="showArmy = !showArmy">⚔️部队</button>
      <button @click="showMap = !showMap">🗺️地图</button>
    </div>
    
    <!-- 部队面板 -->
    <div class="panel" v-if="showArmy">
      <div class="panel-header">
        <span>⚔️ 部队</span>
        <button @click="showArmy = false">×</button>
      </div>
      <div class="army-list">
        <div class="army-unit" v-for="(unit, index) in army" :key="index">
          <span>{{ unit.icon }}</span>
          <span>{{ unit.name }}</span>
          <span>x{{ unit.count }}</span>
        </div>
      </div>
    </div>
    
    <!-- 地图面板 -->
    <div class="panel" v-if="showMap">
      <div class="panel-header">
        <span>🗺️ 世界地图</span>
        <button @click="showMap = false">×</button>
      </div>
      <div class="world-map">
        <div 
          class="map-area" 
          v-for="area in worldMap" 
          :key="area.id"
          :class="{ unlocked: area.unlocked, current: currentArea?.id === area.id }"
          @click="travelTo(area)"
        >
          <span class="area-icon">{{ area.icon }}</span>
          <span class="area-name">{{ area.name }}</span>
        </div>
      </div>
    </div>
    
    <!-- 战斗界面 -->
    <div class="battle-overlay" v-if="inBattle">
      <div class="battle-title">⚔️ 战斗中</div>
      <div class="battle-units">
        <div class="our-side">
          <div v-for="(unit, i) in army" :key="'a'+i" class="battle-unit">
            {{ unit.icon }} x{{ unit.count }}
          </div>
        </div>
        <div class="vs">VS</div>
        <div class="enemy-side">
          <div v-for="(e, i) in battleEnemies" :key="'e'+i" class="battle-unit enemy">
            {{ e.icon }} x{{ e.count }}
          </div>
        </div>
      </div>
    </div>
    
    <!-- 消息日志 -->
    <div class="message-log">
      <div class="message" v-for="(msg, i) in messages" :key="i">{{ msg }}</div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Game',
  data() {
    return {
      canvas: null,
      ctx: null,
      tileSize: 40,
      viewWidth: 15,// 可视区域宽度（格子数）
      viewHeight: 12,// 可视区域高度
      mapWidth: 30,
      mapHeight: 24,
      
      showArmy: false,
      showMap: false,
      inBattle: false,
      messages: [],
      
      // 英雄
      hero: {
        name: '勇者',
        level: 1,
        hp: 100,
        maxHp: 100,
        attack: 10,
        defense: 5,
        x: 5,
        y: 5,
        exp: 0,
        expToLevel: 100
      },
      
      // 资源
      resources: { gold: 100 },
      
      // 部队
      army: [
        { id: 1, name: '步兵', icon: '🗡️', count: 10, attack: 5, defense: 3 }
      ],
      
      // 当前地图
      currentMap: [],
      mapObjects: [],
      
      // 区域系统
      currentArea: null,
      currentRegion: 0,
      regionCount: 4, // 每个区域有4个分区
      regionGates: [], // 关口位置
      
      // 世界地图
      worldMap: [
        { id: 1, name: '新手平原', icon: '🌿', level: 1, unlocked: true },
        { id: 2, name: '黑暗森林', icon: '🌲', level: 3, unlocked: false },
        { id: 3, name: '矿石山脉', icon: '⛰️', level: 5, unlocked: false },
        { id: 4, name: '魔王城', icon: '🏰', level: 10, unlocked: false },
      ],
      
      // 战斗
      battleEnemies: [],
      
      // 动画
      animationFrame: null,
      lastTime: 0,
      keys: {},
      moveDelay: 0,
    }
  },
  computed: {
    enemiesRemaining() {
      return this.mapObjects.filter(o => o.type === 'enemy').length;
    }
  },
  mounted() {
    this.initGame();
  },
  beforeUnmount() {
    if (this.animationFrame) cancelAnimationFrame(this.animationFrame);
    window.removeEventListener('keydown', this.handleKeyDown);
    window.removeEventListener('keyup', this.handleKeyUp);
  },
  methods: {
    initGame() {
      this.canvas = this.$refs.gameCanvas;
      this.ctx = this.canvas.getContext('2d');
      
      this.resizeCanvas();
      window.addEventListener('resize', this.resizeCanvas);
      window.addEventListener('keydown', this.handleKeyDown);
      window.addEventListener('keyup', this.handleKeyUp);
      this.canvas.addEventListener('touchstart', this.handleTouch);
      this.canvas.addEventListener('click', this.handleClick);
      
      this.currentArea = this.worldMap[0];
      this.currentRegion = 0;
      this.generateMap();
      
      this.gameLoop();
      this.addMessage('欢迎来到英雄无敌世界！');
    },
    
    resizeCanvas() {
      this.canvas.width = this.canvas.parentElement.clientWidth;
      this.canvas.height = this.canvas.parentElement.clientHeight;
    },
    
    // 生成地图（带分区）
    generateMap() {
      this.currentMap = [];
      this.mapObjects = [];
      this.regionGates = [];
      
      // 生成基础地图
      for (let y = 0; y < this.mapHeight; y++) {
        const row = [];
        for (let x = 0; x < this.mapWidth; x++) {
          let tile = 0;
          
          // 边界
          if (x === 0 || x === this.mapWidth - 1 || y === 0 || y === this.mapHeight - 1) {
            tile = 1;
          }
          // 分区边界（每6行一个分区）
          else if (y > 0 && y % 6 === 0 && !this.isGatePosition(x)) {
            tile = 1;
          }
          // 随机障碍
          else if (Math.random() < 0.08) {
            tile = Math.random() < 0.7 ? 1 : 2;
          }
          
          row.push(tile);
        }
        this.currentMap.push(row);
      }
      
      // 创建关口
      this.createGates();
      
      // 生成物体
      this.generateObjects();
      
      // 生成敌人
      this.generateEnemies();
      
      // 设置英雄位置
      this.hero.x = 3;
      this.hero.y = 3;
    },
    
    isGatePosition(x) {
      // 关口位置在地图中间区域
      const gates = [10, 11, 12, 13, 14, 15, 16, 17, 18, 19];
      return gates.includes(x);
    },
    
    createGates() {
      // 在每个分区边界创建关口
      const gateY = [6, 12, 18];
      gateY.forEach((y, index) => {
        const gateX = 14 + Math.floor(Math.random() * 3);
        this.currentMap[y][gateX] = 3; // 关口标记
        this.regionGates.push({ x: gateX, y: y, targetRegion: index + 1 });
        
        // 清理关口周围的墙
        for (let dx = -1; dx <= 1; dx++) {
          if (gateX + dx > 0 && gateX + dx < this.mapWidth - 1) {
            this.currentMap[y][gateX + dx] = 0;
          }
        }
      });
    },
    
    generateObjects() {
      const types = [
        { type: 'gold', icon: '💰', effect: { gold: 30 } },
        { type: 'potion', icon: '🧪', effect: { heal: 30 } },
        { type: 'recruit', icon: '⚔️', effect: { recruit: true } },
      ];
      
      for (let i = 0; i < 10; i++) {
        const pos = this.findEmptyTile();
        if (pos) {
          const t = types[Math.floor(Math.random() * types.length)];
          this.mapObjects.push({ ...t, x: pos.x, y: pos.y });
        }
      }
    },
    
    generateEnemies() {
      const level = (this.currentArea?.level || 1) + this.currentRegion;
      const enemies = [
        { name: '哥布林', icon: '👺', hp: 20, attack: 4, exp: 10, gold: 15 },
        { name: '骷髅', icon: '💀', hp: 25, attack: 6, exp: 15, gold: 20 },
        { name: '兽人', icon: '👹', hp: 40, attack: 10, exp: 25, gold: 40 },
      ];
      
      const count = 5 + level * 2;
      for (let i = 0; i < count; i++) {
        const pos = this.findEmptyTile(true);
        if (pos) {
          const e = enemies[Math.min(Math.floor(level / 2), enemies.length - 1)];
          this.mapObjects.push({
            type: 'enemy',
            ...e,
            x: pos.x,
            y: pos.y
          });
        }
      }
    },
    
    findEmptyTile(awayFromHero = false) {
      for (let i = 0; i < 100; i++) {
        const x = Math.floor(Math.random() * (this.mapWidth - 2)) + 1;
        const y = Math.floor(Math.random() * (this.mapHeight - 2)) + 1;
        
        if (this.currentMap[y][x] !== 0) continue;
        if (this.mapObjects.some(o => o.x === x && o.y === y)) continue;
        if (awayFromHero && Math.abs(x - this.hero.x) < 5 && Math.abs(y - this.hero.y) < 5) continue;
        
        return { x, y };
      }
      return null;
    },
    
    gameLoop(timestamp = 0) {
      const dt = timestamp - this.lastTime;
      this.lastTime = timestamp;
      
      this.update(dt);
      this.render();
      
      this.animationFrame = requestAnimationFrame(this.gameLoop);
    },
    
    update(dt) {
      this.moveDelay -= dt;
      if (this.moveDelay > 0) return;
      
      if (this.keys['ArrowUp'] || this.keys['w']) { this.moveHero(0, -1); this.moveDelay = 150; }
      else if (this.keys['ArrowDown'] || this.keys['s']) { this.moveHero(0, 1); this.moveDelay = 150; }
      else if (this.keys['ArrowLeft'] || this.keys['a']) { this.moveHero(-1, 0); this.moveDelay = 150; }
      else if (this.keys['ArrowRight'] || this.keys['d']) { this.moveHero(1, 0); this.moveDelay = 150; }
    },
    
    moveHero(dx, dy) {
      if (this.inBattle) return;
      
      const nx = this.hero.x + dx;
      const ny = this.hero.y + dy;
      
      if (nx < 0 || nx >= this.mapWidth || ny < 0 || ny >= this.mapHeight) return;
      
      const tile = this.currentMap[ny][nx];
      if (tile === 1) return; // 墙
      if (tile === 2) return; // 水
      
      // 检查关口
      if (tile === 3) {
        this.passGate(nx, ny);
        return;
      }
      
      this.hero.x = nx;
      this.hero.y = ny;
      
      this.checkCollision();
    },
    
    passGate(x, y) {
      const gate = this.regionGates.find(g => g.x === x && g.y === y);
      if (!gate) return;
      
      // 检查当前区域是否清空
      if (this.enemiesRemaining > 0) {
        this.addMessage('还有敌人，无法通过！');
        return;
      }
      
      // 只能进入下一个区域
      if (gate.targetRegion !== this.currentRegion + 1) {
        this.addMessage('请先通过前面的区域！');
        return;
      }
      
      if (gate.targetRegion < this.regionCount) {
        this.currentRegion = gate.targetRegion;
        this.hero.y = y + 1;
        this.addMessage(`进入区域 ${this.currentRegion + 1}`);
        
        // 检查是否完成整个区域
        if (this.currentRegion >= this.regionCount - 1 && this.enemiesRemaining === 0) {
          this.completeArea();
        }
      }
    },
    
    completeArea() {
      this.addMessage(`${this.currentArea.name} 已清除！`);
      
      const idx = this.worldMap.findIndex(a => a.id === this.currentArea.id);
      if (idx < this.worldMap.length - 1) {
        this.worldMap[idx + 1].unlocked = true;
        this.addMessage(`新区域解锁：${this.worldMap[idx + 1].name}`);
      }
    },
    
    checkCollision() {
      const obj = this.mapObjects.find(o => o.x === this.hero.x && o.y === this.hero.y);
      if (!obj) return;
      
      if (obj.type === 'enemy') {
        this.startBattle(obj);
      } else {
        this.collectObject(obj);
      }
    },
    
    collectObject(obj) {
      if (obj.effect.gold) {
        this.resources.gold += obj.effect.gold;
        this.addMessage(`+${obj.effect.gold} 金币`);
      }
      if (obj.effect.heal) {
        this.hero.hp = Math.min(this.hero.maxHp, this.hero.hp + obj.effect.heal);
        this.addMessage(`恢复 ${obj.effect.heal} HP`);
      }
      if (obj.effect.recruit) {
        this.recruitUnit();
      }
      
      this.mapObjects = this.mapObjects.filter(o => o !== obj);
    },
    
    recruitUnit() {
      if (this.resources.gold >= 50) {
        const unit = { id: Date.now(), name: '步兵', icon: '🗡️', count: 5, attack: 5, defense: 3 };
        const existing = this.army.find(u => u.name === unit.name);
        if (existing) {
          existing.count += 5;
        } else if (this.army.length < 7) {
          this.army.push(unit);
        }
        this.resources.gold -= 50;
        this.addMessage('招募了步兵 x5');
      }
    },
    
    startBattle(enemy) {
      this.inBattle = true;
      this.battleEnemies = [{ ...enemy, count: 3 + Math.floor(Math.random() * 3) }];
      
      setTimeout(() => this.resolveBattle(enemy), 1500);
    },
    
    resolveBattle(enemy) {
      const ourPower = this.army.reduce((s, u) => s + u.attack * u.count, 0) + this.hero.attack;
      const theirPower = this.battleEnemies.reduce((s, e) => s + e.attack * e.count, 0);
      
      if (ourPower > theirPower) {
        this.addMessage(`胜利！+${enemy.exp} 经验`);
        this.hero.exp += enemy.exp;
        this.resources.gold += enemy.gold || 20;
        
        if (this.hero.exp >= this.hero.expToLevel) {
          this.levelUp();
        }
        
        this.mapObjects = this.mapObjects.filter(o => o !== enemy);
      } else {
        this.addMessage('战败...失去金币');
        this.resources.gold = Math.max(0, this.resources.gold - 30);
      }
      
      this.inBattle = false;
      this.battleEnemies = [];
    },
    
    levelUp() {
      this.hero.level++;
      this.hero.exp -= this.hero.expToLevel;
      this.hero.expToLevel = Math.floor(this.hero.expToLevel * 1.5);
      this.hero.maxHp += 10;
      this.hero.hp = this.hero.maxHp;
      this.hero.attack += 2;
      this.addMessage(`升级！Lv.${this.hero.level}`);
    },
    
    travelTo(area) {
      if (!area.unlocked) return;
      if (area.id === this.currentArea.id) return;
      
      this.currentArea = area;
      this.currentRegion = 0;
      this.generateMap();
      this.showMap = false;
      this.addMessage(`前往 ${area.name}`);
    },
    
    addMessage(msg) {
      this.messages.push(msg);
      if (this.messages.length > 5) this.messages.shift();
    },
    
    // 渲染
    render() {
      const ctx = this.ctx;
      const ts = this.tileSize;
      
      ctx.fillStyle = '#1a1a2e';
      ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
      
      // 相机跟随英雄
      const camX = this.hero.x - Math.floor(this.viewWidth / 2);
      const camY = this.hero.y - Math.floor(this.viewHeight / 2);
      
      const offsetX = (this.canvas.width - this.viewWidth * ts) / 2;
      const offsetY = (this.canvas.height - this.viewHeight * ts) / 2;
      
      ctx.save();
      ctx.translate(offsetX - camX * ts, offsetY - camY * ts);
      
      // 绘制地图
      for (let y = 0; y < this.mapHeight; y++) {
        for (let x = 0; x < this.mapWidth; x++) {
          const tile = this.currentMap[y][x];
          switch (tile) {
            case 0: ctx.fillStyle = '#4a7c59'; break;
            case 1: ctx.fillStyle = '#4a4a4a'; break;
            case 2: ctx.fillStyle = '#4a90a4'; break;
            case 3: ctx.fillStyle = '#ffd700'; break; // 关口
          }
          ctx.fillRect(x * ts, y * ts, ts - 1, ts - 1);
        }
      }
      
      // 绘制物体
      ctx.font = `${ts - 8}px Arial`;
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      
      for (const obj of this.mapObjects) {
        ctx.fillText(obj.icon, obj.x * ts + ts / 2, obj.y * ts + ts / 2);
      }
      
      // 绘制关口（金色门）- 只显示下一个可进入的关口
      ctx.font = `${ts - 4}px Arial`;
      const nextGate = this.regionGates.find(g => g.targetRegion === this.currentRegion + 1);
      if (nextGate) {
        const canPass = this.enemiesRemaining === 0;
        ctx.fillText(canPass ? '🚪' : '🔒', nextGate.x * ts + ts / 2, nextGate.y * ts + ts / 2);
      }
      
      // 绘制英雄（大一点）
      ctx.font = `${ts - 2}px Arial`;
      ctx.fillText('🧙', this.hero.x * ts + ts / 2, this.hero.y * ts + ts / 2);
      
      ctx.restore();
    },
    
    handleKeyDown(e) { this.keys[e.key] = true; },
    handleKeyUp(e) { this.keys[e.key] = false; },
    
    handleTouch(e) {
      e.preventDefault();
      const touch = e.touches[0];
      const rect = this.canvas.getBoundingClientRect();
      
      const camX = this.hero.x - Math.floor(this.viewWidth / 2);
      const camY = this.hero.y - Math.floor(this.viewHeight / 2);
      const offsetX = (this.canvas.width - this.viewWidth * this.tileSize) / 2;
      const offsetY = (this.canvas.height - this.viewHeight * this.tileSize) / 2;
      
      const tileX = Math.floor((touch.clientX - rect.left - offsetX) / this.tileSize) + camX;
      const tileY = Math.floor((touch.clientY - rect.top - offsetY) / this.tileSize) + camY;
      
      const dx = tileX - this.hero.x;
      const dy = tileY - this.hero.y;
      
      if (Math.abs(dy) >= Math.abs(dx)) {
        if (dy !== 0) this.moveHero(0, Math.sign(dy));
      } else {
        if (dx !== 0) this.moveHero(Math.sign(dx), 0);
      }
    },
    
    handleClick(e) {
      const rect = this.canvas.getBoundingClientRect();
      
      const camX = this.hero.x - Math.floor(this.viewWidth / 2);
      const camY = this.hero.y - Math.floor(this.viewHeight / 2);
      const offsetX = (this.canvas.width - this.viewWidth * this.tileSize) / 2;
      const offsetY = (this.canvas.height - this.viewHeight * this.tileSize) / 2;
      
      const tileX = Math.floor((e.clientX - rect.left - offsetX) / this.tileSize) + camX;
      const tileY = Math.floor((e.clientY - rect.top - offsetY) / this.tileSize) + camY;
      
      const dx = tileX - this.hero.x;
      const dy = tileY - this.hero.y;
      
      if (Math.abs(dy) >= Math.abs(dx)) {
        if (dy !== 0) this.moveHero(0, Math.sign(dy));
      } else {
        if (dx !== 0) this.moveHero(Math.sign(dx), 0);
      }
    }
  }
}
</script>

<style>
.game-container {
  width: 100vw;
  height: 100vh;
  background: #1a1a2e;
  position: relative;
  overflow: hidden;
}

.game-canvas {
  width: 100%;
  height: 100%;
}

.top-bar {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  padding: 8px 15px;
  background: rgba(0,0,0,0.8);
  display: flex;
  justify-content: space-between;
  color: white;
  font-size: 14px;
}

.hero-info { display: flex; align-items: center; gap: 10px; }
.hero-name { color: #ffd700; font-weight: bold; }
.hero-level { color: #7fff7f; }
.hp-bar {
  width: 80px;
  height: 10px;
  background: #333;
  border-radius: 5px;
  overflow: hidden;
}
.hp-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff4444, #ff6666);
}

.resources { color: #ffd700; }

.area-info {
  position: absolute;
  top: 40px;
  left: 10px;
  background: rgba(0,0,0,0.8);
  padding: 5px 10px;
  border-radius: 5px;
  color: white;
  font-size: 12px;
}

.enemies-count { margin-left: 10px; color: #ff7f7f; }

.bottom-bar {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 10px;
  background: rgba(0,0,0,0.8);
  display: flex;
  justify-content: center;
  gap: 10px;
}

.bottom-bar button {
  padding: 10px 20px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 14px;
}

.panel {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(20, 20, 40, 0.95);
  border: 2px solid #667eea;
  border-radius: 10px;
  padding: 20px;
  min-width: 280px;
  color: white;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #667eea;
}

.panel-header button {
  background: none;
  border: none;
  color: white;
  font-size: 20px;
}

.army-list { display: flex; flex-direction: column; gap: 8px; }
.army-unit {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px;
  background: rgba(255,255,255,0.1);
  border-radius: 5px;
}

.world-map {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
}

.map-area {
  padding: 15px;
  background: rgba(100,100,100,0.3);
  border-radius: 10px;
  text-align: center;
  cursor: pointer;
}

.map-area.unlocked { background: rgba(102,126,234,0.3); }
.map-area.current { border: 2px solid #ffd700; }
.map-area:not(.unlocked) { opacity: 0.5; cursor: not-allowed; }

.area-icon { font-size: 28px; display: block; }
.area-name { font-size: 12px; }

.battle-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0,0,0,0.9);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
}

.battle-title { font-size: 24px; margin-bottom: 20px; }
.battle-units { display: flex; align-items: center; gap: 30px; }
.battle-unit { padding: 10px; background: rgba(255,255,255,0.1); border-radius: 5px; margin: 5px; }
.battle-unit.enemy { background: rgba(255,0,0,0.2); }
.vs { font-size: 24px; color: #ff4444; }

.message-log {
  position: absolute;
  bottom: 60px;
  left: 10px;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.message {
  background: rgba(0,0,0,0.8);
  padding: 4px 10px;
  border-radius: 3px;
  color: white;
  font-size: 12px;
}
</style>
