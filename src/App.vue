<template>
  <div class="game-container">
    <h1>🎮 Vue 小游戏</h1>
    <p>得分: {{ score }}</p>
    
    <div class="game-area" @click="handleClick">
      <div 
        v-for="target in targets" 
        :key="target.id"
        class="target"
        :style="{ left: target.x + 'px', top: target.y + 'px' }"
        @click.stop="hitTarget(target)"
      >
        {{ target.emoji }}
      </div>
    </div>
    
    <button @click="startGame" v-if="!gameRunning">开始游戏</button>
    <button @click="stopGame" v-else>停止游戏</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      score: 0,
      gameRunning: false,
      targets: [],
      targetId: 0,
      gameInterval: null,
      emojis: ['🍎', '🍊', '🍋', '🍇', '🍓', '🍒', '🥝', '🍑']
    }
  },
  methods: {
    startGame() {
      this.score = 0
      this.targets = []
      this.gameRunning = true
      this.spawnTarget()
      this.gameInterval = setInterval(() => {
        this.spawnTarget()
      }, 1000)
    },
    stopGame() {
      this.gameRunning = false
      clearInterval(this.gameInterval)
      this.targets = []
    },
    spawnTarget() {
      if (this.targets.length < 5) {
        const target = {
          id: this.targetId++,
          x: Math.random() * 300,
          y: Math.random() * 300,
          emoji: this.emojis[Math.floor(Math.random() * this.emojis.length)]
        }
        this.targets.push(target)
        
        // 3秒后自动消失
        setTimeout(() => {
          this.targets = this.targets.filter(t => t.id !== target.id)
        }, 3000)
      }
    },
    hitTarget(target) {
      this.score += 10
      this.targets = this.targets.filter(t => t.id !== target.id)
    },
    handleClick() {
      // 点击空白区域不扣分，但可以扩展
    }
  }
}
</script>

<style>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  color: #42b883;
}

.game-area {
  width: 350px;
  height: 350px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 10px;
  position: relative;
  margin: 20px 0;
  cursor: crosshair;
}

.target {
  position: absolute;
  font-size: 30px;
  cursor: pointer;
  transition: transform 0.1s;
  user-select: none;
}

.target:hover {
  transform: scale(1.2);
}

button {
  padding: 10px 30px;
  font-size: 18px;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

button:hover {
  background: #3aa876;
}
</style>
