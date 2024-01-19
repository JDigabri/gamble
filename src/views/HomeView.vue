<template>
  <div class="main">
    <h1>Betting Game</h1>
    <div v-if="!gameStarted">
      <div>
        <label for="betAmount">Bet Amount:</label>
        <input v-model.number="betAmount" type="number" id="betAmount" min="1">
      </div>
      <button @click="startGame">Start Game</button>
    </div>
    <div v-else>
      <button @click="pullOut">Pull Out</button>
      <p>Multiplier: {{ currentMultiplier.toFixed(2) }}x</p>
    </div>
    <div>Balance: {{ balance.toFixed(2) }}</div>
    <div ref="messageRef" v-if="message" class="message">{{ message }}</div>
    <div class="past-crashes">
    <h2>Past Crashes</h2>
    <ul>
      <li v-for="(crash, index) in pastCrashes" :key="index">
        {{ crash.toFixed(2) }}x
      </li>
    </ul>
  </div>
  </div>
</template>

<script>
import { gsap } from "gsap";


export default {
  data() {
    return {
      balance: 100,
      betAmount: 1,
      houseEdgePercent: 6.66,
      message: '',
      gameStarted: false,
      crashTick: null,
      currentMultiplier: 1.00,
      gameInterval: null,
      pastCrashes: [],

    };
  },
  methods: {
    generateRandomSeed(length) {
      const array = new Uint8Array(length);
      window.crypto.getRandomValues(array);
      return Array.from(array).map(b => b.toString(16).padStart(2, '0')).join('');
    },
    async hashData(data) {
      const encoder = new TextEncoder();
      const encodedData = encoder.encode(data);
      const hashBuffer = await crypto.subtle.digest('SHA-256', encodedData);
      const hashArray = Array.from(new Uint8Array(hashBuffer));
      return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    },
    async calculateCrashTick(seed) {
      const hash = await this.hashData(seed);
      const h = parseInt(hash.slice(0, 52 / 4), 16);
      const e = Math.pow(2, 52);
      const result = (100 * e - h) / (e - h);
      const houseEdgeModifier = 1 - this.houseEdgePercent / 100;
      return Math.max(100, Math.floor(result * houseEdgeModifier));
    },
    startGame() {
      if (this.betAmount > this.balance) {
        this.message = "Insufficient balance to place bet.";
        return;
      }

      this.balance -= this.betAmount;
      this.gameStarted = true;
      this.message = '';
      this.currentMultiplier = 1.00;
      const seed = this.generateRandomSeed(32);
      this.calculateCrashTick(seed).then(crashTick => {
        this.crashTick = crashTick / 100;
        this.currentMultiplier = 1.00;
        this.gameInterval = setInterval(() => {
          if (this.currentMultiplier >= this.crashTick) {
            clearInterval(this.gameInterval);
            this.gameOver();
          } else {
            this.currentMultiplier += 0.01;
          }
        }, 100); // Adjust the interval as needed
      });
    },
    pullOut() {
      clearInterval(this.gameInterval);
      const winnings = this.betAmount * this.currentMultiplier;
      this.balance += winnings;
      this.message = `You pulled out at ${this.currentMultiplier.toFixed(2)}x. New balance: ${this.balance.toFixed(2)}`;
      this.pastCrashes.push(this.crashTick); // Add the crash tick to the past crashes
      this.gameStarted = false;
    },

    gameOver() {
      this.message = `Game crashed at ${this.currentMultiplier.toFixed(2)}x. You lost the bet.`;
      this.pastCrashes.push(this.crashTick); // Add the crash tick to the past crashes
      this.gameStarted = false;
    },

    animateWin() {
      gsap.to(this.$refs.messageRef, { duration: 1, color: "#28a745", scale: 1.2, yoyo: true, repeat: 1 });
    },

    animateLoss() {
      gsap.to(this.$refs.messageRef, { duration: 0.1, x: -10, repeat: 5, yoyo: true, ease: "power1.inOut" });
      gsap.to(this.$refs.messageRef, { duration: 1, color: "#d9534f", delay: 0.5 });
    },
  },
};
</script>

<style>

.main {
  font-family: 'Arial', sans-serif;
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  background-color: #f9f9f9;
  display: flex;
  flex-direction: column;
  align-items: center;

} 
#app {
  font-family: 'Arial', sans-serif;
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  background-color: #f9f9f9;
  display: flex;

}

h1 {
  text-align: center;
  color: #333;
}

div {
  margin-bottom: 15px;
}

label {
  display: block;
  margin-bottom: 5px;
  color: #555;
}

input[type="number"] {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-sizing: border-box;
  margin-bottom: 10px;
}

button {
  width: 100%;
  padding: 10px;
  border: none;
  border-radius: 4px;
  background-color: #007bff;
  color: white;
  cursor: pointer;
  font-size: 16px;
}

button:hover {
  background-color: #0056b3;
}

p, .div {
  text-align: center;
  color: #333;
  font-size: 18px;
}

.message {
  text-align: center;
  color: #d9534f;
  font-size: 16px;
}

.past-crashes {
    margin-top: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .past-crashes h2 {
    font-size: 20px;
    color: #333;
  }

  .past-crashes ul {
    list-style: none;
    padding: 0;
  }

  .past-crashes li {
    margin-bottom: 5px;
    color: #555;
  }
</style>
