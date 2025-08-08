<template>
  <div id="app">
    <div class="header">
      <h1>🐍 贪吃蛇游戏</h1>
      <div class="score-display">
        <span class="score-label">得分</span>
        <span class="score-value">{{ score }}</span>
      </div>
    </div>
    
    <div class="game-container">
      <canvas 
        ref="gameCanvas" 
        :width="canvasWidth" 
        :height="canvasHeight"
        @keydown="handleKeyPress"
        tabindex="0"
        class="game-canvas"
      ></canvas>
      
      <div class="game-over-overlay" v-if="showGameOver">
        <div class="game-over-content">
          <h2>🎮 游戏结束</h2>
          <p>最终得分: <strong>{{ score }}</strong></p>
          <button @click="resetGame" class="restart-btn">重新开始</button>
        </div>
      </div>
    </div>
    
    <div class="controls">
      <button @click="startGame" :disabled="isPlaying" class="btn btn-start">
        <span v-if="!isPlaying">▶️ 开始</span>
        <span v-else>🎮 游戏中</span>
      </button>
      <button @click="pauseGame" :disabled="!isPlaying" class="btn btn-pause">
        ⏸️ 暂停
      </button>
      <button @click="resetGame" class="btn btn-reset">
        🔄 重置
      </button>
    </div>
    
    <!-- 移动端虚拟方向键 -->
    <div class="mobile-controls" v-if="isMobile">
      <div class="dpad">
        <button @touchstart="handleMobileDirection('up')" @click="handleMobileDirection('up')" class="dpad-btn dpad-up">↑</button>
        <div class="dpad-middle">
          <button @touchstart="handleMobileDirection('left')" @click="handleMobileDirection('left')" class="dpad-btn dpad-left">←</button>
          <div class="dpad-center"></div>
          <button @touchstart="handleMobileDirection('right')" @click="handleMobileDirection('right')" class="dpad-btn dpad-right">→</button>
        </div>
        <button @touchstart="handleMobileDirection('down')" @click="handleMobileDirection('down')" class="dpad-btn dpad-down">↓</button>
      </div>
    </div>
    
    <div class="instructions">
      <p v-if="!isMobile">⌨️ 使用方向键或WASD控制蛇的移动</p>
      <p v-else>👆 点击虚拟方向键控制蛇的移动</p>
      <p>🍎 吃到食物可以增加分数和长度</p>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'

