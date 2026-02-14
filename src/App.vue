<template>
  <div class="game-container">
    <!-- 游戏主画布 -->
    <canvas ref="gameCanvas" class="game-canvas"></canvas>
    
    <!-- 顶部状态栏 -->
    <div class="top-bar">
      <div class="hero-info">
        <span class="hero-name">{{ hero.name }}</span>
        <span class="hero-level">Lv.{{ hero.level }}</span>
        <div class="hp-bar">
          <div class="hp-fill" :style="{ width: (hero.hp / hero.maxHp * 100) + '%' }"></div>
          <span class="hp-text">{{ hero.hp }}/{{ hero.maxHp }}</span>
        </div>
      </div>
      <div class="resources">
        <span>金币: {{ resources.gold }}</span>
        <span>木材: {{ resources.wood }}</span>
        <span>矿石: {{ resources.ore }}</span>
      </div>
    </div>
    
    <!-- 区域信息 -->
    <div class="area-info" v-if="currentArea">
      <span class="area-name">{{ currentArea.name }}</span>
      <span class="area-level">难度: {{ currentArea.level }}</span>
    </div>
    
    <!-- 底部操作栏 -->
    <div class="bottom-bar">
      <button @click="showInventory = !showInventory">背包</button>
      <button @click="showArmy = !showArmy">部队</button>
      <button @click="showSkills = !showSkills">技能</button>
      <button @click="showMap = !showMap">地图</button>
    </div>
    
    <!-- 背包面板 -->
    <div class="panel inventory-panel" v-if="showInventory">
      <div class="panel-header">
        <span>背包</span>
        <button @click="showInventory = false">×</button>
      </div>
      <div class="inventory-grid">
        <div class="inventory-slot" v-for="(item, index) in inventory" :key="index" @click="useItem(index)">
          <span v-if="item">{{ item.name }}</span>
        </div>
      </div>
    </div>
    
    <!-- 部队面板 -->
    <div class="panel army-panel" v-if="showArmy">
      <div class="panel-header">
        <span>部队 ({{ army.length }}/7)</span>
        <button @click="showArmy = false">×</button>
      </div>
      <div class="army-list">
        <div class="army-unit" v-for="(unit, index) in army" :key="index">
          <span class="unit-icon">{{ unit.icon }}</span>
          <span class="unit-name">{{ unit.name }}</span>
          <span class="unit-count">x{{ unit.count }}</span>
        </div>
      </div>
    </div>
    
    <!-- 技能面板 -->
    <div class="panel skills-panel" v-if="showSkills">
      <div class="panel-header">
        <span>技能</span>
        <button @click="showSkills = false">×</button>
      </div>
      <div class="skills-list">
        <div class="skill-item" v-for="(skill, index) in skills" :key="index">
          <span class="skill-name">{{ skill.name }}</span>
          <span class="skill-level">Lv.{{ skill.level }}</span>
        </div>
      </div>
    </div>
    
    <!-- 地图面板 -->
    <div class="panel map-panel" v-if="showMap">
      <div class="panel-header">
        <span>世界地图</span>
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
          <span class="area-status" v-if="!area.unlocked">🔒</span>
        </div>
      </div>
    </div>
    
    <!-- 战斗界面 -->
    <div class="battle-overlay" v-if="inBattle">
      <div class="battle-info">
        <span>战斗中...</span>
      </div>
      <div class="battle-units">
        <div class="our-units">
          <div class="battle-unit" v-for="(unit, index) in army" :key="'ally-'+index">
            <span>{{ unit.icon }}</span>
            <span>{{ unit.count }}</span>
          </div>
        </div>
        <span class="vs">VS</span>
        <div class="enemy-units">
          <div class="battle-unit" v-for="(unit, index) in enemies" :key="'enemy-'+index">
            <span>{{ unit.icon }}</span>
            <span>{{ unit.count }}</span>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 消息提示 -->
    <div class="message-log">
      <div class="message" v-for="(msg, index) in messages" :key="index">
        {{ msg }}
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Game',
  data() {
    return {
      // 画布相关
      canvas: null,
      ctx: null,
      tileSize: 32,
      mapWidth: 20,
      mapHeight: 15,
      
      // 游戏状态
      showInventory: false,
      showArmy: false,
      showSkills: false,
      showMap: false,
      inBattle: false,
      messages: [],
      
      // 英雄数据
      hero: {
        name: '勇者',
        level: 1,
        hp: 100,
        maxHp: 100,
        attack: 10,
        defense: 5,
        speed: 5,
        x: 5,
        y: 7,
        exp: 0,
        expToLevel: 100
      },
      
      // 资源
      resources: {
        gold: 100,
        wood: 10,
        ore: 5
      },
      
      // 部队
      army: [
        { id: 1, name: '步兵', icon: '🗡️', count: 10, attack: 5, defense: 3, hp: 20 },
      ],
      
      // 技能
      skills: [
        { id: 1, name: '领导力', level: 1, description: '增加部队攻击力' },
        { id: 2, name: '防御术', level: 1, description: '增加部队防御力' },
      ],
      
      // 背包
      inventory: [
        { id: 1, name: '治疗药水', type: 'consumable', effect: 'heal', value: 50 },
        null, null, null, null, null, null, null, null, null, null, null
      ],
      
      // 当前地图
      currentMap: [],
      
      // 当前区域
      currentArea: null,
      
      // 世界地图
      worldMap: [
        { id: 1, name: '新手平原', icon: '🌿', level: 1, unlocked: true, completed: false },
        { id: 2, name: '黑暗森林', icon: '🌲', level: 3, unlocked: false, completed: false },
        { id: 3, name: '矿石山脉', icon: '⛰️', level: 5, unlocked: false, completed: false },
        { id: 4, name: '龙之巢穴', icon: '🐉', level: 8, unlocked: false, completed: false },
        { id: 5, name: '魔王城', icon: '🏰', level: 10, unlocked: false, completed: false },
      ],
      
      // 敌人
      enemies: [],
      
      // 地图物体
      mapObjects: [],
      
      // 动画帧
      animationFrame: null,
      lastTime: 0,
      
      // 输入状态
      keys: {},
    }
  },
  mounted() {
    this.initGame();
  },
  beforeUnmount() {
    if (this.animationFrame) {
      cancelAnimationFrame(this.animationFrame);
    }
  },
  methods: {
    // 初始化游戏
    initGame() {
      this.canvas = this.$refs.gameCanvas;
      this.ctx = this.canvas.getContext('2d');
      
      // 设置画布大小
      this.resizeCanvas();
      window.addEventListener('resize', this.resizeCanvas);
      
      // 绑定键盘事件
      window.addEventListener('keydown', this.handleKeyDown);
      window.addEventListener('keyup', this.handleKeyUp);
      
      // 绑定触摸事件
      this.canvas.addEventListener('touchstart', this.handleTouch);
      
      // 初始化地图
      this.currentArea = this.worldMap[0];
      this.generateMap();
      
      // 开始游戏循环
      this.gameLoop();
      
      this.addMessage('欢迎来到英雄无敌世界！');
      this.addMessage('使用方向键或点击移动');
    },
    
    // 调整画布大小
    resizeCanvas() {
      const container = this.canvas.parentElement;
      this.canvas.width = container.clientWidth;
      this.canvas.height = container.clientHeight;
    },
    
    // 生成随机地图
    generateMap() {
      this.currentMap = [];
      this.mapObjects = [];
      
      for (let y = 0; y < this.mapHeight; y++) {
        const row = [];
        for (let x = 0; x < this.mapWidth; x++) {
          // 0: 草地, 1: 墙, 2: 水, 3: 路
          let tile = 0;
          
          // 边界墙
          if (x === 0 || x === this.mapWidth - 1 || y === 0 || y === this.mapHeight - 1) {
            tile = 1;
          }
          // 随机地形
          else if (Math.random() < 0.1) {
            tile = Math.random() < 0.5 ? 1 : 2;
          }
          
          row.push(tile);
        }
        this.currentMap.push(row);
      }
      
      // 生成随机物体
      this.generateObjects();
      
      // 生成敌人
      this.generateEnemies();
      
      // 重置英雄位置
      this.hero.x = 2;
      this.hero.y = 2;
    },
    
    // 生成地图物体
    generateObjects() {
      const objectTypes = [
        { type: 'gold', icon: '💰', name: '金币', effect: { gold: 50 } },
        { type: 'wood', icon: '🪵', name: '木材', effect: { wood: 10 } },
        { type: 'ore', icon: '🪨', name: '矿石', effect: { ore: 5 } },
        { type: 'potion', icon: '🧪', name: '治疗药水', effect: { heal: 30 } },
        { type: 'chest', icon: '📦', name: '宝箱', effect: { random: true } },
        { type: ' recruit', icon: '⚔️', name: '招募点', effect: { recruit: true } },
      ];
      
      const numObjects = 5 + Math.floor(Math.random() * 5);
      
      for (let i = 0; i < numObjects; i++) {
        let x, y;
        do {
          x = Math.floor(Math.random() * (this.mapWidth - 2)) + 1;
          y = Math.floor(Math.random() * (this.mapHeight - 2)) + 1;
        } while (this.currentMap[y][x] !== 0 || (x === this.hero.x && y === this.hero.y));
        
        const objType = objectTypes[Math.floor(Math.random() * objectTypes.length)];
        this.mapObjects.push({
          ...objType,
          x,
          y
        });
      }
    },
    
    // 生成敌人
    generateEnemies() {
      const enemyTypes = [
        { name: '哥布林', icon: '👺', hp: 20, attack: 5, exp: 10 },
        { name: '骷髅', icon: '💀', hp: 15, attack: 8, exp: 15 },
        { name: '狼', icon: '🐺', hp: 25, attack: 10, exp: 20 },
        { name: '兽人', icon: '👹', hp: 40, attack: 15, exp: 30 },
      ];
      
      const numEnemies = 3 + Math.floor(Math.random() * 3);
      
      for (let i = 0; i < numEnemies; i++) {
        let x, y;
        do {
          x = Math.floor(Math.random() * (this.mapWidth - 2)) + 1;
          y = Math.floor(Math.random() * (this.mapHeight - 2)) + 1;
        } while (
          this.currentMap[y][x] !== 0 || 
          (Math.abs(x - this.hero.x) < 3 && Math.abs(y - this.hero.y) < 3)
        );
        
        const enemyType = enemyTypes[Math.floor(Math.random() * enemyTypes.length)];
        this.mapObjects.push({
          type: 'enemy',
          ...enemyType,
          x,
          y,
          currentHp: enemyType.hp
        });
      }
    },
    
    // 游戏主循环
    gameLoop(timestamp = 0) {
      const deltaTime = timestamp - this.lastTime;
      this.lastTime = timestamp;
      
      this.update(deltaTime);
      this.render();
      
      this.animationFrame = requestAnimationFrame(this.gameLoop);
    },
    
    // 更新游戏状态
    update(deltaTime) {
      // 处理移动
      const speed = 5;
      if (this.keys['ArrowUp'] || this.keys['w']) this.moveHero(0, -1);
      if (this.keys['ArrowDown'] || this.keys['s']) this.moveHero(0, 1);
      if (this.keys['ArrowLeft'] || this.keys['a']) this.moveHero(-1, 0);
      if (this.keys['ArrowRight'] || this.keys['d']) this.moveHero(1, 0);
    },
    
    // 移动英雄
    moveHero(dx, dy) {
      const newX = this.hero.x + dx;
      const newY = this.hero.y + dy;
      
      // 边界检查
      if (newX < 0 || newX >= this.mapWidth || newY < 0 || newY >= this.mapHeight) return;
      
      // 地形检查
      if (this.currentMap[newY][newX] === 1) return; // 墙
      if (this.currentMap[newY][newX] === 2) return; // 水
      
      // 移动
      this.hero.x = newX;
      this.hero.y = newY;
      
      // 检查物体碰撞
      this.checkObjectCollision();
      
      // 防止连续移动
      this.keys = {};
    },
    
    // 检查物体碰撞
    checkObjectCollision() {
      const objIndex = this.mapObjects.findIndex(
        obj => obj.x === this.hero.x && obj.y === this.hero.y
      );
      
      if (objIndex !== -1) {
        const obj = this.mapObjects[objIndex];
        
        if (obj.type === 'enemy') {
          this.startBattle(obj, objIndex);
        } else {
          this.collectObject(obj, objIndex);
        }
      }
    },
    
    // 收集物体
    collectObject(obj, index) {
      if (obj.effect.gold) {
        this.resources.gold += obj.effect.gold;
        this.addMessage(`获得 ${obj.effect.gold} 金币！`);
      }
      if (obj.effect.wood) {
        this.resources.wood += obj.effect.wood;
        this.addMessage(`获得 ${obj.effect.wood} 木材！`);
      }
      if (obj.effect.ore) {
        this.resources.ore += obj.effect.ore;
        this.addMessage(`获得 ${obj.effect.ore} 矿石！`);
      }
      if (obj.effect.heal) {
        this.hero.hp = Math.min(this.hero.maxHp, this.hero.hp + obj.effect.heal);
        this.addMessage(`恢复了 ${obj.effect.heal} 生命值！`);
      }
      if (obj.effect.recruit) {
        this.recruitUnit();
      }
      
      this.mapObjects.splice(index, 1);
    },
    
    // 招募单位
    recruitUnit() {
      const unitTypes = [
        { name: '步兵', icon: '🗡️', attack: 5, defense: 3, hp: 20 },
        { name: '弓箭手', icon: '🏹', attack: 8, defense: 2, hp: 15 },
        { name: '骑兵', icon: '🐴', attack: 10, defense: 4, hp: 30 },
      ];
      
      if (this.army.length < 7 && this.resources.gold >= 50) {
        const unitType = unitTypes[Math.floor(Math.random() * unitTypes.length)];
        const existingUnit = this.army.find(u => u.name === unitType.name);
        
        if (existingUnit) {
          existingUnit.count += 5;
        } else {
          this.army.push({
            id: Date.now(),
            ...unitType,
            count: 5
          });
        }
        
        this.resources.gold -= 50;
        this.addMessage(`招募了 ${unitType.name}！`);
      } else if (this.army.length >= 7) {
        this.addMessage('部队已满！');
      } else {
        this.addMessage('金币不足！');
      }
    },
    
    // 开始战斗
    startBattle(enemy, index) {
      this.inBattle = true;
      this.enemies = [{ ...enemy, count: 3 + Math.floor(Math.random() * 5) }];
      
      this.addMessage(`遭遇 ${enemy.name}！`);
      
      // 模拟战斗
      setTimeout(() => {
        this.resolveBattle(enemy, index);
      }, 2000);
    },
    
    // 解决战斗
    resolveBattle(enemy, index) {
      // 简单的战斗计算
      const ourPower = this.army.reduce((sum, u) => sum + u.attack * u.count, 0) + this.hero.attack;
      const enemyPower = this.enemies.reduce((sum, e) => sum + e.attack * e.count, 0);
      
      if (ourPower > enemyPower) {
        this.addMessage(`战斗胜利！获得 ${enemy.exp} 经验值！`);
        this.hero.exp += enemy.exp;
        
        // 检查升级
        if (this.hero.exp >= this.hero.expToLevel) {
          this.levelUp();
        }
        
        this.mapObjects.splice(index, 1);
        
        // 检查区域完成
        this.checkAreaComplete();
      } else {
        this.addMessage('战斗失败...失去一些金币');
        this.resources.gold = Math.max(0, this.resources.gold - 50);
      }
      
      this.inBattle = false;
      this.enemies = [];
    },
    
    // 英雄升级
    levelUp() {
      this.hero.level++;
      this.hero.exp -= this.hero.expToLevel;
      this.hero.expToLevel = Math.floor(this.hero.expToLevel * 1.5);
      this.hero.maxHp += 10;
      this.hero.hp = this.hero.maxHp;
      this.hero.attack += 2;
      this.hero.defense += 1;
      
      this.addMessage(`升级了！现在是 ${this.hero.level} 级！`);
    },
    
    // 检查区域完成
    checkAreaComplete() {
      const enemiesLeft = this.mapObjects.filter(obj => obj.type === 'enemy').length;
      
      if (enemiesLeft === 0 && this.currentArea) {
        this.currentArea.completed = true;
        this.addMessage(`${this.currentArea.name} 已清除！`);
        
        // 解锁下一个区域
        const currentIndex = this.worldMap.findIndex(a => a.id === this.currentArea.id);
        if (currentIndex < this.worldMap.length - 1) {
          this.worldMap[currentIndex + 1].unlocked = true;
          this.addMessage(`新区域已解锁：${this.worldMap[currentIndex + 1].name}`);
        }
      }
    },
    
    // 前往区域
    travelTo(area) {
      if (!area.unlocked) {
        this.addMessage('该区域尚未解锁！');
        return;
      }
      
      if (area.id === this.currentArea.id) {
        this.addMessage('你已在该区域');
        return;
      }
      
      this.currentArea = area;
      this.generateMap();
      this.showMap = false;
      this.addMessage(`前往 ${area.name}`);
    },
    
    // 使用物品
    useItem(index) {
      const item = this.inventory[index];
      if (!item) return;
      
      if (item.type === 'consumable') {
        if (item.effect === 'heal') {
          this.hero.hp = Math.min(this.hero.maxHp, this.hero.hp + item.value);
          this.addMessage(`使用了 ${item.name}，恢复 ${item.value} 生命值`);
        }
        this.inventory[index] = null;
      }
    },
    
    // 添加消息
    addMessage(msg) {
      this.messages.push(msg);
      if (this.messages.length > 5) {
        this.messages.shift();
      }
    },
    
    // 渲染
    render() {
      const ctx = this.ctx;
      const ts = this.tileSize;
      
      // 清空画布
      ctx.fillStyle = '#1a1a2e';
      ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
      
      // 计算偏移使地图居中
      const offsetX = (this.canvas.width - this.mapWidth * ts) / 2;
      const offsetY = (this.canvas.height - this.mapHeight * ts) / 2;
      
      ctx.save();
      ctx.translate(offsetX, offsetY);
      
      // 绘制地图
      for (let y = 0; y < this.mapHeight; y++) {
        for (let x = 0; x < this.mapWidth; x++) {
          const tile = this.currentMap[y][x];
          
          switch (tile) {
            case 0: ctx.fillStyle = '#4a7c59'; break; // 草地
            case 1: ctx.fillStyle = '#4a4a4a'; break; // 墙
            case 2: ctx.fillStyle = '#4a90a4'; break; // 水
            case 3: ctx.fillStyle = '#8b7355'; break; // 路
          }
          
          ctx.fillRect(x * ts, y * ts, ts - 1, ts - 1);
        }
      }
      
      // 绘制物体
      ctx.font = `${ts - 4}px Arial`;
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      
      for (const obj of this.mapObjects) {
        ctx.fillText(obj.icon, obj.x * ts + ts / 2, obj.y * ts + ts / 2);
      }
      
      // 绘制英雄
      ctx.fillText('🧙', this.hero.x * ts + ts / 2, this.hero.y * ts + ts / 2);
      
      ctx.restore();
    },
    
    // 键盘事件
    handleKeyDown(e) {
      this.keys[e.key] = true;
    },
    
    handleKeyUp(e) {
      this.keys[e.key] = false;
    },
    
    // 触摸事件 - 改进的移动控制
    handleTouch(e) {
      e.preventDefault();
      const touch = e.touches[0];
      const rect = this.canvas.getBoundingClientRect();
      const x = touch.clientX - rect.left;
      const y = touch.clientY - rect.top;
      
      const offsetX = (this.canvas.width - this.mapWidth * this.tileSize) / 2;
      const offsetY = (this.canvas.height - this.mapHeight * this.tileSize) / 2;
      
      const tileX = Math.floor((x - offsetX) / this.tileSize);
      const tileY = Math.floor((y - offsetY) / this.tileSize);
      
      const dx = tileX - this.hero.x;
      const dy = tileY - this.hero.y;
      
      // 根据点击方向决定移动方向（垂直方向优先）
      if (Math.abs(dy) >= Math.abs(dx)) {
        // 垂直移动优先
        if (dy !== 0) this.moveHero(0, Math.sign(dy));
      } else {
        // 水平移动
        if (dx !== 0) this.moveHero(Math.sign(dx), 0);
      }
    },
    
    // 点击事件（PC端）
    handleClick(e) {
      const rect = this.canvas.getBoundingClientRect();
      const x = e.clientX - rect.left;
      const y = e.clientY - rect.top;
      
      const offsetX = (this.canvas.width - this.mapWidth * this.tileSize) / 2;
      const offsetY = (this.canvas.height - this.mapHeight * this.tileSize) / 2;
      
      const tileX = Math.floor((x - offsetX) / this.tileSize);
      const tileY = Math.floor((y - offsetY) / this.tileSize);
      
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
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  position: relative;
  overflow: hidden;
  font-family: 'Arial', sans-serif;
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
  padding: 10px;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: white;
  font-size: 14px;
}

.hero-info {
  display: flex;
  align-items: center;
  gap: 10px;
}

.hero-name {
  font-weight: bold;
  color: #ffd700;
}

.hero-level {
  color: #7fff7f;
}

.hp-bar {
  width: 100px;
  height: 12px;
  background: #333;
  border-radius: 6px;
  position: relative;
  overflow: hidden;
}

.hp-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff4444, #ff6666);
  transition: width 0.3s;
}

