<template>
  <div class="game-container" :class="currentSkin">
    <!-- 游戏标题和信息 -->
    <div class="game-info">
      <h1>打地鼠小游戏</h1>
      <div class="stats">
        <span>得分: {{ score }}</span>
        <span>倒计时: {{ timeLeft }}s</span>
        <span>最高分: {{ highScore }}</span>
        <span>当前难度: {{ getDifficultyText }}</span>
        <span v-if="isPlaying">连击: {{ combo }} 🔄</span>
      </div>

      <!-- 1. 多档位初始难度选择 -->
      <div class="difficulty-select" v-if="!isPlaying && !isGameOver">
        <span>初始难度:</span>
        <button
          class="diff-btn"
          @click="setInitialDifficulty('easy')"
          :class="{ active: initialDifficulty === 'easy' }"
        >
          简单
        </button>
        <button
          class="diff-btn"
          @click="setInitialDifficulty('medium')"
          :class="{ active: initialDifficulty === 'medium' }"
        >
          中等
        </button>
        <button
          class="diff-btn"
          @click="setInitialDifficulty('hard')"
          :class="{ active: initialDifficulty === 'hard' }"
        >
          困难
        </button>
      </div>

      <!-- 2. 皮肤切换按钮（新增森林/海洋） -->
      <div class="skin-switch">
        <button
          class="skin-btn"
          @click="switchSkin('default')"
          :class="{ active: currentSkin === 'default' }"
        >
          经典
        </button>
        <button
          class="skin-btn"
          @click="switchSkin('cartoon')"
          :class="{ active: currentSkin === 'cartoon' }"
        >
          卡通
        </button>
        <button
          class="skin-btn"
          @click="switchSkin('forest')"
          :class="{ active: currentSkin === 'forest' }"
        >
          森林
        </button>
        <button
          class="skin-btn"
          @click="switchSkin('ocean')"
          :class="{ active: currentSkin === 'ocean' }"
        >
          海洋
        </button>
      </div>

      <!-- 3. 音效开关 + 震动开关 + 排行榜按钮 -->
      <div class="extra-controls">
        <button
          class="control-btn small-btn"
          @click="toggleMute"
          :class="{ muted: isMuted }"
        >
          {{ isMuted ? '🔇 静音' : '🔊 音效' }}
        </button>
        <button
          class="control-btn small-btn"
          @click="toggleVibrate"
          :disabled="!supportsVibrate"
          :class="{ muted: !isVibrate }"
        >
          {{ isVibrate ? '📳 震动' : '📴 震动' }}
        </button>
        <button class="control-btn small-btn" @click="toggleRankList">
          📜 排行榜
        </button>
      </div>

      <button
        class="control-btn main-btn"
        @click="startGame"
        :disabled="isPlaying"
      >
        {{ isPlaying ? '游戏中...' : '开始游戏' }}
      </button>
      <button
        class="control-btn reset-btn"
        @click="resetGame"
        :disabled="!isPlaying && timeLeft === 30"
      >
        重置游戏
      </button>
    </div>

    <!-- 地鼠洞网格（适配图片皮肤） -->
    <div class="mole-grid">
      <div
        v-for="(hole, index) in holes"
        :key="index"
        class="hole"
        @click="whackMole(index)"
        :class="{
          'hole-cartoon': currentSkin === 'cartoon',
          'hole-forest': currentSkin === 'forest',
          'hole-ocean': currentSkin === 'ocean'
        }"
      >
        <div
          class="mole"
          :class="{
            active: hole.isActive,
            'mole-cartoon': currentSkin === 'cartoon',
            'mole-forest': currentSkin === 'forest',
            'mole-ocean': currentSkin === 'ocean'
          }"
        ></div>
      </div>
    </div>

    <!-- 游戏结束提示 -->
    <div class="game-over" v-if="isGameOver">
      <h2>游戏结束！</h2>
      <p>最终得分: {{ score }}</p>
      <p>最高连击: {{ maxCombo }}</p>
      <p v-if="score === highScore && score > 0">🎉 打破最高分！🎉</p>
      <p v-else>最高分: {{ highScore }}</p>
      <button class="control-btn" @click="resetGame">再来一局</button>
      <button class="control-btn" @click="toggleRankList">查看排行榜</button>
    </div>

    <!-- 4. 排行榜弹窗 -->
    <div class="rank-modal" v-if="showRankList">
      <div class="rank-content">
        <h3>🏆 历史排行榜（前10）</h3>
        <div class="rank-list">
          <div class="rank-item" v-for="(item, idx) in rankList" :key="idx">
            <span class="rank-num">{{ idx + 1 }}.</span>
            <span class="rank-score">得分: {{ item.score }}</span>
            <span class="rank-time">{{ item.time }}</span>
          </div>
          <div class="rank-empty" v-if="rankList.length === 0">
            暂无游戏记录～
          </div>
        </div>
        <button class="control-btn" @click="toggleRankList">关闭</button>
      </div>
    </div>

    <!-- 音频资源（隐藏） -->
    <audio ref="hitAudio" src="https://assets.mixkit.co/sfx/preview/mixkit-quick-jump-arcade-game-239.mp3"></audio>
    <audio ref="endAudio" src="https://assets.mixkit.co/sfx/preview/mixkit-game-over-arcade-640.mp3"></audio>
    <audio ref="comboAudio" src="https://assets.mixkit.co/sfx/preview/mixkit-arcade-video-game-bonus-2045.mp3"></audio>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