export default {
  name: 'App',
  setup() {
    const gameCanvas = ref(null)
    const isMobile = ref(false)
    const showGameOver = ref(false)
    const canvasWidth = ref(600)
    const canvasHeight = ref(400)
    const gridSize = 20
    
    const score = ref(0)
    const isPlaying = ref(false)
    const gameInterval = ref(null)
    
    let ctx = null
    let snake = []
    let food = {}
    let direction = { x: 1, y: 0 }
    let nextDirection = { x: 1, y: 0 }
    
    const initGame = () => {
      snake = [
        { x: 3, y: 10 },
        { x: 2, y: 10 },
        { x: 1, y: 10 }
      ]
      direction = { x: 1, y: 0 }
      nextDirection = { x: 1, y: 0 }
      score.value = 0
      generateFood()
    }
    
    const generateFood = () => {
      const maxX = Math.floor(canvasWidth.value / gridSize)
      const maxY = Math.floor(canvasHeight.value / gridSize)
      
      do {
        food = {
          x: Math.floor(Math.random() * maxX),
          y: Math.floor(Math.random() * maxY)
        }
      } while (snake.some(segment => segment.x === food.x && segment.y === food.y))
    }
    
    const drawGame = () => {
      // 清空画布
      ctx.fillStyle = '#000'
      ctx.fillRect(0, 0, canvasWidth.value, canvasHeight.value)
      
      // 绘制蛇身
      snake.forEach((segment, index) => {
        const x = segment.x * gridSize
        const y = segment.y * gridSize
        const size = gridSize - 2
        
        if (index === 0) {
          // 蛇头 - 更亮的绿色和圆角
          const gradient = ctx.createLinearGradient(x, y, x + size, y + size)
          gradient.addColorStop(0, '#7fff00')
          gradient.addColorStop(1, '#32cd32')
          ctx.fillStyle = gradient
          drawRoundRect(x + 1, y + 1, size, size, 4)
          ctx.fill()
          
          // 蛇眼
          ctx.fillStyle = '#000'
          ctx.beginPath()
          ctx.arc(x + size * 0.3, y + size * 0.3, 2, 0, Math.PI * 2)
          ctx.arc(x + size * 0.7, y + size * 0.3, 2, 0, Math.PI * 2)
          ctx.fill()
        } else {
          // 蛇身 - 渐变绿色
          const gradient = ctx.createLinearGradient(x, y, x + size, y + size)
          gradient.addColorStop(0, '#32cd32')
          gradient.addColorStop(1, '#228b22')
          ctx.fillStyle = gradient
          drawRoundRect(x + 1, y + 1, size, size, 3)
          ctx.fill()
          
          // 蛇身纹理
          ctx.fillStyle = 'rgba(0, 0, 0, 0.1)'
          drawRoundRect(x + 3, y + 3, size - 4, size - 4, 2)
          ctx.fill()
        }
      })
      
      // 绘制食物 - 红色圆形苹果样式
      const foodX = food.x * gridSize
      const foodY = food.y * gridSize
      const foodSize = gridSize - 2
      
      // 苹果主体
      const foodGradient = ctx.createRadialGradient(
        foodX + foodSize * 0.3, foodY + foodSize * 0.3, 0,
        foodX + foodSize * 0.5, foodY + foodSize * 0.5, foodSize * 0.7
      )
      foodGradient.addColorStop(0, '#ff6b6b')
      foodGradient.addColorStop(1, '#dc143c')
      ctx.fillStyle = foodGradient
      ctx.beginPath()
      ctx.arc(foodX + foodSize / 2, foodY + foodSize / 2, foodSize / 2 - 1, 0, Math.PI * 2)
      ctx.fill()
      
      // 苹果高光
      ctx.fillStyle = 'rgba(255, 255, 255, 0.6)'
      ctx.beginPath()
      ctx.arc(foodX + foodSize * 0.35, foodY + foodSize * 0.35, 3, 0, Math.PI * 2)
      ctx.fill()
    }
    
    const moveSnake = () => {
      direction = { ...nextDirection }
      
      const head = { ...snake[0] }
      head.x += direction.x
      head.y += direction.y
      
      if (head.x < 0 || head.x >= canvasWidth.value / gridSize ||
          head.y < 0 || head.y >= canvasHeight.value / gridSize ||
          snake.some(segment => segment.x === head.x && segment.y === head.y)) {
        gameOver()
        return
      }
      
      snake.unshift(head)
      
      if (head.x === food.x && head.y === food.y) {
        score.value += 10
        generateFood()
      } else {
        snake.pop()
      }
    }
    
    const gameLoop = () => {
      moveSnake()
      drawGame()
    }
    
    const startGame = () => {
      if (!isPlaying.value) {
        isPlaying.value = true
        gameInterval.value = setInterval(gameLoop, 150)
      }
    }
    
    const pauseGame = () => {
      isPlaying.value = false
      if (gameInterval.value) {
        clearInterval(gameInterval.value)
        gameInterval.value = null
      }
    }
    
    const resetGame = () => {
      pauseGame()
      showGameOver.value = false
      initGame()
      drawGame()
    }
    
    const gameOver = () => {
      pauseGame()
      showGameOver.value = true
    }
    
    const handleKeyPress = (event) => {
      if (!isPlaying.value) return
      
      switch (event.key) {
        case 'ArrowUp':
        case 'w':
        case 'W':
          if (direction.y !== 1) nextDirection = { x: 0, y: -1 }
          break
        case 'ArrowDown':
        case 's':
        case 'S':
          if (direction.y !== -1) nextDirection = { x: 0, y: 1 }
          break
        case 'ArrowLeft':
        case 'a':
        case 'A':
          if (direction.x !== 1) nextDirection = { x: -1, y: 0 }
          break
        case 'ArrowRight':
        case 'd':
        case 'D':
          if (direction.x !== -1) nextDirection = { x: 1, y: 0 }
          break
      }
      event.preventDefault()
    }
    
    const handleMobileDirection = (dir) => {
      if (!isPlaying.value) return
      
      switch (dir) {
        case 'up':
          if (direction.y !== 1) nextDirection = { x: 0, y: -1 }
          break
        case 'down':
          if (direction.y !== -1) nextDirection = { x: 0, y: 1 }
          break
        case 'left':
          if (direction.x !== 1) nextDirection = { x: -1, y: 0 }
          break
        case 'right':
          if (direction.x !== -1) nextDirection = { x: 1, y: 0 }
          break
      }
    }
    
    // 兼容性的roundRect函数
    const drawRoundRect = (x, y, width, height, radius) => {
      if (ctx.roundRect) {
        ctx.roundRect(x, y, width, height, radius)
      } else {
        ctx.beginPath()
        ctx.moveTo(x + radius, y)
        ctx.lineTo(x + width - radius, y)
        ctx.quadraticCurveTo(x + width, y, x + width, y + radius)
        ctx.lineTo(x + width, y + height - radius)
        ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height)
        ctx.lineTo(x + radius, y + height)
        ctx.quadraticCurveTo(x, y + height, x, y + height - radius)
        ctx.lineTo(x, y + radius)
        ctx.quadraticCurveTo(x, y, x + radius, y)
        ctx.closePath()
      }
    }
    
    const checkMobile = () => {
      isMobile.value = window.innerWidth <= 768 || /Android|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
      
      // 根据屏幕大小调整画布尺寸
      if (isMobile.value) {
        const maxWidth = Math.min(window.innerWidth - 40, 400)
        const maxHeight = Math.min(window.innerHeight * 0.4, 300)
        canvasWidth.value = Math.floor(maxWidth / 20) * 20
        canvasHeight.value = Math.floor(maxHeight / 20) * 20
      } else {
        canvasWidth.value = 600
        canvasHeight.value = 400
      }
    }
    
    onMounted(() => {
      checkMobile()
      ctx = gameCanvas.value.getContext('2d')
      initGame()
      drawGame()
      
      document.addEventListener('keydown', handleKeyPress)
      window.addEventListener('resize', checkMobile)
      
      // 阻止移动端的默认触摸行为
      document.addEventListener('touchstart', (e) => {
        if (e.target.closest('.mobile-controls')) {
          e.preventDefault()
        }
      }, { passive: false })
    })
    
    onUnmounted(() => {
      if (gameInterval.value) {
        clearInterval(gameInterval.value)
      }
      document.removeEventListener('keydown', handleKeyPress)
      window.removeEventListener('resize', checkMobile)
    })
    
    return {
      gameCanvas,
      canvasWidth,
      canvasHeight,
      score,
      isPlaying,
      isMobile,
      showGameOver,
      startGame,
      pauseGame,
      resetGame,
      handleKeyPress,
      handleMobileDirection
    }
  }
}
</script>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding: 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

