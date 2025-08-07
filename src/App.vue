<template>
  <div id="app">
    <h1>贪吃蛇游戏</h1>
    <div class="game-info">
      <div class="score">得分: {{ score }}</div>
      <div class="controls">
        <button @click="startGame" :disabled="isPlaying">开始</button>
        <button @click="pauseGame" :disabled="!isPlaying">暂停</button>
        <button @click="resetGame">重置</button>
      </div>
    </div>
    <div class="game-container">
      <canvas 
        ref="gameCanvas" 
        :width="canvasWidth" 
        :height="canvasHeight"
        @keydown="handleKeyPress"
        tabindex="0"
      ></canvas>
    </div>
    <div class="instructions">
      <p>使用方向键控制蛇的移动</p>
      <p>吃到食物可以增加分数和长度</p>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'

export default {
  name: 'App',
  setup() {
    const gameCanvas = ref(null)
    const canvasWidth = 600
    const canvasHeight = 400
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
      const maxX = Math.floor(canvasWidth / gridSize)
      const maxY = Math.floor(canvasHeight / gridSize)
      
      do {
        food = {
          x: Math.floor(Math.random() * maxX),
          y: Math.floor(Math.random() * maxY)
        }
      } while (snake.some(segment => segment.x === food.x && segment.y === food.y))
    }
    
    const drawGame = () => {
      ctx.fillStyle = '#000'
      ctx.fillRect(0, 0, canvasWidth, canvasHeight)
      
      ctx.fillStyle = '#0f0'
      snake.forEach(segment => {
        ctx.fillRect(
          segment.x * gridSize,
          segment.y * gridSize,
          gridSize - 2,
          gridSize - 2
        )
      })
      
      ctx.fillStyle = '#f00'
      ctx.fillRect(
        food.x * gridSize,
        food.y * gridSize,
        gridSize - 2,
        gridSize - 2
      )
    }
    
    const moveSnake = () => {
      direction = { ...nextDirection }
      
      const head = { ...snake[0] }
      head.x += direction.x
      head.y += direction.y
      
      if (head.x < 0 || head.x >= canvasWidth / gridSize ||
          head.y < 0 || head.y >= canvasHeight / gridSize ||
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
      initGame()
      drawGame()
    }
    
    const gameOver = () => {
      pauseGame()
      alert(`游戏结束！最终得分: ${score.value}`)
    }
    
    const handleKeyPress = (event) => {
      if (!isPlaying.value) return
      
      switch (event.key) {
        case 'ArrowUp':
          if (direction.y !== 1) nextDirection = { x: 0, y: -1 }
          break
        case 'ArrowDown':
          if (direction.y !== -1) nextDirection = { x: 0, y: 1 }
          break
        case 'ArrowLeft':
          if (direction.x !== 1) nextDirection = { x: -1, y: 0 }
          break
        case 'ArrowRight':
          if (direction.x !== -1) nextDirection = { x: 1, y: 0 }
          break
      }
      event.preventDefault()
    }
    
    onMounted(() => {
      ctx = gameCanvas.value.getContext('2d')
      initGame()
      drawGame()
      
      document.addEventListener('keydown', handleKeyPress)
    })
    
    onUnmounted(() => {
      if (gameInterval.value) {
        clearInterval(gameInterval.value)
      }
      document.removeEventListener('keydown', handleKeyPress)
    })
    
    return {
      gameCanvas,
      canvasWidth,
      canvasHeight,
      score,
      isPlaying,
      startGame,
      pauseGame,
      resetGame,
      handleKeyPress
    }
  }
}
</script>

<style>
#app {
  text-align: center;
  font-family: Arial, sans-serif;
  padding: 20px;
}

h1 {
  color: #333;
  margin-bottom: 20px;
}

.game-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  max-width: 600px;
  margin: 0 auto 20px;
}

.score {
  font-size: 18px;
  font-weight: bold;
  color: #333;
}

.controls button {
  margin: 0 5px;
  padding: 8px 16px;
  font-size: 14px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.controls button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.controls button:hover:not(:disabled) {
  background-color: #45a049;
}

.game-container {
  display: flex;
  justify-content: center;
  margin: 20px 0;
}

canvas {
  border: 2px solid #333;
  background-color: #000;
}

.instructions {
  max-width: 600px;
  margin: 20px auto;
  color: #666;
}

.instructions p {
  margin: 5px 0;
}
</style>