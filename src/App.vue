<template>
  <div id="app">
    <div id="inputs">
      <div id="dimensions">
        <div class="form-group">
          <label for="width">Width</label>
          <input id="width" type="range" min="1" max="100" v-model.number="widthInput"/>
          <span>{{ widthInput }}</span>
        </div>
        <div class="form-group">
          <label for="height">Height</label>
          <input id="height" type="range" min="1" max="100" v-model.number="heightInput"/>
          <span>{{ heightInput }}</span>
        </div>
        <div class="form-group">
          <label for="size">Size</label>
          <input id="size" type="range" min="1" max="100" v-model.number="size"/>
        </div>
        <div class="form-group">
          <label for="speed">Speed</label>
          <input id="speed" type="range" min="1" :max="maxSpeed" v-model.number="speed"/>
        </div>
        <button @click="onNewBoard">Recalculate</button>
      </div>
      <div id="probabilities">
        <div class="form-group" v-for="(probability, index) in probabilities" :key="index">
          <label :for="index.toString()"><img :src="getImageUrl(index)" /></label>
          <input :id="index.toString()" type="number" min="0" max="100" step="0.01" v-model.number="probabilities[index]"/>
        </div>
      </div>
    </div>
    <div id="grid">
      <div class="row" v-for="(row, index) in grid" :key="index">
        <div class="cell" v-for="(cell, innerIndex) in row" :key="innerIndex" :class="getClass(cell)" :style="{ height: size + 'px' }">
          <img :src="getImageUrl(cell.length === 1 ? cell[0] : -1)" :style="{ height: size + 'px', width: size + 'px' }">
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'

@Component
export default class App extends Vue {
  width = 15;
  widthInput = this.width;
  height = 10;
  heightInput = this.height;
  size = 30;
  speed = 1000;
  maxSpeed = 1000;
  grid: number[][][] = [];
  states: string[] = ['red', 'orange', 'yellow', 'green', 'blue', 'purple'];
  probabilities: number[] = [];
  mappings: number[][][] = [
    // Left
    [
      [0, 5, 1],
      [1, 0, 2],
      [2, 1, 3],
      [3, 2, 4],
      [4, 3, 5],
      [5, 4, 0]
    ],
    // Top
    [
      [0, 5, 1],
      [1, 0, 2],
      [2, 1, 3],
      [3, 2, 4],
      [4, 3, 5],
      [5, 4, 0]
    ],
    // Right
    [
      [0, 5, 1],
      [1, 0, 2],
      [2, 1, 3],
      [3, 2, 4],
      [4, 3, 5],
      [5, 4, 0]
    ],
    // Bottom
    [
      [0, 5, 1],
      [1, 0, 2],
      [2, 1, 3],
      [3, 2, 4],
      [4, 3, 5],
      [5, 4, 0]
    ]
  ]

  created (): void {
    this.initializeProbabilities()
    this.onNewBoard()
  }

  initializeProbabilities (): void {
    for (let i = 0; i < this.states.length; i++) {
      this.probabilities.push(Math.ceil(10000 / this.states.length) / 100)
    }
  }

  initializeGrid (): void {
    this.grid = []
    for (let y = 0; y < this.height; y++) {
      const row:number[][] = []
      for (let x = 0; x < this.width; x++) {
        const cell: number[] = []
        for (let i = 0; i < this.states.length; i++) {
          if (this.probabilities[i] > 0) {
            cell.push(i)
          }
        }
        row.push(cell)
      }

      this.grid.push(row)
    }
  }

  onNewBoard (): void {
    this.width = this.widthInput
    this.height = this.heightInput
    this.initializeGrid()
    this.calculateBoard()
  }

  calculateBoard (): void {
    // Stop looping if we have 0 options for a cell
    const noOptions = this.grid.some(row => row.some(cell => cell.length === 0))
    const allChosen = this.grid.every(row => row.every(cell => cell.length === 1))
    const keepLooping = !noOptions && !allChosen

    if (keepLooping) {
      this.chooseCell()
      this.recalculateGrid()
      this.grid = [...this.grid]
      const delay = this.maxSpeed < this.speed ? 0 : this.maxSpeed - this.speed
      setTimeout(() => this.calculateBoard(), delay)
    }
  }