#app {
  text-align: center;
  padding: 10px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.header {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 20px 30px;
  margin-bottom: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  min-width: 300px;
}

h1 {
  color: #fff;
  margin: 0 0 15px 0;
  font-size: 2.5rem;
  font-weight: 700;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
}

.score-display {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: rgba(255, 255, 255, 0.2);
  padding: 10px 20px;
  border-radius: 15px;
  margin-top: 10px;
}

.score-label {
  color: #fff;
  font-size: 1.1rem;
  font-weight: 500;
}

.score-value {
  color: #fff;
  font-size: 1.5rem;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
}

.game-container {
  position: relative;
  margin: 20px 0;
  display: flex;
  justify-content: center;
  align-items: center;
}

.game-canvas {
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-radius: 15px;
  background-color: #000;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  transition: all 0.3s ease;
}

.game-canvas:focus {
  outline: none;
  border-color: rgba(255, 255, 255, 0.6);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4), 0 0 20px rgba(255, 255, 255, 0.2);
}

.game-over-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 15px;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.game-over-content {
  background: rgba(255, 255, 255, 0.95);
  padding: 30px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.game-over-content h2 {
  margin: 0 0 15px 0;
  color: #333;
  font-size: 1.8rem;
}

.game-over-content p {
  margin: 15px 0;
  color: #666;
  font-size: 1.1rem;
}

.restart-btn {
  background: linear-gradient(45deg, #667eea, #764ba2);
  color: white;
  border: none;
  padding: 12px 30px;
  border-radius: 25px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.restart-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
}

.controls {
  display: flex;
  gap: 15px;
  margin: 20px 0;
  flex-wrap: wrap;
  justify-content: center;
}

.btn {
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.3);
  padding: 12px 20px;
  border-radius: 25px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 100px;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
}

.btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2);
}