// 基础游戏状态
const score = ref(0)
const timeLeft = ref(30)
const isPlaying = ref(false)
const isGameOver = ref(false)
let timer = null
let moleTimer = null

// 1. 多档位初始难度配置
const initialDifficulty = ref('medium') // 默认中等
const difficultyConfig = {
  easy: { interval: 1000 }, // 简单：初始1000ms
  medium: { interval: 800 }, // 中等：初始800ms
  hard: { interval: 600 } // 困难：初始600ms
}
const moleInterval = ref(difficultyConfig[initialDifficulty.value].interval)
const minInterval = 200 // 最小间隔（难度上限）
const intervalStep = 50
const difficultyUpdateTime = 5

// 2. 音效开关
const isMuted = ref(false) // 是否静音
const hitAudio = ref(null)
const endAudio = ref(null)
const comboAudio = ref(null) // 连击音效

// 3. 移动端震动反馈
const isVibrate = ref(true) // 是否开启震动
const supportsVibrate = ref(!!navigator.vibrate) // 检测浏览器是否支持震动

// 4. 连击加分
const combo = ref(0) // 当前连击数
const maxCombo = ref(0) // 最高连击数
let comboTimer = null
const comboTimeout = 2000 // 连击超时时间（2秒）

// 5. 最高分 + 排行榜
const highScore = ref(parseInt(localStorage.getItem('moleGameHighScore')) || 0)
const historyScores = ref(JSON.parse(localStorage.getItem('moleGameHistory')) || [])
const showRankList = ref(false) // 是否显示排行榜

// 6. 更多皮肤（经典/卡通/森林/海洋）
const currentSkin = ref('default')
// 皮肤图片（使用免费在线图片，可替换为本地图片）
const skinImages = {
  forest: 'https://picsum.photos/id/118/120/100', // 森林地鼠
  ocean: 'https://picsum.photos/id/152/120/100' // 海洋地鼠
}

// 初始化地鼠洞
const holes = ref(
  Array(9)
    .fill()
    .map(() => ({ isActive: false }))
)

// 随机显示地鼠
const showRandomMole = () => {
  holes.value.forEach(hole => (hole.isActive = false))
  const randomIndex = Math.floor(Math.random() * 9)
  holes.value[randomIndex].isActive = true
}

