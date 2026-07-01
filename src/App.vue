<script setup lang="ts">
import { computed, ref } from 'vue'
import {
  ArrowRight, Check, ChevronDown, CircleHelp, Lightbulb, Minus, Plus,
  RotateCcw, Sparkles, Trophy, X, Zap,
} from '@lucide/vue'

type Money = {
  id: string
  name: string
  shortName: string
  value: number
  kind: 'bill' | 'coin'
  portrait: string
  fact: string
}
type Round = {
  target: number
  inventory: Record<string, number>
  optimalCount: number
  optimalSelection: Record<string, number>
}

const denominations: Money[] = [
  { id: 'twenty', name: 'Twenty-dollar bill', shortName: '$20', value: 2000, kind: 'bill', portrait: 'JACKSON', fact: 'Andrew Jackson appears on the $20 bill.' },
  { id: 'ten', name: 'Ten-dollar bill', shortName: '$10', value: 1000, kind: 'bill', portrait: 'HAMILTON', fact: 'Alexander Hamilton appears on the $10 bill.' },
  { id: 'five', name: 'Five-dollar bill', shortName: '$5', value: 500, kind: 'bill', portrait: 'LINCOLN', fact: 'Abraham Lincoln appears on both the $5 bill and the penny.' },
  { id: 'one', name: 'One-dollar bill', shortName: '$1', value: 100, kind: 'bill', portrait: 'WASHINGTON', fact: 'The $1 bill features George Washington.' },
  { id: 'quarter', name: 'Quarter', shortName: '25¢', value: 25, kind: 'coin', portrait: 'QUARTER', fact: 'Four quarters make one dollar.' },
  { id: 'dime', name: 'Dime', shortName: '10¢', value: 10, kind: 'coin', portrait: 'DIME', fact: 'The dime is worth 10¢, even though it is the smallest U.S. coin.' },
  { id: 'nickel', name: 'Nickel', shortName: '5¢', value: 5, kind: 'coin', portrait: 'NICKEL', fact: 'A nickel is worth 5¢ and is larger than a dime.' },
  { id: 'penny', name: 'Penny', shortName: '1¢', value: 1, kind: 'coin', portrait: 'PENNY', fact: 'One hundred pennies make one dollar.' },
]

const level = ref(1)
const score = ref(0)
const streak = ref(0)
const solved = ref(0)
const selected = ref<Record<string, number>>({})
const result = ref<'idle' | 'short' | 'over' | 'correct' | 'not-optimal'>('idle')
const hintStep = ref(0)
const showGuide = ref(false)
const showLevelMenu = ref(false)

function formatMoney(cents: number) {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(cents / 100)
}

function solveBounded(target: number, inventory: Record<string, number>) {
  const unreachable = 9999
  const dp = Array(target + 1).fill(unreachable)
  const paths: Array<Record<string, number> | null> = Array(target + 1).fill(null)
  dp[0] = 0
  paths[0] = {}
  for (const money of denominations) {
    for (let copy = 0; copy < (inventory[money.id] || 0); copy++) {
      for (let amount = target; amount >= money.value; amount--) {
        if (dp[amount - money.value] + 1 < dp[amount]) {
          dp[amount] = dp[amount - money.value] + 1
          paths[amount] = {
            ...paths[amount - money.value],
            [money.id]: (paths[amount - money.value]?.[money.id] || 0) + 1,
          }
        }
      }
    }
  }
  return { count: dp[target], selection: paths[target] || {} }
}

