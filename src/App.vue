<template lang="pug">
  #app
    .find-pair
      h3 Игра "Найди пару"
      table.find-pair__field
        tr(
          v-for="(row, i) in gameField"
          :key="i"
          )
          game-cell(
            v-for="(cell, j) in row"
            :key="j"
            :options="cell"
            @click="processingMove(i, j)"
            )
      button.find-pair__toggle-game(@click="toggleGame") {{ gameState | formatButtonText }}
      div {{ stopwatchFormated }}
</template>

<script>
import gameCell from '@/components/gameCell.vue'

// Перечисление цветов для окрашивания клеток игрового поля
const colors = [
  'red',
  'fuchsia',
  'blue',
  'orange',
  'limegreen',
  'aqua',
  'gold',
  'gray'
]

// Перечисление состояний игры
const gameStates = {
  off: 'off', // Игра ещё не начата
  on: 'on', // Игра начата
  paused: 'paused', // Игра поставлена на паузу
  over: 'over' // Игра окончена
}

// Перечисление статусов ячейки игрового поля
const cellStatuses = {
  none: '',
  active: 'active',
  disabled: 'disabled'
}

export default {
  name: 'app',
  components: {
    gameCell
  },
  data: () => ({
    // Параметры секундомера
    stopwatch: new Date(0), // Секундомер
    timerId: '',
    // Параметры игры
    gameState: gameStates.off, // Текущий статус игры
    gameFieldSize: 4, // Размер игрового поля
    gameField: [], // Матрица игрового поля, заполняется в хуке created
    // Параметры хода. Ходом считается последовательный выбор двух клеток, которые сравниваются между собой
    currentMove: [], // Текущий ход. Он состоит из двух выбранных клеток, которые сравниваются между собой
    currentMoveInProcess: false, // При значении true (если ход в обработке), дизаблится игровое поле
    rightMoveCounter: 0 // Счетчик правильных ходов для определения окончания игры
  }),
  computed: {
    gameIsOver() {
      return this.rightMoveCounter === this.gameFieldSize * this.gameFieldSize / 2
    },
    stopwatchFormated() {
      // Вычисляем и форматируем количество миллисекунд
      let ms = String(+this.stopwatch % 1000)
      while (ms.length < 3) {
        ms = "0" + ms;
      }

      // Форматируем и выводим таймер
      const options = {
        minute: 'numeric',
        second: 'numeric',
      }
      return `${this.stopwatch.toLocaleString("ru", options)}.${ms}`
    }
  },
  methods: {
    // Выполнение действий, связанных с изменением статуса игры
    toggleGame() {
      switch (this.gameState) {
        case gameStates.off:
          this.prepareGameField()
          this.gameState = gameStates.on
          this.timerId = setTimeout(this.timer, 4)
          break;
        case gameStates.on:
          this.gameState = gameStates.paused
          break;
        case gameStates.paused:
          this.gameState = gameStates.on
          this.timerId = setTimeout(this.timer, 4)
          break;
        case gameStates.over:
          this.prepareGameField()
          this.gameState = gameStates.on
          this.rightMoveCounter = 0
          this.stopwatch = new Date(0)
          this.timerId = setTimeout(this.timer, 4)
          break;
        default:
          break;
      }
    },
    // Подготовка игрового поля перед началом игры
    prepareGameField() {
      const tmpColors = colors.concat(colors) // Вспомогательный массив для рандомного распределения цветов по полю
      for (let i = 0; i < this.gameFieldSize; i++) {
        for (let j = 0; j < this.gameFieldSize; j++) {
          // Получаем случайный индекс из tmpColors
          let colorIndex = Math.random() * (tmpColors.length)
          colorIndex = Math.floor(colorIndex)

          // Записываем в ячейку цвет и обнуляем её статус
          this.$set(this.gameField[i][j], 'color', tmpColors[colorIndex])
          this.$set(this.gameField[i][j], 'status', cellStatuses.none)

          // Удаляем использованный цвет из вспомогательного массива
          tmpColors.splice(colorIndex, 1)
        }
      }
    },
    // Отсчет времени. Таймер с шагом 4мс, т.к. это минимальная задержка по стандарту http://www.w3.org/TR/html5/webappapis.html#timers
    timer() {
      if (this.gameState === gameStates.on && !this.gameIsOver) {
        this.stopwatch = new Date(+this.stopwatch + 4)
        this.timerId = setTimeout(this.timer, 4)
      }
    },
    // Обработка хода при клике по игровой ячейке
    processingMove(i, j) {
      // Клик обрабатывается если игра не на паузе, игровое поле доступно, и ячейка доступна для выбора
      if ((this.gameState === gameStates.on) && !this.currentMoveInProcess && !this.gameField[i][j].status) {
        // Первые 2 выбранные клетки записываются в текущий ход для последующего сравнения
        if (this.currentMove.length < 2) {
          this.$set(this.gameField[i][j], 'status', 'active')
          // Ячейка текущего хода содержит координаты и цвет
          this.currentMove.push({
            i,
            j,
            color: this.gameField[i][j].color
          })
        }
        // Если 2 ячейки уже выбраны, то ход закончен, и он обрабатывается
        if (this.currentMove.length === 2) {
          // Игровое поле становится недоступным для дальнейшего выбора
          this.currentMoveInProcess = true
          // Если цвета выбранных ячеек совпадают, то ход считается правильным
          if (this.currentMove[0].color === this.currentMove[1].color) {
            this.rightMoveCounter += 1
            this.clearCurrentMove()
            // Т.к. ход правильный, то проверяется, не является ли он последним
            if (this.gameIsOver) {
              this.gameState = gameStates.over
              clearTimeout(this.timerId)

              // Оборачиваем окончание игры в setTimeout, чтобы ячейка успела окраситься в нужный цвет
              setTimeout(this.finishGame, 10)
            }
          } else { // Если ход неправильный, то ставится задержка очистки, чтобы игрок успел запомнить цвета
            setTimeout(this.clearCurrentMove, 1000)
          }
        }
      }
    },
    // Очистка текущего хода
    clearCurrentMove() {
      const newCellStatus = this.currentMove[0].color === this.currentMove[1].color ? cellStatuses.disabled : cellStatuses.none
      this.currentMove.forEach(el => {
        this.gameField[el.i][el.j].status = newCellStatus
      });
      this.currentMove = []
      this.currentMoveInProcess = false
    },
    finishGame() {
      alert(`Вы выиграли! \n Затраченное время: ${this.stopwatchFormated}`)
    },
  },
  filters: {
    formatButtonText(gameState) {
      switch (gameState) {
        case gameStates.off:
          return 'Старт'
        case gameStates.on:
          return 'Пауза'
        case gameStates.paused:
          return 'Продолжить'
        case gameStates.over:
          return 'Старт'
        default:
          break;
      }
    },
  },
  created() {
    // Генерация пустого игрового поля
    for (let i = 0; i < this.gameFieldSize; i++) {
      this.gameField.push([])
      for (let j = 0; j < this.gameFieldSize; j++) {
        this.gameField[i].push({})
      }
    }
  }
}
</script>

<style scoped>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
.find-pair {
  text-align: left;
  width: 200px;
  margin: 0 auto;
}
.find-pair__field {
  border: 1px solid grey;
  border-collapse: collapse;
  margin-bottom: 20px;
}
.find-pair__toggle-game {
  width: 100px;
  margin-bottom: 20px;
}
</style>
