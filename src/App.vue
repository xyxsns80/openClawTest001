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
          <span class="hp-text">{{ hero.hp }}/{{ hero.maxHp }}</span>
        </div>
        <div class="exp-bar">
          <div class="exp-fill" :style="{ width: (hero.exp / hero.expToLevel * 100) + '%' }"></div>
        </div>
        <span class="hero-stats-mini">⚔️{{ totalAttack }} 🛡️{{ totalDefense }} 🔥{{ skillPoints }}</span>
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
      <button @click="showShop = !showShop">🏪商店</button>
      <button @click="showEquipment = !showEquipment">🎒装备</button>
      <button @click="showEnhance = !showEnhance">⚒️强化</button>
      <button @click="showSkills = !showSkills">🔥技能</button>
      <button @click="autoBattle = !autoBattle" :class="{ active: autoBattle }">🤖自动</button>
      <button @click="saveGame">💾存档</button>
    </div>
    
    <!-- 存档提示 -->
    <div class="save-toast" v-if="showSaveToast">
      {{ saveToastMsg }}
    </div>
    
    <!-- 技能栏（战斗中显示） -->
    <div class="skill-bar" v-if="inBattle">
      <div 
        class="skill-slot" 
        v-for="skill in skills.filter(s => s.unlocked)" 
        :key="skill.id"
        :class="{ disabled: skill.currentCd > 0 }"
        @click="useSkill(skill)"
      >
        <span class="skill-icon">{{ skill.icon }}</span>
        <span class="skill-cd" v-if="skill.currentCd > 0">{{ skill.currentCd }}</span>
      </div>
    </div>
    
    <!-- 技能面板 -->
    <div class="panel" v-if="showSkills">
      <div class="panel-header">
        <span>🔥 技能 (点数: {{ skillPoints }})</span>
        <button @click="showSkills = false">×</button>
      </div>
      <div class="skill-list">
        <div class="skill-item" v-for="skill in skills" :key="skill.id">
          <span class="skill-icon">{{ skill.icon }}</span>
          <div class="skill-info">
            <span class="skill-name">{{ skill.name }}</span>
            <span class="skill-desc">{{ skill.desc }}</span>
            <span class="skill-cd-info">冷却: {{ skill.cooldown }}回合</span>
          </div>
          <button 
            v-if="!skill.unlocked && skillPoints > 0" 
            @click="unlockSkill(skill)"
            class="unlock-btn"
          >
            解锁
          </button>
          <span v-if="skill.unlocked" class="unlocked-badge">✓</span>
        </div>
      </div>
    </div>
    
    <!-- 成就面板 -->
    <div class="panel" v-if="showAchievements">
      <div class="panel-header">
        <span>🏆 成就</span>
        <button @click="showAchievements = false">×</button>
      </div>
      <div class="achievement-list">
        <div class="achievement-item" v-for="ach in achievements" :key="ach.id" :class="{ unlocked: ach.unlocked }">
          <span class="ach-icon">{{ ach.icon }}</span>
          <div class="ach-info">
            <span class="ach-name">{{ ach.name }}</span>
            <span class="ach-desc">{{ ach.desc }}</span>
          </div>
          <span class="ach-reward" v-if="ach.unlocked">✓</span>
          <span class="ach-reward" v-else>{{ ach.rewardType === 'skillPoint' ? '技能点' : '💰' + ach.reward }}</span>
        </div>
      </div>
    </div>
    
    <!-- 部队面板 -->
    <div class="panel" v-if="showArmy">
      <div class="panel-header">
        <span>⚔️ 部队</span>
        <button @click="showArmy = false">×</button>
      </div>
      <div class="army-list">
        <div class="army-unit" v-for="(unit, index) in army" :key="index" @click="showUnitInfo(unit)">
          <span class="unit-icon">{{ unit.icon }}</span>
          <div class="unit-details">
            <span class="unit-name">{{ unit.name }}</span>
            <span class="unit-stats">⚔️{{ unit.attack }} 🛡️{{ unit.defense }}</span>
          </div>
          <span class="unit-count">x{{ unit.count }}</span>
        </div>
        <div v-if="army.length === 0" class="empty-army">暂无部队</div>
      </div>
    </div>
    
    <!-- 单位详情弹窗 -->
    <div class="panel unit-info-panel" v-if="selectedUnit">
      <div class="panel-header">
        <span>{{ selectedUnit.icon }} {{ selectedUnit.name }}</span>
        <button @click="selectedUnit = null">×</button>
      </div>
      <div class="unit-info-content">
        <div class="info-row">
          <span class="info-label">数量</span>
          <span class="info-value">x{{ selectedUnit.count }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">攻击力</span>
          <span class="info-value">⚔️ {{ selectedUnit.attack }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">防御力</span>
          <span class="info-value">🛡️ {{ selectedUnit.defense }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">总攻击</span>
          <span class="info-value highlight">⚔️ {{ selectedUnit.attack * selectedUnit.count }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">总防御</span>
          <span class="info-value highlight">🛡️ {{ selectedUnit.defense * selectedUnit.count }}</span>
        </div>
        <div class="unit-actions" v-if="!inBattle">
          <button @click="reinforceUnit(selectedUnit)" :disabled="resources.gold < 50">
            增援 (50金)
          </button>
        </div>
      </div>
    </div>
    
    <!-- 装备强化面板 -->
    <div class="panel enhance-panel" v-if="showEnhance">
      <div class="panel-header">
        <span>⚒️ 装备强化</span>
        <button @click="showEnhance = false">×</button>
      </div>
      <div class="enhance-list">
        <div class="enhance-item" v-for="(item, i) in equippableItems" :key="i">
          <span class="item-icon">{{ item.icon }}</span>
          <div class="item-info">
            <span class="item-name">{{ item.name }}</span>
            <span class="item-stats">+{{ item.bonus }} {{ item.type }} (Lv.{{ item.enhanceLevel || 0 }})</span>
          </div>
          <button 
            @click="enhanceItem(item, i)" 
            :disabled="resources.gold < getEnhanceCost(item)"
          >
            强化 ({{ getEnhanceCost(item) }}金)
          </button>
        </div>
      </div>
    </div>
    
    <!-- 商店面板 -->
    <div class="panel shop-panel" v-if="showShop">
      <div class="panel-header">
        <span>🏪 商店</span>
        <button @click="showShop = false">×</button>
      </div>
      <div class="shop-tabs">
        <button :class="{ active: shopTab === 'units' }" @click="shopTab = 'units'">兵种</button>
        <button :class="{ active: shopTab === 'equipment' }" @click="shopTab = 'equipment'">装备</button>
        <button :class="{ active: shopTab === 'items' }" @click="shopTab = 'items'">道具</button>
      </div>
      <div class="shop-items" v-if="shopTab === 'units'">
        <div class="shop-item" v-for="unit in shopUnits" :key="unit.id" @click="buyUnit(unit)">
          <span class="item-icon">{{ unit.icon }}</span>
          <div class="item-info">
            <span class="item-name">{{ unit.name }}</span>
            <span class="item-stats">攻:{{ unit.attack }} 防:{{ unit.defense }}</span>
          </div>
          <span class="item-price">💰{{ unit.price }}</span>
        </div>
      </div>
      <div class="shop-items" v-if="shopTab === 'equipment'">
        <div class="shop-item" v-for="eq in shopEquipment" :key="eq.id" @click="buyEquipment(eq)">
          <span class="item-icon">{{ eq.icon }}</span>
          <div class="item-info">
            <span class="item-name">{{ eq.name }}</span>
            <span class="item-stats">{{ eq.type }} +{{ eq.bonus }}</span>
          </div>
          <span class="item-price">💰{{ eq.price }}</span>
        </div>
      </div>
      <div class="shop-items" v-if="shopTab === 'items'">
        <div class="shop-item" v-for="item in shopItems" :key="item.id" @click="buyItem(item)">
          <span class="item-icon">{{ item.icon }}</span>
          <div class="item-info">
            <span class="item-name">{{ item.name }}</span>
            <span class="item-stats">{{ item.desc }}</span>
          </div>
          <span class="item-price">💰{{ item.price }}</span>
        </div>
      </div>
    </div>

    <!-- 装备面板 -->
    <div class="panel equipment-panel" v-if="showEquipment">
      <div class="panel-header">
        <span>🎒 装备 & 背包</span>
        <button @click="showEquipment = false">×</button>
      </div>
      <div class="equipped-section">
        <div class="section-title">已装备</div>
        <div class="equipped-slots">
          <div class="equip-slot" :class="{ empty: !hero.equipment.weapon }">
            <span class="slot-label">武器</span>
            <span v-if="hero.equipment.weapon">{{ hero.equipment.weapon.icon }} {{ hero.equipment.weapon.name }}</span>
            <span v-else class="empty-slot">空</span>
          </div>
          <div class="equip-slot" :class="{ empty: !hero.equipment.armor }">
            <span class="slot-label">护甲</span>
            <span v-if="hero.equipment.armor">{{ hero.equipment.armor.icon }} {{ hero.equipment.armor.name }}</span>
            <span v-else class="empty-slot">空</span>
          </div>
          <div class="equip-slot" :class="{ empty: !hero.equipment.accessory }">
            <span class="slot-label">饰品</span>
            <span v-if="hero.equipment.accessory">{{ hero.equipment.accessory.icon }} {{ hero.equipment.accessory.name }}</span>
            <span v-else class="empty-slot">空</span>
          </div>
        </div>
      </div>
      <div class="inventory-section">
        <div class="section-title">背包</div>
        <div class="inventory-grid">
          <div 
            class="inventory-item" 
            v-for="(item, i) in inventory" 
            :key="i"
            @click="useItem(item, i)"
          >
            {{ item.icon }} {{ item.name }}
          </div>
          <div v-if="inventory.length === 0" class="empty-inventory">空空如也</div>
        </div>
      </div>
      <div class="hero-stats">
        <div class="section-title">英雄属性</div>
        <div class="stats-grid">
          <span>攻击: {{ totalAttack }}</span>
          <span>防御: {{ totalDefense }}</span>
          <span>生命: {{ hero.maxHp }}</span>
          <span>暴击: {{ hero.crit || 0 }}%</span>
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
      <div class="battle-box">
        <div class="battle-title">⚔️ 战斗</div>
        
        <div class="battle-sides">
          <!-- 我方 -->
          <div class="battle-side our-side">
            <div class="side-label">我方战力: {{ ourBattlePower }}</div>
            <div class="hero-card">
              <span class="card-icon">🧙</span>
              <div class="card-info">
                <span class="card-name">{{ hero.name }} Lv.{{ hero.level }}</span>
                <div class="hp-bar-mini">
                  <div class="hp-fill" :style="{ width: (hero.hp / hero.maxHp * 100) + '%' }"></div>
                </div>
              </div>
            </div>
            <div class="unit-cards">
              <div class="unit-card" v-for="(unit, i) in army" :key="'a'+i">
                <span class="card-icon">{{ unit.icon }}</span>
                <span class="card-count">x{{ unit.count }}</span>
              </div>
            </div>
          </div>
          
          <div class="battle-vs">⚔️</div>
          
          <!-- 敌方 -->
          <div class="battle-side enemy-side">
            <div class="side-label">敌方战力: {{ enemyBattlePower }}</div>
            <div class="unit-cards">
              <div class="unit-card enemy" v-for="(e, i) in battleEnemies" :key="'e'+i">
                <span class="card-icon">{{ e.icon }}</span>
                <span class="card-name">{{ e.name }}</span>
                <span class="card-count">x{{ e.count }}</span>
                <span class="boss-badge" v-if="e.isBoss">BOSS</span>
              </div>
            </div>
          </div>
        </div>
        
        <!-- 战斗日志 -->
        <div class="battle-log">
          <div v-for="(log, i) in battleLog" :key="i" :class="['log-item', log.type]">
            {{ log.text }}
          </div>
        </div>
        
        <!-- 战斗进度条 -->
        <div class="battle-progress">
          <div class="progress-bar">
            <div class="progress-fill" :style="{ width: battleProgress + '%' }"></div>
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
      showShop: false,
      showEquipment: false,
      showSkills: false,
      showAchievements: false,
      showEnhance: false,
      autoBattle: false, // 自动战斗模式
      showSaveToast: false,
      saveToastMsg: '',
      shopTab: 'units',
      inBattle: false,
      messages: [],
      selectedUnit: null,
      
      // 技能系统
      skills: [
        { id: 1, name: '火焰冲击', icon: '🔥', desc: '造成200%攻击伤害', cooldown: 3, currentCd: 0, unlocked: true },
        { id: 2, name: '雷霆一击', icon: '⚡', desc: '造成150%伤害+眩晕', cooldown: 5, currentCd: 0, unlocked: false },
        { id: 3, name: '治愈之光', icon: '💚', desc: '恢复30%最大HP', cooldown: 4, currentCd: 0, unlocked: false },
        { id: 4, name: '战吼', icon: '📢', desc: '本次战斗攻击+50%', cooldown: 6, currentCd: 0, unlocked: false },
      ],
      skillPoints: 0,
      battleBuff: 1,
      
      // 成就系统
      achievements: [
        { id: 1, name: '初出茅庐', icon: '🗡️', desc: '击败第一个敌人', condition: 'killFirstEnemy', unlocked: false, reward: 50 },
        { id: 2, name: '屠龙者', icon: '🐉', desc: '击败第一个Boss', condition: 'killFirstBoss', unlocked: false, reward: 200 },
        { id: 3, name: '富甲一方', icon: '💰', desc: '累计获得1000金币', condition: 'gold1000', unlocked: false, reward: 1, rewardType: 'skillPoint' },
        { id: 4, name: '老兵', icon: '⭐', desc: '达到10级', condition: 'level10', unlocked: false, reward: 2, rewardType: 'skillPoint' },
        { id: 5, name: '征服者', icon: '👑', desc: '通关第一个区域', condition: 'completeArea', unlocked: false, rewardType: 'legendary' },
        { id: 6, name: '军团长', icon: '⚔️', desc: '拥有5种兵种', condition: 'army5', unlocked: false, reward: 300 },
      ],
      totalGoldEarned: 0,
      
      // 英雄
      hero: {
        name: '勇者',
        level: 1,
        hp: 100,
        maxHp: 100,
        attack: 10,
        defense: 5,
        crit: 5,
        x: 5,
        y: 5,
        exp: 0,
        expToLevel: 100,
        equipment: {
          weapon: null,
          armor: null,
          accessory: null
        }
      },

      // 背包
      inventory: [],

      // 商店 - 兵种
      shopUnits: [
        { id: 1, name: '步兵', icon: '🗡️', attack: 5, defense: 3, price: 50, count: 5 },
        { id: 2, name: '弓箭手', icon: '🏹', attack: 8, defense: 2, price: 80, count: 3 },
        { id: 3, name: '骑士', icon: '🐴', attack: 12, defense: 8, price: 150, count: 2 },
        { id: 4, name: '法师', icon: '🔮', attack: 15, defense: 1, price: 200, count: 2 },
        { id: 5, name: '天使', icon: '👼', attack: 25, defense: 15, price: 500, count: 1 },
        { id: 6, name: '巨龙', icon: '🐉', attack: 40, defense: 20, price: 1000, count: 1 },
        { id: 7, name: '泰坦', icon: '🗿', attack: 35, defense: 25, price: 800, count: 1 },
        { id: 8, name: '恶魔', icon: '😈', attack: 50, defense: 10, price: 1200, count: 1 },
      ],

      // 商店 - 装备
      shopEquipment: [
        { id: 1, name: '铁剑', icon: '⚔️', type: '攻击', bonus: 5, price: 100, slot: 'weapon' },
        { id: 2, name: '钢剑', icon: '🗡️', type: '攻击', bonus: 12, price: 300, slot: 'weapon' },
        { id: 3, name: '传说之剑', icon: '⚡', type: '攻击', bonus: 25, price: 800, slot: 'weapon' },
        { id: 4, name: '皮甲', icon: '🛡️', type: '防御', bonus: 5, price: 100, slot: 'armor' },
        { id: 5, name: '板甲', icon: '🛡️', type: '防御', bonus: 15, price: 350, slot: 'armor' },
        { id: 6, name: '龙鳞甲', icon: '🐉', type: '防御', bonus: 30, price: 900, slot: 'armor' },
        { id: 7, name: '幸运戒指', icon: '💍', type: '暴击', bonus: 10, price: 200, slot: 'accessory' },
        { id: 8, name: '力量项链', icon: '📿', type: '攻击', bonus: 8, price: 250, slot: 'accessory' },
      ],

      // 商店 - 道具
      shopItems: [
        { id: 1, name: '生命药水', icon: '🧪', desc: '恢复50HP', price: 30, effect: { heal: 50 } },
        { id: 2, name: '大生命药水', icon: '⚗️', desc: '恢复100HP', price: 60, effect: { heal: 100 } },
        { id: 3, name: '经验卷轴', icon: '📜', desc: '+50经验', price: 100, effect: { exp: 50 } },
        { id: 4, name: '召唤卷轴', icon: '📃', desc: '召唤步兵x10', price: 80, effect: { summon: 'infantry' } },
      ],
      
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
      
      // 战斗相关
      battleEnemies: [],
      battleLog: [],
      battleProgress: 0,
      
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
    },
    totalAttack() {
      let atk = this.hero.attack;
      if (this.hero.equipment.weapon) atk += this.hero.equipment.weapon.bonus;
      if (this.hero.equipment.accessory && this.hero.equipment.accessory.type === '攻击') {
        atk += this.hero.equipment.accessory.bonus;
      }
      return atk;
    },
    totalDefense() {
      let def = this.hero.defense;
      if (this.hero.equipment.armor) def += this.hero.equipment.armor.bonus;
      return def;
    },
    ourBattlePower() {
      return this.army.reduce((s, u) => s + u.attack * u.count, 0) + this.totalAttack;
    },
    enemyBattlePower() {
      return this.battleEnemies.reduce((s, e) => s + e.attack * (e.count || 1), 0);
    },
    equippableItems() {
      return this.inventory.filter(i => i.isEquipment);
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
      
      // 尝试加载存档
      if (this.loadGame()) {
        this.addMessage('💾 存档已加载！');
      }
      
      this.currentArea = this.currentArea || this.worldMap[0];
      this.currentRegion = this.currentRegion || 0;
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
        { name: '狼人', icon: '🐺', hp: 35, attack: 12, exp: 30, gold: 35 },
        { name: '亡灵', icon: '👻', hp: 30, attack: 8, exp: 20, gold: 25 },
        { name: '精灵弓手', icon: '🧝', hp: 25, attack: 15, exp: 35, gold: 50 },
        { name: '黑骑士', icon: '🖤', hp: 60, attack: 18, exp: 50, gold: 80 },
      ];
      
      // Boss列表
      const bosses = [
        { name: '哥布林王', icon: '👑', hp: 100, attack: 15, exp: 100, gold: 200, isBoss: true },
        { name: '森林巨魔', icon: '🌳', hp: 150, attack: 20, exp: 150, gold: 300, isBoss: true },
        { name: '石像巨人', icon: '🗿', hp: 200, attack: 25, exp: 200, gold: 400, isBoss: true },
        { name: '黑暗领主', icon: '🦹', hp: 300, attack: 35, exp: 500, gold: 800, isBoss: true },
        { name: '魔王', icon: '😈', hp: 500, attack: 50, exp: 1000, gold: 2000, isBoss: true },
      ];

      // 最后一个区域放Boss
      if (this.currentRegion === this.regionCount - 1) {
        const areaIndex = this.worldMap.findIndex(a => a.id === this.currentArea.id);
        const boss = bosses[Math.min(areaIndex, bosses.length - 1)];
        const pos = this.findEmptyTile(true);
        if (pos) {
          this.mapObjects.push({
            type: 'enemy',
            ...boss,
            x: pos.x,
            y: pos.y
          });
        }
      }

      const count = 3 + level * 2;
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
      
      // 自动战斗模式
      if (this.autoBattle && !this.inBattle) {
        this.autoMoveDelay = (this.autoMoveDelay || 0) - dt;
        if (this.autoMoveDelay <= 0) {
          this.autoMove();
          this.autoMoveDelay = 300; // 300ms移动一次
        }
      }
      
      this.animationFrame = requestAnimationFrame(this.gameLoop);
    },
    
    autoMove() {
      // 找到最近的敌人
      let nearestEnemy = null;
      let minDist = Infinity;
      
      for (const obj of this.mapObjects) {
        if (obj.type === 'enemy') {
          const dist = Math.abs(obj.x - this.hero.x) + Math.abs(obj.y - this.hero.y);
          if (dist < minDist) {
            minDist = dist;
            nearestEnemy = obj;
          }
        }
      }
      
      if (nearestEnemy) {
        // 向敌人移动
        const dx = Math.sign(nearestEnemy.x - this.hero.x);
        const dy = Math.sign(nearestEnemy.y - this.hero.y);
        
        if (dx !== 0 && Math.random() > 0.5) {
          this.moveHero(dx, 0);
        } else if (dy !== 0) {
          this.moveHero(0, dy);
        } else if (dx !== 0) {
          this.moveHero(dx, 0);
        }
      } else {
        // 没有敌人，寻找关口
        const nextGate = this.regionGates.find(g => g.targetRegion === this.currentRegion + 1);
        if (nextGate && this.enemiesRemaining === 0) {
          const dx = Math.sign(nextGate.x - this.hero.x);
          const dy = Math.sign(nextGate.y - this.hero.y);
          if (dx !== 0) this.moveHero(dx, 0);
          else if (dy !== 0) this.moveHero(0, dy);
        }
      }
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
        this.addMessage('还有敌人，传送门无法激活！');
        return;
      }
      
      // 只能进入下一个区域
      if (gate.targetRegion !== this.currentRegion + 1) {
        this.addMessage('请先通过前面的区域！');
        return;
      }
      
      if (gate.targetRegion < this.regionCount) {
        // 传送动画效果
        this.addMessage(`🌀 传送门启动...`);
        
        // 立即执行传送
        this.currentRegion = gate.targetRegion;
        // 传送到新区域的起始位置
        this.hero.x = 3;
        this.hero.y = this.currentRegion * 6 + 2;
        this.addMessage(`📍 传送到区域 ${this.currentRegion + 1}`);
        
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
      this.battleLog = [];
      this.battleProgress = 0;
      this.battleBuff = 1; // 重置战斗buff
      
      const enemyCount = enemy.isBoss ? 1 : (3 + Math.floor(Math.random() * 3));
      this.battleEnemies = [{ ...enemy, count: enemyCount }];

      this.battleLog.push({ text: `遭遇 ${enemy.name}${enemyCount > 1 ? ' x' + enemyCount : ''}！`, type: 'info' });
      
      if (enemy.isBoss) {
        this.battleLog.push({ text: `⚠️ Boss战！`, type: 'warning' });
      }

      // 战斗动画
      this.animateBattle(enemy);
    },
    
    animateBattle(enemy) {
      let progress = 0;
      const interval = setInterval(() => {
        progress += 10;
        this.battleProgress = progress;
        
        // 添加战斗日志 - 区分伤害来源
        if (progress === 20) {
          // 英雄攻击
          const heroDmg = Math.floor(this.totalAttack * 0.5);
          this.battleLog.push({ text: `🧙 ${this.hero.name} 攻击，造成 ${heroDmg} 伤害！`, type: 'our' });
        }
        if (progress === 40) {
          // 部队攻击
          if (this.army.length > 0) {
            const unit = this.army[Math.floor(Math.random() * this.army.length)];
            const unitDmg = Math.floor(unit.attack * unit.count * 0.4);
            this.battleLog.push({ text: `${unit.icon} ${unit.name} x${unit.count} 攻击，造成 ${unitDmg} 伤害！`, type: 'our' });
          } else {
            this.battleLog.push({ text: `部队全军覆没...`, type: 'warning' });
          }
        }
        if (progress === 60) {
          // 敌方反击 - 英雄受伤
          const enemyAtk = this.battleEnemies.reduce((s, e) => s + e.attack * (e.count || 1), 0);
          const heroDmg = Math.max(1, Math.floor(enemyAtk * 0.3) - this.totalDefense);
          this.hero.hp = Math.max(0, this.hero.hp - heroDmg);
          this.battleLog.push({ text: `敌方攻击 🧙 ${this.hero.name}，受到 ${heroDmg} 伤害！`, type: 'enemy' });
        }
        if (progress === 80) {
          // 敌方攻击部队
          if (this.army.length > 0) {
            const unit = this.army[Math.floor(Math.random() * this.army.length)];
            const loss = Math.min(unit.count, Math.floor(Math.random() * 3) + 1);
            unit.count -= loss;
            this.battleLog.push({ text: `敌方攻击 ${unit.icon} ${unit.name}，损失 x${loss}！`, type: 'enemy' });
            if (unit.count <= 0) {
              this.army = this.army.filter(u => u.count > 0);
              this.battleLog.push({ text: `${unit.icon} ${unit.name} 全军覆没！`, type: 'warning' });
            }
          }
        }
        
        if (progress >= 100) {
          clearInterval(interval);
          this.resolveBattle(enemy);
        }
      }, 200);
    },
    
    resolveBattle(enemy) {
      const ourPower = this.army.reduce((s, u) => s + u.attack * u.count, 0) + this.totalAttack;
      let theirPower = this.battleEnemies.reduce((s, e) => s + e.attack * e.count, 0);

      // 暴击判定
      let critMultiplier = 1;
      if (Math.random() * 100 < (this.hero.crit || 0)) {
        critMultiplier = 2;
        this.addMessage('💥 暴击！');
      }

      if (ourPower * critMultiplier > theirPower) {
        const expGain = enemy.exp * critMultiplier;
        this.addMessage(`胜利！+${Math.floor(expGain)} 经验`);
        this.hero.exp += Math.floor(expGain);
        this.resources.gold += enemy.gold || 20;

        // Boss战额外奖励
        if (enemy.isBoss) {
          this.addMessage(`🎉 击败Boss！获得丰厚奖励！`);
          this.resources.gold += 200;
          this.totalGoldEarned += 200;
          this.checkAchievement('killFirstBoss');

          // Boss必定掉落装备
          if (this.inventory.length < 12) {
            const drops = [
              { name: '传奇武器', icon: '⚔️', type: '攻击', bonus: 15, price: 500, slot: 'weapon', isEquipment: true },
              { name: '传奇护甲', icon: '🛡️', type: '防御', bonus: 20, price: 500, slot: 'armor', isEquipment: true },
              { name: '龙之戒', icon: '💍', type: '暴击', bonus: 15, price: 400, slot: 'accessory', isEquipment: true },
            ];
            const drop = drops[Math.floor(Math.random() * drops.length)];
            this.inventory.push(drop);
            this.addMessage(`掉落: ${drop.icon} ${drop.name}`);
          }
        } else {
          // 普通敌人
          this.checkAchievement('killFirstEnemy');
          
          // 5%几率掉落装备
          if (Math.random() < 0.05 && this.inventory.length < 12) {
            const drops = [
              { name: '破损的剑', icon: '🗡️', type: '攻击', bonus: 3, price: 30, slot: 'weapon', isEquipment: true },
              { name: '旧护甲', icon: '🛡️', type: '防御', bonus: 3, price: 30, slot: 'armor', isEquipment: true },
            ];
            const drop = drops[Math.floor(Math.random() * drops.length)];
            this.inventory.push(drop);
            this.addMessage(`掉落: ${drop.icon} ${drop.name}`);
          }
        }
        
        this.totalGoldEarned += enemy.gold || 20;
        if (this.totalGoldEarned >= 1000) this.checkAchievement('gold1000');
        if (this.army.length >= 5) this.checkAchievement('army5');

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
      this.skills.forEach(s => s.currentCd = 0); // 重置技能冷却
    },
    
    levelUp() {
      while (this.hero.exp >= this.hero.expToLevel) {
        this.hero.exp -= this.hero.expToLevel;
        this.hero.level++;
        this.hero.maxHp += 10;
        this.hero.hp = this.hero.maxHp;
        this.hero.attack += 2;
        this.hero.defense += 1;
        this.hero.expToLevel = Math.floor(this.hero.expToLevel * 1.5);
        this.skillPoints++;
      }
      this.addMessage(`🎉 升级！Lv.${this.hero.level} 技能点+1`);
      if (this.hero.level >= 10) this.checkAchievement('level10');
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

    // 商店系统
    buyUnit(unit) {
      if (this.resources.gold < unit.price) {
        this.addMessage('金币不足！');
        return;
      }
      if (this.army.length >= 7) {
        this.addMessage('部队已满！');
        return;
      }

      this.resources.gold -= unit.price;
      const existing = this.army.find(u => u.name === unit.name);
      if (existing) {
        existing.count += unit.count;
      } else {
        this.army.push({
          id: Date.now(),
          name: unit.name,
          icon: unit.icon,
          count: unit.count,
          attack: unit.attack,
          defense: unit.defense
        });
      }
      this.addMessage(`招募 ${unit.name} x${unit.count}`);
    },

    buyEquipment(eq) {
      if (this.resources.gold < eq.price) {
        this.addMessage('金币不足！');
        return;
      }

      this.resources.gold -= eq.price;
      this.inventory.push({ ...eq, isEquipment: true });
      this.addMessage(`购买 ${eq.name}`);
    },

    buyItem(item) {
      if (this.resources.gold < item.price) {
        this.addMessage('金币不足！');
        return;
      }
      if (this.inventory.length >= 12) {
        this.addMessage('背包已满！');
        return;
      }

      this.resources.gold -= item.price;
      this.inventory.push({ ...item, isItem: true });
      this.addMessage(`购买 ${item.name}`);
    },

    useItem(item, index) {
      if (item.isEquipment) {
        // 装备物品
        const oldEquip = this.hero.equipment[item.slot];
        this.hero.equipment[item.slot] = item;
        this.inventory.splice(index, 1);
        if (oldEquip) {
          this.inventory.push(oldEquip);
        }
        this.addMessage(`装备 ${item.name}`);
      } else if (item.isItem) {
        // 使用道具
        if (item.effect.heal) {
          this.hero.hp = Math.min(this.hero.maxHp, this.hero.hp + item.effect.heal);
          this.addMessage(`恢复 ${item.effect.heal} HP`);
        }
        if (item.effect.exp) {
          this.hero.exp += item.effect.exp;
          this.addMessage(`+${item.effect.exp} 经验`);
          if (this.hero.exp >= this.hero.expToLevel) {
            this.levelUp();
          }
        }
        if (item.effect.summon) {
          const existing = this.army.find(u => u.name === '步兵');
          if (existing) {
            existing.count += 10;
          } else if (this.army.length < 7) {
            this.army.push({
              id: Date.now(),
              name: '步兵',
              icon: '🗡️',
              count: 10,
              attack: 5,
              defense: 3
            });
          }
          this.addMessage('召唤步兵 x10');
        }
        this.inventory.splice(index, 1);
      }
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
    
    showUnitInfo(unit) {
      this.selectedUnit = unit;
    },
    
    // 技能系统
    unlockSkill(skill) {
      if (this.skillPoints > 0 && !skill.unlocked) {
        skill.unlocked = true;
        this.skillPoints--;
        this.addMessage(`解锁技能: ${skill.icon} ${skill.name}`);
      }
    },
    
    useSkill(skill) {
      if (!this.inBattle || skill.currentCd > 0) return;
      
      const enemy = this.battleEnemies[0];
      if (!enemy) return;
      
      switch(skill.id) {
        case 1: // 火焰冲击
          const fireDmg = Math.floor(this.totalAttack * 2 * this.battleBuff);
          this.battleLog.push({ text: `🔥 火焰冲击！造成 ${fireDmg} 火焰伤害！`, type: 'our' });
          break;
        case 2: // 雷霆一击
          const thunderDmg = Math.floor(this.totalAttack * 1.5 * this.battleBuff);
          this.battleLog.push({ text: `⚡ 雷霆一击！造成 ${thunderDmg} 伤害，敌人眩晕！`, type: 'our' });
          break;
        case 3: // 治愈之光
          const heal = Math.floor(this.hero.maxHp * 0.3);
          this.hero.hp = Math.min(this.hero.maxHp, this.hero.hp + heal);
          this.battleLog.push({ text: `💚 治愈之光！恢复 ${heal} HP！`, type: 'our' });
          break;
        case 4: // 战吼
          this.battleBuff = 1.5;
          this.battleLog.push({ text: `📢 战吼！攻击力提升50%！`, type: 'our' });
          break;
      }
      
      skill.currentCd = skill.cooldown;
    },
    
    // 部队增援
    reinforceUnit(unit) {
      if (this.resources.gold < 50) {
        this.addMessage('金币不足！');
        return;
      }
      this.resources.gold -= 50;
      unit.count += Math.ceil(unit.count * 0.2); // 增加20%数量
      this.addMessage(`${unit.icon} ${unit.name} 增援成功！`);
    },
    
    // 装备强化
    getEnhanceCost(item) {
      const level = item.enhanceLevel || 0;
      return (level + 1) * 100;
    },
    
    enhanceItem(item, index) {
      const cost = this.getEnhanceCost(item);
      if (this.resources.gold < cost) {
        this.addMessage('金币不足！');
        return;
      }
      
      this.resources.gold -= cost;
      item.enhanceLevel = (item.enhanceLevel || 0) + 1;
      item.bonus += Math.ceil(item.bonus * 0.1); // 提升10%
      this.addMessage(`${item.icon} ${item.name} 强化至 Lv.${item.enhanceLevel}！`);
    },
    
    // 存档系统
    saveGame() {
      const saveData = {
        hero: this.hero,
        army: this.army,
        resources: this.resources,
        inventory: this.inventory,
        skills: this.skills,
        skillPoints: this.skillPoints,
        achievements: this.achievements,
        totalGoldEarned: this.totalGoldEarned,
        worldMap: this.worldMap,
        currentArea: this.currentArea,
        currentRegion: this.currentRegion,
        timestamp: Date.now()
      };
      
      localStorage.setItem('heroGame_save', JSON.stringify(saveData));
      this.saveToastMsg = '💾 游戏已保存！';
      this.showSaveToast = true;
      setTimeout(() => this.showSaveToast = false, 2000);
    },
    
    loadGame() {
      const saved = localStorage.getItem('heroGame_save');
      if (!saved) return false;
      
      try {
        const data = JSON.parse(saved);
        this.hero = data.hero || this.hero;
        this.army = data.army || this.army;
        this.resources = data.resources || this.resources;
        this.inventory = data.inventory || [];
        this.skills = data.skills || this.skills;
        this.skillPoints = data.skillPoints || 0;
        this.achievements = data.achievements || this.achievements;
        this.totalGoldEarned = data.totalGoldEarned || 0;
        this.worldMap = data.worldMap || this.worldMap;
        this.currentArea = data.currentArea || this.worldMap[0];
        this.currentRegion = data.currentRegion || 0;
        return true;
      } catch(e) {
        return false;
      }
    },
    
    checkAchievement(condition) {
      const ach = this.achievements.find(a => a.condition === condition && !a.unlocked);
      if (!ach) return;
      
      ach.unlocked = true;
      this.addMessage(`🏆 成就解锁: ${ach.icon} ${ach.name}！`);
      
      if (ach.rewardType === 'skillPoint') {
        this.skillPoints += ach.reward || 1;
        this.addMessage(`获得技能点 +${ach.reward || 1}`);
      } else if (ach.rewardType === 'legendary') {
        const legendary = { name: '传奇之刃', icon: '⚔️', type: '攻击', bonus: 30, price: 1000, slot: 'weapon', isEquipment: true };
        this.inventory.push(legendary);
        this.addMessage(`获得传奇装备: ${legendary.icon} ${legendary.name}`);
      } else {
        this.resources.gold += ach.reward;
        this.addMessage(`获得金币 +${ach.reward}`);
      }
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
  width: 100px; 
  height: 12px; 
  background: #333; 
  border-radius: 6px; 
  overflow: hidden; 
  position: relative;
}
.hp-text {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  font-size: 9px;
  color: white;
  text-shadow: 0 0 2px black;
}
.exp-bar { 
  width: 80px; 
  height: 4px; 
  background: #333; 
  border-radius: 2px; 
  overflow: hidden; 
}
.exp-fill { 
  height: 100%; 
  background: linear-gradient(90deg, #4CAF50, #8BC34A); 
  transition: width 0.3s; 
}
.hero-stats-mini { font-size: 11px; color: #aaa; }
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
.bottom-bar button.active {
  background: linear-gradient(135deg, #4CAF50, #8BC34A);
  box-shadow: 0 0 10px rgba(76, 175, 80, 0.5);
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
  padding: 10px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
  cursor: pointer;
}
.army-unit:hover { background: rgba(102,126,234,0.3); }
.army-unit .unit-icon { font-size: 24px; }
.army-unit .unit-details { flex: 1; display: flex; flex-direction: column; }
.army-unit .unit-name { font-weight: bold; }
.army-unit .unit-stats { font-size: 11px; color: #888; }
.army-unit .unit-count { font-size: 16px; color: #ffd700; }
.empty-army { text-align: center; color: #666; padding: 20px; }

.unit-info-panel { min-width: 280px; }
.unit-info-content { display: flex; flex-direction: column; gap: 10px; }
.info-row { display: flex; justify-content: space-between; padding: 8px; background: rgba(255,255,255,0.05); border-radius: 5px; }
.info-label { color: #888; }
.info-value { font-weight: bold; }
.info-value.highlight { color: #ffd700; }
.unit-actions { margin-top: 15px; }
.unit-actions button {
  width: 100%;
  padding: 10px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  border: none;
  color: white;
  border-radius: 8px;
  cursor: pointer;
}
.unit-actions button:disabled { opacity: 0.5; cursor: not-allowed; }

/* 强化面板样式 */
.enhance-panel { min-width: 300px; max-height: 80vh; overflow-y: auto; }
.enhance-list { display: flex; flex-direction: column; gap: 8px; }
.enhance-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
}
.enhance-item button {
  padding: 5px 10px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  border: none;
  color: white;
  border-radius: 5px;
  cursor: pointer;
}
.enhance-item button:disabled { opacity: 0.5; cursor: not-allowed; }

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
/* 商店样式 */
.shop-panel { min-width: 320px; max-width: 90vw; max-height: 80vh; overflow-y: auto; }
.shop-tabs {
  display: flex;
  gap: 5px;
  margin-bottom: 15px;
}
.shop-tabs button {
  flex: 1;
  padding: 8px;
  background: rgba(255,255,255,0.1);
  border: none;
  color: white;
  border-radius: 5px;
  cursor: pointer;
}
.shop-tabs button.active {
  background: linear-gradient(135deg, #667eea, #764ba2);
}
.shop-items { display: flex; flex-direction: column; gap: 8px; }
.shop-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}
.shop-item:hover { background: rgba(102,126,234,0.3); transform: scale(1.02); }
.item-icon { font-size: 24px; width: 35px; text-align: center; }
.item-info { flex: 1; display: flex; flex-direction: column; }
.item-name { font-weight: bold; }
.item-stats { font-size: 11px; opacity: 0.7; }
.item-price { color: #ffd700; font-weight: bold; }

/* 技能栏样式 */
.skill-bar {
  position: absolute;
  bottom: 70px;
  right: 10px;
  display: flex;
  gap: 8px;
}
.skill-slot {
  width: 50px;
  height: 50px;
  background: rgba(30,30,50,0.9);
  border: 2px solid #667eea;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  position: relative;
}
.skill-slot:hover { background: rgba(102,126,234,0.3); }
.skill-slot.disabled { opacity: 0.5; cursor: not-allowed; }
.skill-icon { font-size: 24px; }
.skill-cd {
  position: absolute;
  top: 2px;
  right: 4px;
  font-size: 12px;
  color: #ff4444;
  font-weight: bold;
}

/* 技能面板样式 */
.skill-list { display: flex; flex-direction: column; gap: 8px; }
.skill-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
}
.skill-item .skill-icon { font-size: 28px; }
.skill-info { flex: 1; display: flex; flex-direction: column; }
.skill-name { font-weight: bold; }
.skill-desc { font-size: 11px; color: #888; }
.skill-cd-info { font-size: 10px; color: #667eea; }
.unlock-btn {
  padding: 5px 10px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  border: none;
  color: white;
  border-radius: 5px;
  cursor: pointer;
}
.unlocked-badge { color: #7fff7f; font-size: 18px; }

/* 成就面板样式 */
.achievement-list { display: flex; flex-direction: column; gap: 8px; }
.achievement-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: rgba(255,255,255,0.05);
  border-radius: 8px;
  border: 1px solid rgba(255,255,255,0.1);
}
.achievement-item.unlocked { 
  background: rgba(102,126,234,0.2); 
  border-color: #ffd700;
}
.ach-icon { font-size: 28px; }
.ach-info { flex: 1; display: flex; flex-direction: column; }
.ach-name { font-weight: bold; }
.ach-desc { font-size: 11px; color: #888; }
.ach-reward { color: #ffd700; font-size: 12px; }

/* 装备面板样式 */
.equipment-panel { min-width: 300px; max-height: 80vh; overflow-y: auto; }
.section-title { 
  font-size: 12px; 
  color: #888; 
  margin-bottom: 8px; 
  text-transform: uppercase;
}
.equipped-section { margin-bottom: 15px; }
.equipped-slots { display: flex; flex-direction: column; gap: 8px; }
.equip-slot {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
}
.equip-slot.empty { opacity: 0.5; }
.slot-label { 
  width: 40px; 
  font-size: 12px; 
  color: #888; 
}
.empty-slot { color: #666; }

.inventory-section { margin-bottom: 15px; }
.inventory-grid { 
  display: grid; 
  grid-template-columns: repeat(2, 1fr); 
  gap: 8px; 
  max-height: 150px; 
  overflow-y: auto; 
}
.inventory-item {
  padding: 8px;
  background: rgba(255,255,255,0.1);
  border-radius: 5px;
  font-size: 12px;
  cursor: pointer;
  text-align: center;
}
.inventory-item:hover { background: rgba(102,126,234,0.3); }
.empty-inventory { 
  grid-column: span 2; 
  text-align: center; 
  color: #666; 
  padding: 20px; 
}

.hero-stats { border-top: 1px solid #667eea; padding-top: 10px; }
.stats-grid { 
  display: grid; 
  grid-template-columns: repeat(2, 1fr); 
  gap: 8px; 
  font-size: 13px; 
}

.area-icon { font-size: 28px; display: block; }
.area-name { font-size: 12px; }

.battle-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0,0,0,0.95);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  z-index: 100;
}

.battle-box {
  background: rgba(30,30,50,0.95);
  border: 2px solid #667eea;
  border-radius: 15px;
  padding: 20px;
  width: 90%;
  max-width: 400px;
}

.battle-title { 
  font-size: 22px; 
  text-align: center;
  margin-bottom: 15px;
  color: #ffd700;
}

.battle-sides {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 10px;
  margin-bottom: 15px;
}

.battle-side {
  flex: 1;
  padding: 10px;
  background: rgba(255,255,255,0.05);
  border-radius: 10px;
}

.side-label {
  font-size: 12px;
  color: #888;
  margin-bottom: 8px;
}

.hero-card, .unit-card {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
  margin-bottom: 5px;
}

.unit-card.enemy {
  background: rgba(255,0,0,0.2);
  border: 1px solid rgba(255,100,100,0.3);
}

.card-icon { font-size: 24px; }
.card-name { font-size: 12px; flex: 1; }
.card-count { font-size: 11px; color: #ffd700; }
.card-info { flex: 1; }
.hp-bar-mini {
  width: 60px;
  height: 6px;
  background: #333;
  border-radius: 3px;
  overflow: hidden;
}
.hp-bar-mini .hp-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff4444, #ff6666);
  transition: width 0.3s;
}

.boss-badge {
  background: #ff4444;
  color: white;
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 3px;
}

.battle-vs {
  font-size: 20px;
  color: #ff4444;
  padding-top: 40px;
}

.battle-log {
  max-height: 100px;
  overflow-y: auto;
  background: rgba(0,0,0,0.3);
  border-radius: 8px;
  padding: 10px;
  margin-bottom: 15px;
}

.log-item {
  font-size: 12px;
  padding: 3px 0;
  border-bottom: 1px solid rgba(255,255,255,0.1);
}
.log-item.our { color: #7fff7f; }
.log-item.enemy { color: #ff7f7f; }
.log-item.info { color: #aaddff; }
.log-item.warning { color: #ffaa00; }

.battle-progress {
  margin-top: 10px;
}

.progress-bar {
  height: 8px;
  background: #333;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #667eea, #764ba2);
  transition: width 0.15s;
}

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

.save-toast {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0,0,0,0.9);
  padding: 15px 30px;
  border-radius: 10px;
  color: #7fff7f;
  font-size: 18px;
  font-weight: bold;
  z-index: 1000;
}
</style>
