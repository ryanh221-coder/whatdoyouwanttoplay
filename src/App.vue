<template>
  <div class="app">
    <header class="header">
      <div class="h1">What Do You Want To Play?</div>
      <div v-if="phase === 'setup'" class="sub">
        Pass the phone. Tap a card to eliminate it.
      </div>
      <div v-else class="sub">
        Remaining: <b>{{ alive.length }}</b>
        <span v-if="phase === 'play'">
          | Turn: <b>Player {{ turn + 1 }}</b>
        </span>
      </div>
    </header>

    <div v-if="phase === 'setup'" class="panel">
      <label class="label">
        Players tonight
        <input
          class="input"
          type="number"
          min="1"
          max="20"
          v-model.number="playerCount"
        />
      </label>

      <button class="primary" @click="start">Start</button>

      <div class="small">
        Library loaded: <b>{{ allGames.length }}</b> games
      </div>
    </div>

    <div v-else-if="phase === 'play' && pair" class="grid2">
      <button class="card" @click="eliminate(pair[0].id)">
        <div class="cardTop">
          <span v-if="showsBestAt(pair[0])" class="badge">Best at {{ playerCount }}</span>
          <div class="title">{{ pair[0].name }}</div>
          <div class="desc">{{ pair[0].descriptor }}</div>
        </div>

        <div class="stats">
          <span class="pill">{{ pair[0].players.min }}-{{ pair[0].players.max }}p</span>
          <span class="pill">
            {{ pair[0].playtimeMinutes.min }}<span v-if="pair[0].playtimeMinutes.max !== pair[0].playtimeMinutes.min">-{{ pair[0].playtimeMinutes.max }}</span>m
          </span>
          <span class="pill">Wt {{ to2(pair[0].weight) }}</span>
          <span class="pill">⭐ {{ to2(pair[0].rating) }}</span>
        </div>

        <img
          v-if="pair[0].image"
          class="cover"
          :src="imgSrc(pair[0].image)"
          :alt="pair[0].name"
          @error="hideImage"
        />

        <div class="tapHint">Tap to eliminate</div>
      </button>

      <button class="card" @click="eliminate(pair[1].id)">
        <div class="cardTop">
          <span v-if="showsBestAt(pair[1])" class="badge">Best at {{ playerCount }}</span>
          <div class="title">{{ pair[1].name }}</div>
          <div class="desc">{{ pair[1].descriptor }}</div>
        </div>

        <div class="stats">
          <span class="pill">{{ pair[1].players.min }}-{{ pair[1].players.max }}p</span>
          <span class="pill">
            {{ pair[1].playtimeMinutes.min }}<span v-if="pair[1].playtimeMinutes.max !== pair[1].playtimeMinutes.min">-{{ pair[1].playtimeMinutes.max }}</span>m
          </span>
          <span class="pill">Wt {{ to2(pair[1].weight) }}</span>
          <span class="pill">⭐ {{ to2(pair[1].rating) }}</span>
        </div>

        <img
          v-if="pair[1].image"
          class="cover"
          :src="imgSrc(pair[1].image)"
          :alt="pair[1].name"
          @error="hideImage"
        />

        <div class="tapHint">Tap to eliminate</div>
      </button>
    </div>

    <div v-else-if="phase === 'finalists'" class="panel">
      <div class="h2">Finalists</div>

      <div class="finalists">
        <div v-for="g in finalists" :key="g.id" class="finalistRow">
          <span v-if="showsBestAt(g)" class="badge">Best at {{ playerCount }}</span>
          <span class="finalistName">{{ g.name }}</span>
        </div>
      </div>

      <button class="primary" @click="pickRandomWinner">
        Randomly pick tonight's game
      </button>
      <button class="secondary" @click="reset">Start over</button>
    </div>

    <div v-else-if="phase === 'result' && winner" class="panel">
      <div class="h2">Tonight we play</div>
      <div class="winner">{{ winner.name }}</div>
      <div class="desc">{{ winner.descriptor }}</div>
      <button class="primary" @click="reset">Start over</button>
    </div>
  </div>
</template>

<script setup>
import { computed, ref } from "vue";
import gamesData from "./data/games.json";

const allGames = computed(() => gamesData);

const playerCount = ref(4);

const phase = ref("setup"); // setup | play | finalists | result
const alive = ref([]);
const pair = ref(null);
const turn = ref(0);
const recentlyShown = ref([]);
const finalists = ref([]);
const winner = ref(null);