  chooseCell (): void {
    // 1) Find the min entropy
    let indexes: number[][] = []
    let minEntropy = this.states.length

    for (let y = 0; y < this.height; y++) {
      for (let x = 0; x < this.width; x++) {
        const cell = this.grid[y][x]
        const cellEntropy = cell.length
        if (cellEntropy > 1 && cellEntropy < minEntropy) {
          indexes = [[x, y]]
          minEntropy = cellEntropy
        } else if (cellEntropy === minEntropy) {
          indexes.push([x, y])
        }
      }
    }

    if (indexes.length > 0) {
      // Select a cell at random
      const cell = indexes[Math.floor(Math.random() * indexes.length)]

      // Based on the cell's options and the configured probabilities, choose a value
      let nextValue = 0

      const randomValue = Math.random() * this.probabilities.reduce((sum, p) => sum + p, 0)

      const x = cell[0]
      const y = cell[1]

      for (const value of this.grid[y][x]) {
        const previousValue = nextValue
        // Adjust the probability based on the available options
        nextValue += this.probabilities[value] * this.probabilities.filter((p, index) => this.grid[y][x].includes(index)).reduce((sum, p) => sum + p, 0) / this.probabilities.reduce((sum, p) => sum + p, 0)

        if (previousValue < randomValue && randomValue <= nextValue) {
          this.grid[y][x] = [value]
          break
        }
      }
    } else {
      alert('Error')
      this.grid = []
    }
  }

  recalculateGrid (): void {
    let changeFound
    do {
      changeFound = false
      for (let y = 0; y < this.height; y++) {
        for (let x = 0; x < this.width; x++) {
          if (this.grid[y][x].length === 1) {
            continue
          }
          const optionGroups: number[][] = []
          if (x > 0) {
            optionGroups.push([...new Set(this.grid[y][x - 1].flatMap(value => this.mappings[0][value]))])
          }
          if (y > 0) {
            optionGroups.push([...new Set(this.grid[y - 1][x].flatMap(value => this.mappings[1][value]))])
          }
          if (x < this.width - 1) {
            optionGroups.push([...new Set(this.grid[y][x + 1].flatMap(value => this.mappings[2][value]))])
          }
          if (y < this.height - 1) {
            optionGroups.push([...new Set(this.grid[y + 1][x].flatMap(value => this.mappings[3][value]))])
          }

          let options = optionGroups[0]

          for (let i = 1; i < optionGroups.length; i++) {
            options = [...new Set([...options].filter(j => new Set(optionGroups[i]).has(j)))]
          }

          if (options.length < this.grid[y][x].length) {
            changeFound = true
            this.grid[y][x] = options
          }
        }
      }
    } while (changeFound)
  }

  getClass (values: number[]): string {
    if (values.length !== 1) {
      return 'blank'
    } else {
      return ['red', 'orange', 'yellow', 'green', 'blue', 'purple'][values[0]]
    }
  }

  getImageUrl (value: number) {
    const images = require.context('./assets/', false, /\.svg$/)
    const fileName = value === -1 ? 'blank' : this.states[value]
    return images(`./${fileName}.svg`)
  }
}

</script>

<style>
html, body {
  display: flex;
  flex: 1;
  flex-direction: column;
  align-items: center;
  height: 100%;
  justify-content: center;
  overflow: hidden;
}

body {
  width: 100%;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  overflow: hidden;
  flex: 1;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

#grid {
  overflow: auto;
  max-height: 75vh;
  max-width: 99vw;
  flex: 1;
}

.form-group {
  display: block;
  width: 10rem;
}

.row {
  display: flex;
  justify-content: center;
}

label {
  display: block;
}

#dimensions {
  display: flex;
}

#dimensions .form-group {
  margin-right: 1rem;
}

#probabilities {
  display: flex;
  margin: 1rem 0;
}
</style>