// 核心：打地鼠逻辑（整合连击/音效/震动）
const whackMole = (index) => {
  if (!isPlaying.value || !holes.value[index].isActive) return

  // 连击逻辑
  clearTimeout(comboTimer)
  combo.value += 1
  maxCombo.value = Math.max(maxCombo.value, combo.value)

  // 连击加分规则：1连击+1，2-4连击+2，5+连击+3
  if (combo.value >= 5) {
    score.value += 3
    !isMuted.value && comboAudio.value?.play().catch(err => console.log('连击音效失败:', err))
  } else if (combo.value >= 2) {
    score.value += 2
  } else {
    score.value += 1
  }

  // 震动反馈（移动端）
  if (isVibrate.value && supportsVibrate.value) {
    navigator.vibrate(50) // 震动50ms
  }

  // 音效播放（判断静音）
  !isMuted.value && hitAudio.value?.play().catch(err => console.log('击中音效失败:', err))

  holes.value[index].isActive = false

  // 连击超时重置
  comboTimer = setTimeout(() => {
    combo.value = 0
  }, comboTimeout)
}

// 设置初始难度
const setInitialDifficulty = (diff) => {
  initialDifficulty.value = diff
  moleInterval.value = difficultyConfig[diff].interval
}

// 更新难度
const updateDifficulty = () => {
  if (moleInterval.value > minInterval) {
    moleInterval.value -= intervalStep
    clearInterval(moleTimer)
    moleTimer = setInterval(showRandomMole, moleInterval.value)
  }
}

// 开始游戏
const startGame = () => {
  // 清除旧定时器，防止叠加
  clearInterval(timer)
  clearInterval(moleTimer)
  clearTimeout(comboTimer)

  // 重置状态
  score.value = 0
  timeLeft.value = 30
  combo.value = 0
  maxCombo.value = 0
  isPlaying.value = true
  isGameOver.value = false
  moleInterval.value = difficultyConfig[initialDifficulty.value].interval

  // 倒计时
  timer = setInterval(() => {
    timeLeft.value -= 1
    // 每5秒升级难度
    if (timeLeft.value % difficultyUpdateTime === 0 && timeLeft.value > 0) {
      updateDifficulty()
    }
    // 游戏结束
    if (timeLeft.value <= 0) {
      endGame()
    }
  }, 1000)

  // 地鼠出现
  moleTimer = setInterval(showRandomMole, moleInterval.value)
}

// 结束游戏（整合排行榜）
const endGame = () => {
  clearInterval(timer)
  clearInterval(moleTimer)
  clearTimeout(comboTimer)
  isPlaying.value = false
  isGameOver.value = true
  holes.value.forEach(hole => (hole.isActive = false))

  // 播放结束音效
  !isMuted.value && endAudio.value?.play().catch(err => console.log('结束音效失败:', err))

  // 更新最高分
  if (score.value > highScore.value) {
    highScore.value = score.value
    localStorage.setItem('moleGameHighScore', highScore.value)
  }

  // 保存历史记录到排行榜
  if (score.value > 0) {
    const now = new Date()
    const timeStr = `${now.getMonth() + 1}/${now.getDate()} ${now.getHours()}:${String(now.getMinutes()).padStart(2, '0')}`
    historyScores.value.push({
      score: score.value,
      time: timeStr
    })
    // 去重 + 排序 + 保留前10
    const uniqueScores = Array.from(new Set(historyScores.value.map(item => JSON.stringify(item))))
      .map(str => JSON.parse(str))
      .sort((a, b) => b.score - a.score)
      .slice(0, 10)
    historyScores.value = uniqueScores
    localStorage.setItem('moleGameHistory', JSON.stringify(historyScores.value))
  }
}

// 重置游戏
const resetGame = () => {
  clearInterval(timer)
  clearInterval(moleTimer)
  clearTimeout(comboTimer)
  score.value = 0
  timeLeft.value = 30
  combo.value = 0
  maxCombo.value = 0
  isPlaying.value = false
  isGameOver.value = false
  moleInterval.value = difficultyConfig[initialDifficulty.value].interval
  holes.value.forEach(hole => (hole.isActive = false))
}

// 切换皮肤
const switchSkin = (skin) => {
  currentSkin.value = skin
}

// 切换音效静音
const toggleMute = () => {
  isMuted.value = !isMuted.value
}

// 切换震动
const toggleVibrate = () => {
  isVibrate.value = !isVibrate.value
}

// 切换排行榜显示
const toggleRankList = () => {
  showRankList.value = !showRankList.value
}

// 难度文本计算属性
const getDifficultyText = computed(() => {
  if (moleInterval.value >= 900) return '简单'
  if (moleInterval.value >= 700) return '中等'
  if (moleInterval.value >= 400) return '困难'
  return '地狱'
})