function shuffle(arr) {
  const a = [...arr];
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

function sampleTwo(aliveList, recentIds) {
  const recentSet = new Set(recentIds);
  const candidates = aliveList.filter((g) => !recentSet.has(g.id));
  const pool = candidates.length >= 2 ? candidates : aliveList;

  const a = pool[Math.floor(Math.random() * pool.length)];
  let b = pool[Math.floor(Math.random() * pool.length)];
  while (b.id === a.id) b = pool[Math.floor(Math.random() * pool.length)];
  return [a, b];
}

function start() {
  const initial = shuffle(allGames.value);

  if (initial.length <= playerCount.value) {
    alive.value = initial;
    finalists.value = initial;
    phase.value = "finalists";
    pair.value = null;
    turn.value = 0;
    recentlyShown.value = [];
    winner.value = null;
    return;
  }

  const [a, b] = sampleTwo(initial, []);
  alive.value = initial;
  pair.value = [a, b];
  phase.value = "play";
  turn.value = 0;
  recentlyShown.value = [a.id, b.id];
  finalists.value = [];
  winner.value = null;
}

function eliminate(gameId) {
  if (!pair.value) return;

  const nextAlive = alive.value.filter((g) => g.id !== gameId);

  if (nextAlive.length <= playerCount.value) {
    alive.value = nextAlive;
    finalists.value = nextAlive;
    phase.value = "finalists";
    pair.value = null;
    return;
  }

  const [a, b] = sampleTwo(nextAlive, recentlyShown.value.slice(-6));
  alive.value = nextAlive;
  pair.value = [a, b];
  turn.value = (turn.value + 1) % playerCount.value;
  recentlyShown.value = [...recentlyShown.value, a.id, b.id].slice(-12);
}

function pickRandomWinner() {
  const w = finalists.value[Math.floor(Math.random() * finalists.value.length)];
  winner.value = w;
  phase.value = "result";
}

function reset() {
  phase.value = "setup";
  alive.value = [];
  pair.value = null;
  turn.value = 0;
  recentlyShown.value = [];
  finalists.value = [];
  winner.value = null;
}

function showsBestAt(game) {
  return Array.isArray(game.bestAt) && game.bestAt.includes(playerCount.value);
}

function to2(n) {
  const x = Number(n);
  return Number.isFinite(x) ? x.toFixed(2) : "";
}

function imgSrc(imagePath) {
  // Your JSON uses "images/xyz.jpg" and Vite serves /public as /
  return "/" + String(imagePath).replace(/^\/+/, "");
}

function hideImage(e) {
  e.target.style.display = "none";
}
</script>

<style>
:root {
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
  line-height: 1.3;
}

body {
  margin: 0;
  background: #0b0d12;
  color: #e9eef7;
}

.app {
  max-width: 900px;
  margin: 0 auto;
  padding: 18px;
}

.header {
  margin-bottom: 14px;
}

.h1 {
  font-size: 1.4rem;
  font-weight: 700;
}

.h2 {
  font-size: 1.2rem;
  font-weight: 700;
  margin-bottom: 10px;
}

.sub {
  margin-top: 6px;
  opacity: 0.85;
  font-size: 0.95rem;
}

.panel {
  background: #121625;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 18px;
  padding: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
}

.label {
  display: grid;
  gap: 8px;
  margin-bottom: 12px;
  font-size: 0.95rem;
}

.input {
  width: 100%;
  max-width: 160px;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.14);
  background: #0b0d12;
  color: #e9eef7;
  font-size: 1rem;
}

.primary,
.secondary {
  width: 100%;
  padding: 12px 14px;
  border-radius: 14px;
  border: 1px solid rgba(255, 255, 255, 0.14);
  background: #1b2440;
  color: #e9eef7;
  font-size: 1rem;
  font-weight: 650;
  cursor: pointer;
}

.secondary {
  margin-top: 10px;
  background: transparent;
}

.small {
  margin-top: 10px;
  opacity: 0.8;
  font-size: 0.9rem;
}

.grid2 {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
}

@media (min-width: 700px) {
  .grid2 {
    grid-template-columns: 1fr 1fr;
  }
}

.card {
  text-align: left;
  background: #121625;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 18px;
  padding: 14px;
  cursor: pointer;
  color: inherit;
}

.card:active {
  transform: scale(0.99);
}

.cardTop {
  display: grid;
  gap: 8px;
}

.badge {
  display: inline-block;
  padding: 4px 8px;
  border-radius: 999px;
  background-color: #e6f2ec;
  color: #2f6f5a;
  font-size: 0.78rem;
  font-weight: 600;
  width: fit-content;
}

.title {
  font-size: 1.05rem;
  font-weight: 750;
}

.desc {
  opacity: 0.88;
  font-size: 0.95rem;
}

.stats {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 12px;
}

.pill {
  padding: 6px 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.08);
  font-size: 0.85rem;
  opacity: 0.95;
}

.cover {
  width: 100%;
  height: 180px;
  object-fit: cover;
  border-radius: 14px;
  margin-top: 12px;
  border: 1px solid rgba(255, 255, 255, 0.08);
}

.tapHint {
  margin-top: 10px;
  opacity: 0.7;
  font-size: 0.85rem;
}

.finalists {
  display: grid;
  gap: 10px;
  margin: 12px 0 14px;
}

.finalistRow {
  display: flex;
  align-items: center;
  gap: 10px;
}

.finalistName {
  font-weight: 650;
}

.winner {
  font-size: 1.4rem;
  font-weight: 800;
  margin: 8px 0 6px;
}
</style>
