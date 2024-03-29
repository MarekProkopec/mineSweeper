<template>
  <v-app>
    <div class="app">
      <div class="gameInfo">
        <v-card v-if="showEndMeesage" class="endMessage">
          <v-card-text>
            <div class="title text-h4">
              <h1 :class="won ? 'won' : 'lost'">
                You {{ won ? "won" : "Lost" }}
              </h1>
            </div>
          </v-card-text>
        </v-card>
        <v-card>
          <v-card-title> AI Controls </v-card-title>
          <v-card-text class="d-flex justify-space-between">
            <v-btn @click="makeAiMove">Move</v-btn>
            <v-btn @click="aiGame">Play entire game</v-btn>
          </v-card-text>
        </v-card>
        <v-card>
          <v-card-title> Game info </v-card-title>
          <v-card-text>
            <p>Bombs left: {{ bombsLeft }}</p>
            <p class="difficulty">
              Difficulty:
              <span :class="selectedDifficulty">{{ selectedDifficulty }}</span>
            </p>
          </v-card-text>
        </v-card>
        <v-card>
          <v-card-title> Controls </v-card-title>
          <v-card-text class="d-flex justify-space-between">
            <v-btn @click="reset()">reset</v-btn>
            <v-btn @click="playEmpty()">Play empty</v-btn>
          </v-card-text>
        </v-card>
        <v-card>
          <v-card-title> Select difficulty </v-card-title>
          <v-card-text class="d-flex justify-space-between align-center">
            <v-btn
              text
              v-for="difficulty in difficulties"
              :key="difficulty.name"
              @click="changeDifficulty(difficulty)"
              :class="difficulty.name"
            >
              {{ difficulty.name }}
            </v-btn>
          </v-card-text>
        </v-card>
      </div>

      <div class="minesweeper">
        <div class="tableRow" v-for="(row, y) in revealed" :key="y">
          <div
            class="cell"
            v-for="(cell, x) in row"
            :key="`${x * fieldSize}${cell}`"
            @mouseup="handleMove(x, y, $event)"
            :class="`c-${cell}`"
          >
            {{ cell == "h" ? "" : cell }}
          </div>
        </div>
      </div>
    </div>
  </v-app>
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
    bombsLeft() {
      let num = this.bombCount;

      for (let row of this.revealed) {
        num -= row.filter((e) => {
          return e == "f";
        }).length;
      }
      return num;
    },
  },
  methods: {
    makeAiMove(oneMove = false) {
      let madeMove = false;
      for (let y = 0; y < this.revealed.length; y++) {
        const row = this.revealed[y];
        for (let x = 0; x < row.length; x++) {
          const item = row[x];
          if (item == "h") continue;
          if (isNaN(item)) continue;

          let emptyFields = [];
          let bombFields = [];
          let hiddenFields = [];

          const bombCount = Number(item);

          for (let i = -1; i < 2; i++) {
            for (let j = -1; j < 2; j++) {
              const rx = x + i;
              const ry = y + j;

              if (
                rx < 0 ||
                ry < 0 ||
                ry >= this.revealed.length ||
                rx >= this.revealed[0].length ||
                (x == 0 && y == 0)
              )
                continue;
              const field = this.revealed[ry][rx];
              if (field == "" || !isNaN(field)) emptyFields.push([rx, ry]);
              else if (field == "h") hiddenFields.push([rx, ry]);
              else if (field == "f") bombFields.push([rx, ry]);
            }
          }

          if (hiddenFields.length == 0) continue;

          // Flag all bombs
          if (bombCount - bombFields.length == hiddenFields.length) {
            for (let bomb of hiddenFields) {
              this.handleMove(...bomb, { button: 2 });
              madeMove = true;
              if (oneMove) return;
            }
          }
          if (bombCount - bombFields.length == 0) {
            for (let field of hiddenFields) {
              this.handleMove(...field);
              madeMove = true;
              if (oneMove) return;
            }
          }
        }
      }
      return madeMove;
    },
    aiGame() {
      while (this.makeAiMove()) {
        setTimeout(() => {
          this.aiGame();
        }, 10);
        return;
      }
      console.log("No moves available");
    },
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
        if (this.revealed[y][x] == "f") return;
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
    playEmpty(onlyOne = true) {
      for (let y = 0; y < this.field.length; y++) {
        const row = this.field[y];
        for (let x = 0; x < row.length; x++) {
          const item = row[x];
          if (item == "" && this.revealed[y][x] == "h") {
            this.handleMove(x, y);
            if (onlyOne) return;
          }
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

$main1: #171717;
$main2: #444444;
$main3: #da0037;
$main4: #ededed;

@font-face {
  font-family: DSEG;
  src: url("./assets/fonts/DSEG7-Classic/DSEG7Classic-Bold.woff2");
}

html {
  overflow: hidden !important;
}

.app {
  height: 100vw;
  height: 100vh;
  display: flex;
  justify-content: space-evenly;
  align-items: center;
  // flex-direction: column;
  gap: 5vw;
  background: $main1;
  color: $main4;
  padding: 1vw;

  .gameInfo {
    display: flex;
    flex-direction: column;
    gap: 20px;

    .endMessage {
      h1 {
        &.won {
          color: green;
        }
        &.lost {
          color: red;
        }
      }
    }

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

  .minesweeper {
    background: $main2;

    // border: solid 3px $main3;
    box-shadow: 0 0 0 4px $main3;
    padding: 1px;
    border-radius: 10px;

    font-family: DSEG;
    height: min(50vw, 90vh);
    aspect-ratio: 1 / 1;

    display: flex;
    flex-direction: column;
    justify-content: space-evenly;
    background: $main3;
    $cellGap: 2px;
    gap: $cellGap;

    .tableRow {
      display: flex;
      height: 100%;
      justify-content: space-evenly;
      align-items: center;
      gap: $cellGap;

      .cell {
        background: $main1;
        border-radius: 3px;

        // border: solid 1px rgba($color: $main3, $alpha: 0.3);
        height: 100%;
        width: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        cursor: pointer;
        font-size: 100%;
        user-select: none;
        overflow: hidden;

        &.c-h {
          background: $main2;
          // background: rgba($color: #000000, $alpha: 0.1);
          transition: background linear 0.1s;
          &:hover {
            background: darken($color: $main2, $amount: 5);
          }
        }
        &.c-b,
        &.c-f {
          background-position: center;
          background-repeat: no-repeat;
          background-size: cover;
          color: transparent;

          &.c-b {
            background-image: url("./assets/images/bomb.png");
            background-color: $main3;
          }
          &.c-f {
            // background-image: url("https://img1.cgtrader.com/items/3764877/1dfa3f1782/large/minesweeper-flag-icon-v1-002-3d-model-low-poly-max-obj-3ds-fbx-ma-stl.jpg");
            background-image: url("./assets/images/flag.png");
            background-color: lighten($color: $main1, $amount: 8);
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
          color: #0f4c75;
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
}
</style>