function makeRound(difficulty: number): Round {
  const limits = difficulty === 1 ? { bill: 3, coin: 4 } : difficulty === 2 ? { bill: 4, coin: 5 } : { bill: 5, coin: 7 }
  let inventory: Record<string, number> = {}
  let target = 0
  let answer = { count: 9999, selection: {} as Record<string, number> }
  do {
    inventory = {}
    denominations.forEach((money) => {
      const max = money.kind === 'bill' ? limits.bill : limits.coin
      inventory[money.id] = Math.floor(Math.random() * max) + (money.id === 'penny' ? 2 : 1)
    })
    const chosen: Record<string, number> = {}
    denominations.forEach((money) => {
      const chance = difficulty === 1 ? 0.45 : 0.62
      chosen[money.id] = Math.random() < chance
        ? Math.floor(Math.random() * Math.min(inventory[money.id] ?? 0, difficulty + 1)) + 1
        : 0
    })
    target = denominations.reduce((sum, money) => sum + (chosen[money.id] ?? 0) * money.value, 0)
    answer = solveBounded(target, inventory)
  } while (target < 35 || answer.count < 3 || answer.count > (difficulty === 1 ? 7 : 11))
  return { target, inventory, optimalCount: answer.count, optimalSelection: answer.selection }
}

const round = ref<Round>(makeRound(level.value))
const selectedTotal = computed(() => denominations.reduce((sum, money) => sum + (selected.value[money.id] || 0) * money.value, 0))
const selectedCount = computed(() => Object.values(selected.value).reduce((sum, count) => sum + count, 0))
const remaining = computed(() => round.value.target - selectedTotal.value)
const progress = computed(() => Math.min((selectedTotal.value / round.value.target) * 100, 100))
const bills = computed(() => denominations.filter((item) => item.kind === 'bill'))
const coins = computed(() => denominations.filter((item) => item.kind === 'coin'))
const currentTip = computed(() => denominations[(solved.value + level.value) % denominations.length]?.fact ?? '')
const feedback = computed(() => {
  if (result.value === 'short') return `You're ${formatMoney(remaining.value)} short. Keep going!`
  if (result.value === 'over') return `That's ${formatMoney(Math.abs(remaining.value))} too much.`
  if (result.value === 'not-optimal') return `Exact amount — but it can be done with ${round.value.optimalCount} pieces.`
  if (result.value === 'correct') return `Perfect! ${selectedCount.value} pieces is the fewest possible.`
  return ''
})
const currentHint = computed(() => {
  const items = denominations.filter((money) => (round.value.optimalSelection[money.id] || 0) > 0)
  const money = items[Math.min(hintStep.value - 1, items.length - 1)]
  return money ? `Try using ${round.value.optimalSelection[money.id]} × ${money.name}.` : ''
})