.hp-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 10px;
  color: white;
}

.resources {
  display: flex;
  gap: 15px;
}

.resources span {
  color: #ffd700;
}

.area-info {
  position: absolute;
  top: 50px;
  left: 10px;
  background: rgba(0, 0, 0, 0.7);
  padding: 8px 15px;
  border-radius: 5px;
  color: white;
  font-size: 12px;
}

.area-name {
  font-weight: bold;
  color: #7fff7f;
}

.area-level {
  margin-left: 10px;
  color: #ff7f7f;
}

.bottom-bar {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 10px;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  gap: 10px;
}

.bottom-bar button {
  padding: 10px 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 14px;
  cursor: pointer;
}

.bottom-bar button:hover {
  opacity: 0.9;
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
  min-width: 300px;
  max-width: 90vw;
  max-height: 80vh;
  overflow-y: auto;
  color: white;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #667eea;
}

.panel-header span {
  font-size: 18px;
  font-weight: bold;
}

.panel-header button {
  background: none;
  border: none;
  color: white;
  font-size: 24px;
  cursor: pointer;
}

.inventory-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
}

.inventory-slot {
  width: 60px;
  height: 60px;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid #444;
  border-radius: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.inventory-slot:hover {
  border-color: #667eea;
}

.army-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.army-unit {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 5px;
}

.unit-icon {
  font-size: 24px;
}

.unit-name {
  flex: 1;
}

.unit-count {
  color: #ffd700;
}

.skills-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.skill-item {
  display: flex;
  justify-content: space-between;
  padding: 10px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 5px;
}

.skill-level {
  color: #7fff7f;
}

.world-map {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
}

.map-area {
  padding: 15px;
  background: rgba(100, 100, 100, 0.3);
  border-radius: 10px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s;
}

.map-area.unlocked {
  background: rgba(102, 126, 234, 0.3);
}

.map-area.current {
  border: 2px solid #ffd700;
}

.map-area:not(.unlocked) {
  opacity: 0.5;
  cursor: not-allowed;
}

.area-icon {
  font-size: 32px;
  display: block;
  margin-bottom: 5px;
}

.map-area .area-name {
  display: block;
  font-size: 14px;
}

.battle-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
}

.battle-info {
  font-size: 24px;
  margin-bottom: 20px;
}

.battle-units {
  display: flex;
  align-items: center;
  gap: 30px;
}

.battle-unit {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  font-size: 24px;
}

.vs {
  font-size: 32px;
  color: #ff4444;
}

.message-log {
  position: absolute;
  bottom: 60px;
  left: 10px;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.message {
  background: rgba(0, 0, 0, 0.7);
  padding: 5px 10px;
  border-radius: 3px;
  color: white;
  font-size: 12px;
  animation: fadeIn 0.3s;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateX(-20px); }
  to { opacity: 1; transform: translateX(0); }
}
</style>
