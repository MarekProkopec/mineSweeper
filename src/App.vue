<template>
  <div id="app">
    <div class="endMessage">
      <h1 v-if="showEndMeesage" :class="won ? 'won' : 'lost'">
        You {{ won ? "won" : "Lost" }}
      </h1>
    </div>
    <div class="minesweeper">
      <div class="row" v-for="(row, y) in revealed" :key="y">
        <div
          class="cell"
          v-for="(cell, x) in row"
          :key="`${x}${cell}`"
          @mouseup="handleMove(x, y, $event)"
          :class="`c-${cell}`"
        >
          {{ cell == "h" ? "" : cell }}
        </div>
      </div>
    </div>
    <div class="bottomSelect">
      <button @click="reset()">reset</button>
      <div class="difficulties">
        <button
          v-for="difficulty in difficulties"
          :key="difficulty.name"
          @click="changeDifficulty(difficulty)"
          :class="difficulty.name"
        >
          {{ difficulty.name }}
        </button>
      </div>
      <p class="difficulty">
        Difficulty:
        <span :class="selectedDifficulty">{{ selectedDifficulty }}</span>
      </p>
    </div>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      fieldSize: 5,
      bombCount: 3,
      showEndMeesage: false,
      difficulties: [
        { name: "Beginner", size: 9, mines: 10 },
        { name: "Intermediate", size: 16, mines: 40 },
        { name: "Expert", size: 22, mines: 99 },
      ],
      selectedDifficulty: "",

      // b: bomb
      // {number}: distance from bomb
      // "": empty
      field: [],
      // "h": hidden
      // "": revealed
      // f: flag
      // {number}: distance from bomb
      revealed: [],
    };
  },
  computed: {
    won() {
      for (let y in this.revealed) {
        for (let x in this.revealed[y]) {
          const hidden = this.field[y][x];
          const shown = this.revealed[y][x];
          if (hidden != "b" && shown == "h") return false;
        }
      }
      return true;
    },
  },
  methods: {
    changeDifficulty(difficulty) {
      this.fieldSize = difficulty.size;
      this.bombCount = difficulty.mines;
      this.selectedDifficulty = difficulty.name;
      this.reset();
    },
    initField() {
      // Initialize field
      const field = [];
      for (let i = 0; i < this.fieldSize; i++) {
        const row = [];
        for (let j = 0; j < this.fieldSize; j++) {
          row.push("");
        }
        field.push(row);
      }

      // Add bombs
      if (this.bombCount > this.fieldSize * this.fieldSize)
        this.bombCount = this.fieldSize * this.fieldSize;
      for (let i = 0; i < this.bombCount; i++) {
        const x = Math.floor(Math.random() * this.fieldSize);
        const y = Math.floor(Math.random() * this.fieldSize);

        if (field[y][x] == "b") {
          i--;
          continue;
        }

        field[y][x] = "b";
      }

      // Set distances from bombs
      for (let y = 0; y < field.length; y++) {
        for (let x = 0; x < field[y].length; x++) {
          const cell = field[y][x];
          if (cell != "") continue;

          let bombCount = 0;
          for (let i = -1; i < 2; i++) {
            for (let j = -1; j < 2; j++) {
              const ry = y + i;
              const rx = x + j;
              if (
                ry < 0 ||
                rx < 0 ||
                ry >= field.length ||
                rx >= field[y].length ||
                (i == 0 && j == 0)
              )
                continue;
              if (field[ry][rx] == "b") bombCount++;
            }
          }

          if (bombCount != 0) field[y][x] = bombCount;
        }
      }

      this.field = field;
    },
    initRevealed() {
      const field = [];
      for (let y = 0; y < this.fieldSize; y++) {
        const row = [];
        for (let x = 0; x < this.fieldSize; x++) {
          row.push("h");
        }
        field.push(row);
      }

      this.revealed = field;
    },
    reset() {
      this.showEndMeesage = false;
      this.initField();
      this.initRevealed();
    },
    unfoldEmpty(x, y) {
      const value = this.field[y][x];
      if (value != "") return;
      this.$set(this.revealed[y], x, value);

      // Unfold empty fields
      const checks = [
        [0, -1],
        [0, 1],
        [-1, 0],
        [1, 0],
      ];
      for (let item of checks) {
        const ry = y + item[1];
        const rx = x + item[0];
        if (
          ry < 0 ||
          rx < 0 ||
          ry >= this.field.length ||
          rx >= this.field[y].length
        )
          continue;

        if (this.field[ry][rx] == "" && this.revealed[ry][rx] == "h") {
          this.unfoldEmpty(rx, ry);
          continue;
        }
      }

      // Show border numbers
      for (let i = -1; i < 2; i++) {
        for (let j = -1; j < 2; j++) {
          const ry = y + i;
          const rx = x + j;
          if (
            ry < 0 ||
            rx < 0 ||
            ry >= this.field.length ||
            rx >= this.field[y].length
          )
            continue;
          if (this.field[ry][rx] != "" && !isNaN(this.field[ry][rx])) {
            this.$set(this.revealed[ry], rx, this.field[ry][rx]);
          }
        }
      }
    },
    lost() {
      this.$nextTick(() => {
        this.showEndMeesage = true;
      });
    },
    handleMove(x, y, event = { button: 0 }) {
      if (this.showEndMeesage) return;

      // Left click
      if (event.button == 0) {
        const value = this.field[y][x];
        this.$set(this.revealed[y], x, value);

        if (value == "") this.unfoldEmpty(x, y);
        if (value == "b") this.lost();
      }

      if (event.button == 2) {
        if (this.revealed[y][x] == "h") {
          this.$set(this.revealed[y], x, "f");
        } else if (this.revealed[y][x] == "f") {
          this.$set(this.revealed[y], x, "h");
        }
      }
    },
  },
  created() {
    this.selectedDifficulty = this.difficulties[0].name;
    this.changeDifficulty(this.difficulties[0]);
    document.addEventListener(
      "contextmenu",
      function (evt) {
        evt.preventDefault();
      },
      false
    );

    this.reset();
  },
  watch: {
    won(newValue, oldValue) {
      if (newValue)
        this.$nextTick(() => {
          this.showEndMeesage = true;
        });
    },
  },
};
</script>