// 排行榜列表（排序后）
const rankList = computed(() => {
  return historyScores.value.sort((a, b) => b.score - a.score).slice(0, 10)
})

// 预加载资源
onMounted(() => {
  hitAudio.value?.load()
  endAudio.value?.load()
  comboAudio.value?.load()
  // 检测震动权限（部分浏览器需要用户交互后才能使用）
  if (supportsVibrate.value) {
    navigator.vibrate(0) // 初始化震动
  }
})

// 清除定时器
onUnmounted(() => {
  clearInterval(timer)
  clearInterval(moleTimer)
  clearTimeout(comboTimer)
})
</script>

<style scoped>
/* 全局重置 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Arial', sans-serif;
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}

/* 容器适配 */
.game-container {
  max-width: 700px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
  background-color: #f5f5f5;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  width: 95vw;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

/* 皮肤样式：卡通 */
.game-container.cartoon {
  background-color: #fdf2f8;
}
/* 皮肤样式：森林 */
.game-container.forest {
  background-color: #e8f5e9;
  background-image: url('https://picsum.photos/id/122/1000/800');
  background-size: cover;
  background-opacity: 0.8;
}
/* 皮肤样式：海洋 */
.game-container.ocean {
  background-color: #e1f5fe;
  background-image: url('https://picsum.photos/id/152/1000/800');
  background-size: cover;
  background-opacity: 0.8;
}

.game-info h1 {
  color: #333;
  margin-bottom: 15px;
  font-size: clamp(1.8rem, 5vw, 2.5rem);
  text-shadow: 1px 1px 3px rgba(0,0,0,0.1);
}

.stats {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px 20px;
  font-size: clamp(1rem, 3vw, 1.2rem);
  margin-bottom: 10px;
  color: #555;
  font-weight: 500;
}

/* 1. 初始难度选择样式 */
.difficulty-select {
  margin: 10px 0;
}
.diff-btn {
  padding: 6px 12px;
  margin: 0 5px;
  border: 2px solid #42b983;
  border-radius: 4px;
  background: white;
  color: #42b983;
  cursor: pointer;
  transition: all 0.2s;
}
.diff-btn.active {
  background: #42b983;
  color: white;
}

/* 2. 皮肤切换样式 */
.skin-switch {
  margin: 10px 0;
}
.skin-btn {
  padding: 8px 15px;
  margin: 0 5px;
  border: 2px solid #42b983;
  border-radius: 5px;
  background-color: white;
  color: #42b983;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s;
}
.skin-btn.active {
  background-color: #42b983;
  color: white;
}

/* 3. 音效/震动/排行榜按钮 */
.extra-controls {
  margin: 10px 0;
}
.small-btn {
  padding: 6px 12px !important;
  min-height: 40px !important;
  min-width: 80px !important;
  font-size: 0.9rem !important;
  margin: 0 4px !important;
}
.small-btn.muted {
  background-color: #999 !important;
}

/* 主按钮样式 */
.main-btn {
  margin: 5px 0 !important;
}
.control-btn {
  padding: clamp(8px, 3vw, 10px) clamp(15px, 4vw, 20px);
  margin: 0 8px;
  border: none;
  border-radius: 5px;
  background-color: #42b983;
  color: white;
  font-size: clamp(1rem, 3vw, 1.1rem);
  cursor: pointer;
  transition: background-color 0.3s;
  min-height: 44px;
  min-width: 120px;
}
.control-btn:disabled {
  background-color: #999;
  cursor: not-allowed;
}
.control-btn:hover:not(:disabled) {
  background-color: #359469;
}
.reset-btn {
  background-color: #f56c6c;
}
.reset-btn:hover:not(:disabled) {
  background-color: #e45656;
}

/* 地鼠网格 */
.mole-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
  gap: clamp(10px, 3vw, 15px);
  margin: 20px 0;
  flex: 1;
  align-items: center;
}