function adjust(money: Money, delta: number) {
  if (result.value === 'correct') return
  const current = selected.value[money.id] || 0
  selected.value[money.id] = Math.max(0, Math.min(round.value.inventory[money.id] ?? 0, current + delta))
  result.value = 'idle'
}
function checkAnswer() {
  if (remaining.value > 0) result.value = 'short'
  else if (remaining.value < 0) result.value = 'over'
  else if (selectedCount.value > round.value.optimalCount) result.value = 'not-optimal'
  else {
    result.value = 'correct'
    score.value += 100 + Math.max(0, 3 - hintStep.value) * 25 + streak.value * 10
    streak.value += 1
    solved.value += 1
  }
}
function resetSelection() {
  selected.value = {}
  result.value = 'idle'
  hintStep.value = 0
}
function nextRound() {
  if (solved.value > 0 && solved.value % 3 === 0 && level.value < 3) level.value += 1
  round.value = makeRound(level.value)
  resetSelection()
}
function changeLevel(next: number) {
  level.value = next
  streak.value = 0
  round.value = makeRound(next)
  resetSelection()
  showLevelMenu.value = false
}
function useHint() {
  hintStep.value += 1
  result.value = 'idle'
}
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Cash Class home">
        <span class="brand-mark"><span>$</span></span>
        <span>Cash<span>Class</span></span>
      </a>
      <div class="level-picker">
        <button class="level-button" @click="showLevelMenu = !showLevelMenu">
          <span class="level-dot">{{ level }}</span>
          <span><small>LEARNING PATH</small>Level {{ level }} · {{ ['The Basics', 'Making Change', 'Cash Master'][level - 1] }}</span>
          <ChevronDown :size="16" />
        </button>
        <div v-if="showLevelMenu" class="level-menu">
          <button v-for="n in 3" :key="n" :class="{ active: level === n }" @click="changeLevel(n)">
            Level {{ n }} · {{ ['The Basics', 'Making Change', 'Cash Master'][n - 1] }}
          </button>
        </div>
      </div>
      <div class="header-actions">
        <div class="streak"><Zap :size="16" fill="currentColor" /> {{ streak }} streak</div>
        <button class="icon-button" aria-label="Open money guide" @click="showGuide = true"><CircleHelp :size="20" /></button>
        <div class="avatar">JD</div>
      </div>
    </header>

    <main>
      <section class="mission">
        <div class="mission-copy">
          <div class="eyebrow"><span></span> TODAY'S CHALLENGE</div>
          <h1>Make the exact amount.</h1>
          <p>Use the fewest bills and coins possible from your wallet.</p>
        </div>
        <div class="target-card">
          <span>YOUR TARGET</span>
          <strong>{{ formatMoney(round.target) }}</strong>
          <em><Sparkles :size="13" /> Fewest pieces wins</em>
        </div>
      </section>

      <section class="game-layout">
        <div class="wallet-panel">
          <div class="panel-heading">
            <div>
              <span class="step-number">1</span>
              <span><small>YOUR WALLET</small><strong>Choose your money</strong></span>
            </div>
            <button class="text-button" @click="resetSelection"><RotateCcw :size="15" /> Reset</button>
          </div>

          <div class="money-section">
            <h2><span>BILLS</span><i></i></h2>
            <div class="bills-grid">
              <article v-for="money in bills" :key="money.id" class="money-option">
                <button class="bill" :class="{ selected: selected[money.id] }" @click="adjust(money, 1)">
                  <span class="bill-value left">{{ money.shortName }}</span>
                  <span class="bill-seal">$</span>
                  <span class="portrait">{{ money.portrait }}</span>
                  <span class="bill-value right">{{ money.shortName }}</span>
                </button>
                <div class="money-meta">
                  <span><strong>{{ money.shortName }}</strong><small>{{ round.inventory[money.id] }} available</small></span>
                  <div class="counter">
                    <button :disabled="!selected[money.id]" @click="adjust(money, -1)"><Minus :size="13" /></button>
                    <b>{{ selected[money.id] || 0 }}</b>
                    <button :disabled="(selected[money.id] ?? 0) >= (round.inventory[money.id] ?? 0)" @click="adjust(money, 1)"><Plus :size="13" /></button>
                  </div>
                </div>
              </article>
            </div>
          </div>

          <div class="money-section coins-section">
            <h2><span>COINS</span><i></i></h2>
            <div class="coins-grid">
              <article v-for="money in coins" :key="money.id" class="money-option coin-option">
                <button class="coin" :class="[money.id, { selected: selected[money.id] }]" @click="adjust(money, 1)">
                  <span class="coin-face">{{ money.shortName }}</span>
                </button>
                <div class="coin-name"><strong>{{ money.shortName }}</strong><small>{{ money.name }}</small></div>
                <div class="counter coin-counter">
                  <button :disabled="!selected[money.id]" @click="adjust(money, -1)"><Minus :size="13" /></button>
                  <b>{{ selected[money.id] || 0 }}</b>
                  <button :disabled="(selected[money.id] ?? 0) >= (round.inventory[money.id] ?? 0)" @click="adjust(money, 1)"><Plus :size="13" /></button>
                </div>
                <small class="available">{{ round.inventory[money.id] }} available</small>
              </article>
            </div>
          </div>
        </div>

        <aside class="tray-panel">
          <div class="panel-heading compact">
            <div><span class="step-number">2</span><span><small>YOUR ANSWER</small><strong>Cash tray</strong></span></div>
          </div>
          <div class="total-display">
            <span>SELECTED TOTAL</span>
            <strong :class="{ over: remaining < 0, exact: remaining === 0 }">{{ formatMoney(selectedTotal) }}</strong>
            <div class="progress-track"><i :style="{ width: `${progress}%` }"></i></div>
            <div class="target-caption">
              <span>{{ remaining > 0 ? `${formatMoney(remaining)} to go` : remaining < 0 ? `${formatMoney(-remaining)} over` : 'Exact amount!' }}</span>
              <span>Target {{ formatMoney(round.target) }}</span>
            </div>
          </div>

          <div v-if="selectedCount" class="tray-items">
            <div v-for="money in denominations.filter((item) => selected[item.id])" :key="money.id" class="tray-row">
              <span :class="['mini-money', money.kind, money.id]">{{ money.shortName }}</span>
              <span><strong>{{ money.name }}</strong><small>{{ selected[money.id] }} × {{ money.shortName }}</small></span>
              <b>{{ formatMoney((selected[money.id] ?? 0) * money.value) }}</b>
              <button @click="adjust(money, -(selected[money.id] ?? 0))"><X :size="14" /></button>
            </div>
          </div>
          <div v-else class="empty-tray">
            <span><Plus :size="22" /></span>
            <strong>Your tray is empty</strong>
            <p>Tap bills or coins to add them here.</p>
          </div>

          <div v-if="feedback" :class="['feedback', result]">
            <span><Check v-if="result === 'correct'" :size="19" /><Lightbulb v-else :size="19" /></span>
            <p>{{ feedback }}</p>
          </div>
          <div v-if="hintStep" class="hint-box">
            <Lightbulb :size="17" /><span><small>HINT {{ hintStep }}</small>{{ currentHint }}</span>
          </div>

          <div class="tray-footer">
            <div class="piece-count">
              <span><strong>{{ selectedCount }}</strong> pieces used</span>
              <span v-if="result === 'correct'"><Trophy :size="14" /> Optimal!</span>
            </div>
            <button v-if="result !== 'correct'" class="primary-button" :disabled="!selectedCount" @click="checkAnswer">
              Check my answer <ArrowRight :size="17" />
            </button>
            <button v-else class="primary-button success" @click="nextRound">Next challenge <ArrowRight :size="17" /></button>
            <button v-if="result !== 'correct'" class="hint-button" @click="useHint"><Lightbulb :size="15" /> Need a hint?</button>
          </div>
        </aside>
      </section>

      <section class="stats-strip">
        <div><span class="stat-icon blue"><Trophy :size="18" /></span><span><small>SESSION SCORE</small><strong>{{ score.toLocaleString() }} pts</strong></span></div>
        <i></i>
        <div><span class="stat-icon amber"><Zap :size="18" /></span><span><small>CURRENT STREAK</small><strong>{{ streak }} correct</strong></span></div>
        <i></i>
        <div class="tip"><span class="stat-icon green"><Lightbulb :size="18" /></span><span><small>QUICK TIP</small><strong>{{ currentTip }}</strong></span></div>
      </section>
    </main>

    <footer><span>CashClass</span> · Learn money by doing <button @click="showGuide = true">Money guide</button></footer>

    <div v-if="showGuide" class="modal-backdrop" @click.self="showGuide = false">
      <section class="guide-modal">
        <button class="modal-close" @click="showGuide = false"><X :size="20" /></button>
        <div class="eyebrow"><span></span> MONEY GUIDE</div>
        <h2>Meet U.S. money</h2>
        <p>Remember each name and value. The size of a coin does not always tell you what it is worth.</p>
        <div class="guide-list">
          <article v-for="money in denominations" :key="money.id">
            <span :class="['guide-token', money.kind, money.id]">{{ money.shortName }}</span>
            <span><strong>{{ money.name }}</strong><small>{{ money.fact }}</small></span>
          </article>
        </div>
      </section>
    </div>
  </div>
</template>

<style src="./assets/app.css"></style>