<style lang="scss">
@import url("./styles/reset.css");

@font-face {
  font-family: DSEG;
  src: url("./assets/fonts/DSEG7-Classic/DSEG7Classic-Bold.woff2");
}

#app {
  height: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  gap: 10vh;
  padding: Min(1vh, 1vw);

  .endMessage {
    $size: 10;
    $sizeStyle: Min(#{$size}vh, #{$size}vw);
    height: $sizeStyle;
    h1 {
      max-height: 100%;
      font-size: $sizeStyle;
      &.won {
        color: green;
      }
      &.lost {
        color: red;
      }
    }
  }

  .minesweeper {
    border: solid 2px rgba($color: #000000, $alpha: 0.2);
    $size: 70;
    $sizeStyle: Min(#{$size}vh, #{$size}vw);
    height: $sizeStyle;
    width: $sizeStyle;
    font-family: DSEG;

    display: flex;
    flex-direction: column;
    justify-content: space-evenly;

    .row {
      display: flex;
      height: 100%;
      justify-content: space-evenly;
      align-items: center;

      .cell {
        border: solid 1px rgba($color: #000000, $alpha: 0.1);
        height: 100%;
        width: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        cursor: pointer;
        background: white;
        font-size: 100%;
        user-select: none;

        &.c-h {
          background: rgba($color: #000000, $alpha: 0.05);
          transition: background linear 0.1s;
          &:hover {
            background: rgba($color: #000000, $alpha: 0.1);
          }
        }
        &.c-b,
        &.c-f {
          background-position: center;
          background-repeat: no-repeat;
          background-size: cover;
          color: transparent;

          &.c-b {
            background-image: url("https://www.giantbomb.com/a/uploads/scale_small/8/87790/3216800-icon_mine.png");
          }
          &.c-f {
            background-image: url("https://img1.cgtrader.com/items/3764877/1dfa3f1782/large/minesweeper-flag-icon-v1-002-3d-model-low-poly-max-obj-3ds-fbx-ma-stl.jpg");
            background-color: rgba($color: #000000, $alpha: 0.3);
          }
        }
        &.c-1 {
          color: #0000ff;
        }
        &.c-2 {
          color: #008200;
        }
        &.c-3 {
          color: #fe0000;
        }
        &.c-4 {
          color: #000084;
        }
        &.c-5 {
          color: #840000;
        }
        &.c-6 {
          color: #008284;
        }
        &.c-7 {
          color: #840084;
        }
        &.c-8 {
          color: #757575;
        }
      }
    }
  }

  .bottomSelect {
    $size: 15;
    $sizeStyle: Min(#{$size}vh, #{$size}vw);
    height: $sizeStyle;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    gap: Min(1vh, 1vw);

    .difficulties {
      display: flex;
      justify-content: center;
      align-items: center;
      gap: 10px;
    }

    p.difficulty {
      span {
        text-transform: uppercase;
        font-weight: bold;
      }
    }
    button {
      text-transform: uppercase;
      padding: 8px 16px;
      cursor: pointer;
    }
    .Beginner {
      color: green;
    }
    .Intermediate {
      color: orange;
    }
    .Expert {
      color: red;
    }
  }
}
</style>