.btn:disabled {
  background: rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.5);
  cursor: not-allowed;
  transform: none;
}

.btn-start {
  background: linear-gradient(45deg, #4CAF50, #45a049);
}

.btn-pause {
  background: linear-gradient(45deg, #ff9800, #f57c00);
}

.btn-reset {
  background: linear-gradient(45deg, #f44336, #d32f2f);
}

.mobile-controls {
  margin: 30px 0;
  display: flex;
  justify-content: center;
}

.dpad {
  display: grid;
  grid-template-rows: 1fr 1fr 1fr;
  gap: 5px;
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  padding: 20px;
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.dpad-middle {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 5px;
  align-items: center;
}

.dpad-btn {
  width: 60px;
  height: 60px;
  background: rgba(255, 255, 255, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 15px;
  color: white;
  font-size: 1.5rem;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  user-select: none;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
}

.dpad-btn:active {
  background: rgba(255, 255, 255, 0.4);
  transform: scale(0.95);
}

.dpad-center {
  width: 40px;
  height: 40px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.dpad-up {
  grid-column: 2;
}

.dpad-down {
  grid-column: 2;
}

.instructions {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  border-radius: 15px;
  padding: 20px;
  margin: 20px auto;
  max-width: 500px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: white;
}

.instructions p {
  margin: 8px 0;
  font-size: 1rem;
  line-height: 1.5;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
}

/* 移动端适配 */
@media (max-width: 768px) {
  #app {
    padding: 5px;
  }
  
  h1 {
    font-size: 2rem;
  }
  
  .header {
    padding: 15px 20px;
    margin-bottom: 15px;
    min-width: auto;
    width: 100%;
    max-width: 400px;
  }
  
  .score-display {
    padding: 8px 15px;
  }
  
  .score-label {
    font-size: 1rem;
  }
  
  .score-value {
    font-size: 1.3rem;
  }
  
  .controls {
    gap: 10px;
    margin: 15px 0;
  }
  
  .btn {
    padding: 10px 15px;
    font-size: 0.9rem;
    min-width: 80px;
  }
  
  .game-canvas {
    border-width: 2px;
  }
  
  .dpad {
    padding: 15px;
  }
  
  .dpad-btn {
    width: 50px;
    height: 50px;
    font-size: 1.3rem;
  }
  
  .dpad-center {
    width: 30px;
    height: 30px;
  }
  
  .instructions {
    padding: 15px;
    margin: 15px 10px;
  }
  
  .instructions p {
    font-size: 0.9rem;
  }
}

@media (max-width: 480px) {
  h1 {
    font-size: 1.8rem;
  }
  
  .header {
    padding: 12px 15px;
  }
  
  .controls {
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }
  
  .btn {
    width: 200px;
  }
  
  .dpad {
    padding: 12px;
  }
  
  .dpad-btn {
    width: 45px;
    height: 45px;
    font-size: 1.2rem;
  }
}

/* 防止移动端缩放 */
@media (max-width: 768px) {
  * {
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -khtml-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }
  
  input, textarea {
    -webkit-user-select: text;
    -moz-user-select: text;
    -ms-user-select: text;
    user-select: text;
  }
}
</style>