/* 洞样式：基础 */
.hole {
  width: clamp(100px, 25vw, 150px);
  height: clamp(100px, 25vw, 150px);
  background-color: #8b5a2b;
  border-radius: 50%;
  position: relative;
  cursor: pointer;
  overflow: hidden;
  margin: 0 auto;
  box-shadow: inset 0 0 15px rgba(0,0,0,0.3);
}
/* 洞样式：卡通 */
.hole-cartoon {
  background-color: #e53e3e;
  box-shadow: inset 0 0 10px #c53030;
}
/* 洞样式：森林 */
.hole-forest {
  background-color: #558b2f;
  box-shadow: inset 0 0 10px #33691e;
}
/* 洞样式：海洋 */
.hole-ocean {
  background-color: #0277bd;
  box-shadow: inset 0 0 10px #01579b;
}

/* 地鼠样式：基础 */
.mole {
  width: clamp(80px, 20vw, 120px);
  height: clamp(70px, 18vw, 100px);
  background-color: #8b4513;
  border-radius: 50% 50% 0 0;
  position: absolute;
  bottom: -100px;
  left: 50%;
  transform: translateX(-50%);
  transition: bottom 0.3s ease;
  box-shadow: 0 0 10px rgba(0,0,0,0.2);
}
/* 地鼠样式：卡通 */
.mole-cartoon {
  background-color: #ed8936;
  border-radius: 50% 50% 20% 20%;
  box-shadow: 0 0 10px #dd6b20;
}
/* 地鼠样式：森林（图片） */
.mole-forest {
  background: url('https://picsum.photos/id/118/120/100') center/cover no-repeat;
  border-radius: 50%;
}
/* 地鼠样式：海洋（图片） */
.mole-ocean {
  background: url('https://picsum.photos/id/152/120/100') center/cover no-repeat;
  border-radius: 50%;
}

/* 地鼠激活 */
.mole.active {
  bottom: 10px;
  animation: bounce 0.2s ease-in-out;
}
@keyframes bounce {
  0% { bottom: 5px; }
  50% { bottom: 15px; }
  100% { bottom: 10px; }
}

/* 游戏结束弹窗 */
.game-over {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: white;
  padding: clamp(20px, 5vw, 30px);
  border-radius: 10px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.2);
  z-index: 100;
  width: clamp(280px, 80vw, 400px);
}
.game-over h2 {
  color: #e63946;
  margin-bottom: 15px;
  font-size: clamp(1.5rem, 4vw, 2rem);
}
.game-over p {
  font-size: clamp(1rem, 3vw, 1.2rem);
  margin-bottom: 15px;
}

/* 4. 排行榜弹窗 */
.rank-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 200;
}
.rank-content {
  background: white;
  padding: 20px;
  border-radius: 10px;
  width: clamp(280px, 80vw, 400px);
  max-height: 80vh;
  overflow-y: auto;
}
.rank-content h3 {
  color: #42b983;
  margin-bottom: 15px;
  font-size: 1.5rem;
}
.rank-list {
  margin-bottom: 20px;
}
.rank-item {
  display: flex;
  justify-content: space-between;
  padding: 8px 10px;
  border-bottom: 1px solid #eee;
  font-size: 1rem;
}
.rank-num {
  color: #e63946;
  font-weight: bold;
  width: 30px;
}
.rank-score {
  flex: 1;
  text-align: center;
}
.rank-time {
  color: #999;
  font-size: 0.9rem;
}
.rank-empty {
  text-align: center;
  color: #999;
  padding: 20px 0;
}

/* 媒体查询优化 */
@media (max-width: 480px) {
  .game-info {
    margin-bottom: 10px;
  }
  .stats {
    font-size: 0.9rem;
  }
  .control-btn {
    margin-bottom: 10px;
  }
  .skin-btn, .diff-btn {
    font-size: 12px;
    padding: 6px 10px;
  }
}

/* 移动端移除hover */
@media (hover: none) {
  .control-btn:hover:not(:disabled) {
    background-color: inherit;
  }
  .reset-btn:hover:not(:disabled) {
    background-color: inherit;
  }
  .skin-btn:hover, .diff-btn:hover {
    background-color: inherit;
  }
  .skin-btn.active:hover, .diff-btn.active:hover {
    background-color: #42b983;
  }
}
</style>