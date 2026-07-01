<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import {
  ArrowRight, Check, ChevronDown, CircleHelp, Hand, Lightbulb, Move,
  RotateCcw, Trophy, X, Zap,
} from '@lucide/vue'
import bill1Url from './assets/money/bill-1.jpg'
import bill1BackUrl from './assets/money/bill-1-back.jpg'
import bill5Url from './assets/money/bill-5.png'
import bill5BackUrl from './assets/money/bill-5-back.jpg'
import bill10Url from './assets/money/bill-10.jpg'
import bill10BackUrl from './assets/money/bill-10-back.jpg'
import bill20Url from './assets/money/bill-20.jpg'
import bill20BackUrl from './assets/money/bill-20-back.jpg'
import bill50Url from './assets/money/bill-50.jpg'
import bill50BackUrl from './assets/money/bill-50-back.jpg'
import bill100Url from './assets/money/bill-100.jpg'
import bill100BackUrl from './assets/money/bill-100-back.jpg'
import pennyUrl from './assets/money/penny.jpg'
import pennyBackUrl from './assets/money/penny-back.jpg'
import nickelUrl from './assets/money/nickel.jpg'
import nickelBackUrl from './assets/money/nickel-back.jpg'
import dimeUrl from './assets/money/dime.jpg'
import dimeBackUrl from './assets/money/dime-back.jpg'
import quarterUrl from './assets/money/quarter.jpg'
import quarterAlaskaUrl from './assets/money/quarter-alaska.jpg'
import quarterCaliforniaUrl from './assets/money/quarter-california.jpg'
import quarterColoradoUrl from './assets/money/quarter-colorado.jpg'
import quarterFloridaUrl from './assets/money/quarter-florida.jpg'
import quarterHawaiiUrl from './assets/money/quarter-hawaii.jpg'
import quarterNewYorkUrl from './assets/money/quarter-new-york.jpg'
import quarterPennsylvaniaUrl from './assets/money/quarter-pennsylvania.jpg'
import quarterTexasUrl from './assets/money/quarter-texas.jpg'

type MoneyBackVariant = {
  name: string
  image: string
}

type Money = {
  id: string
  name: string
  shortName: string
  value: number
  kind: 'bill' | 'coin'
  portrait: string
  fact: string
  image: string
  backImage: string
  backVariants?: MoneyBackVariant[]
  diameter?: number
}
type Round = {
  target: number
  inventory: Record<string, number>
  optimalCount: number
  optimalSelection: Record<string, number>
  layoutSeed: number
  generation?: number
}

const quarterBackVariants: MoneyBackVariant[] = [
  { name: 'California', image: quarterCaliforniaUrl },
  { name: 'New York', image: quarterNewYorkUrl },
  { name: 'Texas', image: quarterTexasUrl },
  { name: 'Florida', image: quarterFloridaUrl },
  { name: 'Hawaii', image: quarterHawaiiUrl },
  { name: 'Alaska', image: quarterAlaskaUrl },
  { name: 'Colorado', image: quarterColoradoUrl },
  { name: 'Pennsylvania', image: quarterPennsylvaniaUrl },
]

const denominations: Money[] = [
  { id: 'hundred', name: 'One-hundred-dollar bill', shortName: '$100', value: 10000, kind: 'bill', portrait: 'FRANKLIN', fact: 'Benjamin Franklin appears on the $100 bill, although he was never president.', image: bill100Url, backImage: bill100BackUrl },
  { id: 'fifty', name: 'Fifty-dollar bill', shortName: '$50', value: 5000, kind: 'bill', portrait: 'GRANT', fact: 'President Ulysses S. Grant appears on the $50 bill.', image: bill50Url, backImage: bill50BackUrl },
  { id: 'twenty', name: 'Twenty-dollar bill', shortName: '$20', value: 2000, kind: 'bill', portrait: 'JACKSON', fact: 'Andrew Jackson appears on the $20 bill.', image: bill20Url, backImage: bill20BackUrl },
  { id: 'ten', name: 'Ten-dollar bill', shortName: '$10', value: 1000, kind: 'bill', portrait: 'HAMILTON', fact: 'Alexander Hamilton appears on the $10 bill.', image: bill10Url, backImage: bill10BackUrl },
  { id: 'five', name: 'Five-dollar bill', shortName: '$5', value: 500, kind: 'bill', portrait: 'LINCOLN', fact: 'Abraham Lincoln appears on both the $5 bill and the penny.', image: bill5Url, backImage: bill5BackUrl },
  { id: 'one', name: 'One-dollar bill', shortName: '$1', value: 100, kind: 'bill', portrait: 'WASHINGTON', fact: 'The $1 bill features George Washington.', image: bill1Url, backImage: bill1BackUrl },
  { id: 'quarter', name: 'Quarter', shortName: '25¢', value: 25, kind: 'coin', portrait: 'QUARTER', fact: 'Four quarters make one dollar. State quarters can have many different reverse designs.', image: quarterUrl, backImage: quarterCaliforniaUrl, backVariants: quarterBackVariants, diameter: 24.26 },
  { id: 'dime', name: 'Dime', shortName: '10¢', value: 10, kind: 'coin', portrait: 'DIME', fact: 'The dime is worth 10¢, even though it is the smallest U.S. coin.', image: dimeUrl, backImage: dimeBackUrl, diameter: 17.91 },
  { id: 'nickel', name: 'Nickel', shortName: '5¢', value: 5, kind: 'coin', portrait: 'NICKEL', fact: 'A nickel is worth 5¢ and is larger than a dime.', image: nickelUrl, backImage: nickelBackUrl, diameter: 21.21 },
  { id: 'penny', name: 'Penny', shortName: '1¢', value: 1, kind: 'coin', portrait: 'PENNY', fact: 'One hundred pennies make one dollar.', image: pennyUrl, backImage: pennyBackUrl, diameter: 19.05 },
]

type WalletPiece = {
  key: string
  money: Money
  index: number
  x: number
  y: number
  rotation: number
}

type DragState = {
  key: string
  money: Money
  source: 'table' | 'tray'
  pointerId: number
  startX: number
  startY: number
  startCenterX: number
  startCenterY: number
  grabX: number
  grabY: number
  dx: number
  dy: number
  moved: boolean
  overTray: boolean
}

type PanState = {
  pointerId: number
  startX: number
  startY: number
  scrollLeft: number
  scrollTop: number
}

type CoinPreview = {
  piece: WalletPiece
  left: number
  top: number
}

type Result = 'idle' | 'short' | 'over' | 'correct' | 'not-optimal'
type PersistedProgress = {
  version: 1
  level: number
  score: number
  streak: number
  solved: number
  selected: Record<string, number>
  result: Result
  hintStep: number
  round: Round
  pieceOffsets: Record<string, { x: number; y: number }>
  pieceLayers: Record<string, number>
  pickedPieceKeys: Record<string, boolean>
  flippedFaces: Record<string, 'front' | 'back'>
}

const PROGRESS_STORAGE_KEY = 'cash-class-progress'

function loadProgress(): PersistedProgress | null {
  try {
    const stored = localStorage.getItem(PROGRESS_STORAGE_KEY)
    if (!stored) return null
    const progress = JSON.parse(stored) as Partial<PersistedProgress>
    if (
      progress.version !== 1
      || !Number.isInteger(progress.level)
      || progress.level! < 1
      || progress.level! > 4
      || typeof progress.score !== 'number'
      || typeof progress.streak !== 'number'
      || typeof progress.solved !== 'number'
      || !progress.round
      || typeof progress.round.target !== 'number'
      || typeof progress.round.layoutSeed !== 'number'
    ) {
      return null
    }
    return progress as PersistedProgress
  } catch {
    return null
  }
}

const savedProgress = loadProgress()
const hasStaleLimitedRound = savedProgress?.level === 4
  && savedProgress.round.generation !== 2
const level = ref(savedProgress?.level ?? 1)
const score = ref(savedProgress?.score ?? 0)
const streak = ref(savedProgress?.streak ?? 0)
const solved = ref(savedProgress?.solved ?? 0)
const selected = ref<Record<string, number>>(hasStaleLimitedRound ? {} : (savedProgress?.selected ?? {}))
const result = ref<Result>(hasStaleLimitedRound ? 'idle' : (savedProgress?.result ?? 'idle'))
const hintStep = ref(hasStaleLimitedRound ? 0 : (savedProgress?.hintStep ?? 0))
const showGuide = ref(false)
const showLevelMenu = ref(false)
const dragState = ref<DragState | null>(null)
const panState = ref<PanState | null>(null)
const pieceOffsets = ref<Record<string, { x: number; y: number }>>(hasStaleLimitedRound ? {} : (savedProgress?.pieceOffsets ?? {}))
const pieceLayers = ref<Record<string, number>>(hasStaleLimitedRound ? {} : (savedProgress?.pieceLayers ?? {}))
const pickedPieceKeys = ref<Record<string, boolean>>(hasStaleLimitedRound ? {} : (savedProgress?.pickedPieceKeys ?? {}))
const flippedFaces = ref<Record<string, 'front' | 'back'>>(hasStaleLimitedRound ? {} : (savedProgress?.flippedFaces ?? {}))
const coinPreview = ref<CoinPreview | null>(null)
const tableRef = ref<HTMLElement | null>(null)
const tableCanvasRef = ref<HTMLElement | null>(null)
const trayRef = ref<HTMLElement | null>(null)
const billDenominations = denominations.filter((item) => item.kind === 'bill')
const coinDenominations = denominations.filter((item) => item.kind === 'coin')
const levelNames = ['Quarter Basics', 'Quarter Combos', 'Real-World Cents', 'Limited Wallet']
const quarterTips = [
  'Think in quarters first: 25¢, 50¢, 75¢ — then fill the remainder.',
  'For 65¢, start with two quarters. Only 15¢ remains.',
  'A quarter plus a nickel makes 30¢ with just two coins.',
  'Three quarters make 75¢. That shortcut replaces seven dimes and a nickel.',
]
const limitedWalletTips = [
  'Start with the largest useful piece you actually have, not the one you wish you had.',
  'When a usual denomination is missing, replace it with smaller bills or coins.',
  'There may be several valid payments. In this level, exact is what matters.',
  'Count what is available before deciding how to build the amount.',
]
let nextPieceLayer = Math.max(100, ...Object.values(pieceLayers.value))

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

function randomInteger(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1)) + min
}

function makeLimitedWalletRound(): Round {
  for (let attempt = 0; attempt < 250; attempt++) {
    const target = randomInteger(5, 45) * 100 + randomInteger(25, 99)
    const inventory: Record<string, number> = {
      hundred: Math.random() < 0.08 ? 1 : 0,
      fifty: Math.random() < 0.18 ? 1 : 0,
      twenty: randomInteger(0, 3),
      ten: randomInteger(0, 4),
      five: randomInteger(0, 4),
      one: randomInteger(2, 8),
      quarter: randomInteger(1, 5),
      dime: randomInteger(1, 7),
      nickel: randomInteger(0, 4),
      penny: randomInteger(1, 9),
    }
    const availableAnswer = solveBounded(target, inventory)
    if (availableAnswer.count >= 9999 || availableAnswer.count > 16) continue

    return {
      target,
      inventory,
      optimalCount: availableAnswer.count,
      optimalSelection: availableAnswer.selection,
      layoutSeed: Math.floor(Math.random() * 1_000_000_000),
      generation: 2,
    }
  }

  const target = 1265
  const inventory: Record<string, number> = {
    hundred: 0,
    fifty: 0,
    twenty: 1,
    ten: 0,
    five: 3,
    one: 4,
    quarter: 1,
    dime: 5,
    nickel: 3,
    penny: 4,
  }
  const availableAnswer = solveBounded(target, inventory)
  return {
    target,
    inventory,
    optimalCount: availableAnswer.count,
    optimalSelection: availableAnswer.selection,
    layoutSeed: Math.floor(Math.random() * 1_000_000_000),
    generation: 2,
  }
}

function makeRound(difficulty: number): Round {
  if (difficulty === 4) return makeLimitedWalletRound()

  const billScenarios = difficulty === 1
    ? [
        { dollars: 1, bills: { one: 1 } },
        { dollars: 5, bills: { five: 1 } },
        { dollars: 10, bills: { ten: 1 } },
        { dollars: 20, bills: { twenty: 1 } },
      ]
    : difficulty === 2
      ? [
          { dollars: 6, bills: { five: 1, one: 1 } },
          { dollars: 11, bills: { ten: 1, one: 1 } },
          { dollars: 15, bills: { ten: 1, five: 1 } },
          { dollars: 21, bills: { twenty: 1, one: 1 } },
          { dollars: 25, bills: { twenty: 1, five: 1 } },
        ]
      : [
          { dollars: 7, bills: { five: 1, one: 2 } },
          { dollars: 12, bills: { ten: 1, one: 2 } },
          { dollars: 17, bills: { ten: 1, five: 1, one: 2 } },
          { dollars: 22, bills: { twenty: 1, one: 2 } },
          { dollars: 27, bills: { twenty: 1, five: 1, one: 2 } },
          { dollars: 32, bills: { twenty: 1, ten: 1, one: 2 } },
        ]
  const centPatterns = difficulty === 1
    ? [25, 50, 75]
    : difficulty === 2
      ? [30, 35, 40, 45, 55, 60, 65, 80, 85, 90, 95]
      : [26, 31, 36, 41, 46, 51, 56, 61, 66, 71, 76, 81, 86, 91, 96]
  const scenario = billScenarios[Math.floor(Math.random() * billScenarios.length)] ?? billScenarios[0]!
  const inventory: Record<string, number> = {
    hundred: 0,
    fifty: 0,
    twenty: 0,
    ten: 0,
    five: 0,
    one: 0,
    quarter: 4,
    dime: 10,
    nickel: 5,
    penny: difficulty === 3 ? 10 : 5,
  }
  Object.entries(scenario.bills).forEach(([id, count]) => {
    inventory[id] = count
  })
  const distractorCount = difficulty === 1
    ? (Math.random() < 0.35 ? 1 : 0)
    : difficulty === 2
      ? 1
      : (Math.random() < 0.5 ? 2 : 1)
  billDenominations
    .filter((money) => !(money.id in scenario.bills))
    .sort(() => Math.random() - 0.5)
    .slice(0, distractorCount)
    .forEach((money) => {
      inventory[money.id] = 1
    })
  const cents = centPatterns[Math.floor(Math.random() * centPatterns.length)] ?? 25
  const target = scenario.dollars * 100 + cents
  const answer = solveBounded(target, inventory)
  return {
    target,
    inventory,
    optimalCount: answer.count,
    optimalSelection: answer.selection,
    layoutSeed: Math.floor(Math.random() * 1_000_000_000),
  }
}

const round = ref<Round>(
  savedProgress && !hasStaleLimitedRound ? savedProgress.round : makeRound(level.value),
)
const selectedTotal = computed(() => denominations.reduce((sum, money) => sum + (selected.value[money.id] || 0) * money.value, 0))
const selectedCount = computed(() => Object.values(selected.value).reduce((sum, count) => sum + count, 0))
const remaining = computed(() => round.value.target - selectedTotal.value)
const progress = computed(() => Math.min((selectedTotal.value / round.value.target) * 100, 100))
const currentTip = computed(() => {
  const tips = level.value === 4 ? limitedWalletTips : quarterTips
  return tips[(solved.value + level.value - 1) % tips.length] ?? ''
})
const walletInstructions = computed(() => level.value === 4
  ? 'The fewest-piece combination may not be available. Make the exact amount with what you have.'
  : 'Build the dollar part quickly, then look for quarters before dimes.')
const targetLabel = computed(() => level.value === 4 ? 'LIMITED WALLET · TARGET' : 'QUARTER FOCUS · TARGET')

watch(
  [level, score, streak, solved, selected, result, hintStep, round, pieceOffsets, pieceLayers, pickedPieceKeys, flippedFaces],
  () => {
    const progress: PersistedProgress = {
      version: 1,
      level: level.value,
      score: score.value,
      streak: streak.value,
      solved: solved.value,
      selected: selected.value,
      result: result.value,
      hintStep: hintStep.value,
      round: round.value,
      pieceOffsets: pieceOffsets.value,
      pieceLayers: pieceLayers.value,
      pickedPieceKeys: pickedPieceKeys.value,
      flippedFaces: flippedFaces.value,
    }
    try {
      localStorage.setItem(PROGRESS_STORAGE_KEY, JSON.stringify(progress))
    } catch {
      // The game remains usable when storage is disabled or full.
    }
  },
  { deep: true },
)

function seededNumber(seedText: string) {
  let hash = 2166136261
  for (const character of seedText) {
    hash ^= character.charCodeAt(0)
    hash = Math.imul(hash, 16777619)
  }

  // FNV alone leaves similar seeds (such as the x/y coordinate suffixes)
  // correlated. Avalanche the result so each coordinate varies independently.
  hash ^= hash >>> 16
  hash = Math.imul(hash, 0x85ebca6b)
  hash ^= hash >>> 13
  hash = Math.imul(hash, 0xc2b2ae35)
  hash ^= hash >>> 16

  return (hash >>> 0) / 4294967296
}

function seededFace(key: string) {
  const seed = seededNumber(`${round.value.layoutSeed}-${level.value}-${key}-face`)
  if (level.value === 1) return seed < 0.25 ? 'front' : 'back'
  return seed < 0.5 ? 'front' : 'back'
}

function pieceFace(piece: WalletPiece) {
  return flippedFaces.value[piece.key] ?? seededFace(piece.key)
}

function pieceBackVariant(piece: WalletPiece) {
  const variants = piece.money.backVariants
  if (!variants?.length) return null
  const offset = Math.floor(seededNumber(`${round.value.layoutSeed}-${piece.money.id}-variant-offset`) * variants.length)
  return variants[(offset + piece.index) % variants.length] ?? null
}

function pieceImage(piece: WalletPiece) {
  if (pieceFace(piece) === 'front') return piece.money.image
  return pieceBackVariant(piece)?.image ?? piece.money.backImage
}

function pieceSideLabel(piece: WalletPiece) {
  if (pieceFace(piece) === 'front') return 'Front side'
  const variant = pieceBackVariant(piece)
  return variant ? `${variant.name} reverse` : 'Reverse side'
}

function flipPiece(key: string) {
  const currentFace = flippedFaces.value[key] ?? seededFace(key)
  flippedFaces.value[key] = currentFace === 'front' ? 'back' : 'front'
}

const billAnchors = [
  { x: 260, y: 220 },
  { x: 500, y: 260 },
  { x: 300, y: 490 },
  { x: 520, y: 520 },
  { x: 280, y: 730 },
  { x: 520, y: 750 },
]

function makeWalletPiece(money: Money, index: number): WalletPiece {
  const isBill = money.kind === 'bill'
  const groupIndex = isBill
    ? billDenominations
        .filter((item) => (round.value.inventory[item.id] ?? 0) > 0)
        .findIndex((item) => item.id === money.id)
    : coinDenominations.findIndex((item) => item.id === money.id)
  const billAnchor = billAnchors[groupIndex] ?? { x: 300, y: 200 }
  const positionSeed = `${round.value.layoutSeed}-${money.id}-${index}`
  return {
    key: `${money.id}-${index}`,
    money,
    index,
    x: isBill
      ? billAnchor.x + index * 13
      : 150 + Math.round(seededNumber(`${positionSeed}-x`) * 700),
    y: isBill
      ? billAnchor.y + index * 9
      : 150 + Math.round(seededNumber(`${positionSeed}-y`) * 760),
    rotation: Math.round(seededNumber(`${positionSeed}-rotation`) * 16) - 8,
  }
}

const walletPieces = computed<WalletPiece[]>(() => {
  const pieces: WalletPiece[] = []
  denominations.forEach((money) => {
    const inventoryCount = round.value.inventory[money.id] ?? 0
    for (let index = 0; index < inventoryCount; index++) {
      const key = `${money.id}-${index}`
      if (pickedPieceKeys.value[key]) continue
      pieces.push(makeWalletPiece(money, index))
    }
  })
  return pieces
})
const selectedPieces = computed<WalletPiece[]>(() => {
  const pieces: WalletPiece[] = []
  const selectedBillCount = billDenominations.reduce(
    (sum, money) => sum + (selected.value[money.id] ?? 0),
    0,
  )
  const billRows = Math.ceil(selectedBillCount / 2)
  const billRowGap = billRows > 1 ? Math.min(52, 155 / (billRows - 1)) : 0
  const lastBillY = 185 + Math.max(0, billRows - 1) * billRowGap
  const coinStartY = billRows ? lastBillY + 70 : 185
  let selectedBillIndex = 0
  let selectedCoinIndex = 0
  denominations.forEach((money) => {
    for (let index = 0; index < (round.value.inventory[money.id] ?? 0); index++) {
      const key = `${money.id}-${index}`
      if (!pickedPieceKeys.value[key]) continue
      const displayIndex = money.kind === 'bill' ? selectedBillIndex++ : selectedCoinIndex++
      pieces.push({
        key,
        money,
        index,
        x: money.kind === 'bill' ? 105 + (displayIndex % 2) * 130 : 30 + (displayIndex % 8) * 42,
        y: money.kind === 'bill'
          ? 185 + Math.floor(displayIndex / 2) * billRowGap
          : coinStartY + Math.floor(displayIndex / 8) * 42,
        rotation: ((displayIndex * 5) % 9) - 4,
      })
    }
  })
  return pieces
})
const feedback = computed(() => {
  if (result.value === 'short') return `You're ${formatMoney(remaining.value)} short. Keep going!`
  if (result.value === 'over') return `That's ${formatMoney(Math.abs(remaining.value))} too much.`
  if (result.value === 'not-optimal') {
    const selectedQuarters = selected.value.quarter ?? 0
    const optimalQuarters = round.value.optimalSelection.quarter ?? 0
    if (selectedQuarters < optimalQuarters) {
      return `Exact amount — now replace some dimes with quarters. The best answer uses ${round.value.optimalCount} pieces.`
    }
    return `Exact amount — but it can be done with ${round.value.optimalCount} pieces.`
  }
  if (result.value === 'correct') {
    return level.value === 4
      ? `Exact payment! In this level, the correct total matters more than the piece count.`
      : `Perfect! ${selectedCount.value} pieces is the fewest possible.`
  }
  return ''
})
const currentHint = computed(() => {
  const hintPriority = ['quarter', 'dime', 'nickel', 'penny', 'hundred', 'fifty', 'twenty', 'ten', 'five', 'one']
  const items = denominations
    .filter((money) => (round.value.optimalSelection[money.id] || 0) > 0)
    .sort((a, b) => hintPriority.indexOf(a.id) - hintPriority.indexOf(b.id))
  const money = items[Math.min(hintStep.value - 1, items.length - 1)]
  return money ? `Try using ${round.value.optimalSelection[money.id]} × ${money.name}.` : ''
})

function adjust(money: Money, delta: number) {
  if (result.value === 'correct') return
  const current = selected.value[money.id] || 0
  const target = Math.max(0, Math.min(round.value.inventory[money.id] ?? 0, current + delta))
  if (target > current) {
    for (let index = 0; index < (round.value.inventory[money.id] ?? 0) && selected.value[money.id] !== target; index++) {
      const key = `${money.id}-${index}`
      if (!pickedPieceKeys.value[key]) {
        pickedPieceKeys.value[key] = true
        selected.value[money.id] = (selected.value[money.id] ?? 0) + 1
      }
    }
  } else if (target < current) {
    for (let index = (round.value.inventory[money.id] ?? 0) - 1; index >= 0 && selected.value[money.id] !== target; index--) {
      const key = `${money.id}-${index}`
      if (pickedPieceKeys.value[key]) {
        delete pickedPieceKeys.value[key]
        selected.value[money.id] = (selected.value[money.id] ?? 0) - 1
      }
    }
  }
  result.value = 'idle'
}

function pickPiece(money: Money, key: string) {
  if (pickedPieceKeys.value[key]) return
  pickedPieceKeys.value[key] = true
  selected.value[money.id] = (selected.value[money.id] ?? 0) + 1
  result.value = 'idle'
}

function bringPieceToFront(key: string) {
  if (nextPieceLayer >= 250) {
    const orderedKeys = Object.entries(pieceLayers.value)
      .sort(([, layerA], [, layerB]) => layerA - layerB)
      .map(([pieceKey]) => pieceKey)
    pieceLayers.value = Object.fromEntries(orderedKeys.map((pieceKey, index) => [pieceKey, index + 100]))
    nextPieceLayer = 100 + orderedKeys.length
  }
  pieceLayers.value[key] = ++nextPieceLayer
}

function returnPiece(money: Money, key: string) {
  if (!pickedPieceKeys.value[key]) return
  delete pickedPieceKeys.value[key]
  selected.value[money.id] = Math.max(0, (selected.value[money.id] ?? 0) - 1)
  result.value = 'idle'
}
function checkAnswer() {
  if (remaining.value > 0) result.value = 'short'
  else if (remaining.value < 0) result.value = 'over'
  else if (level.value !== 4 && selectedCount.value > round.value.optimalCount) result.value = 'not-optimal'
  else {
    result.value = 'correct'
    score.value += 100 + Math.max(0, 3 - hintStep.value) * 25 + streak.value * 10
    streak.value += 1
    solved.value += 1
  }
}
function resetSelection() {
  selected.value = {}
  pickedPieceKeys.value = {}
  pieceOffsets.value = {}
  pieceLayers.value = {}
  nextPieceLayer = 100
  flippedFaces.value = {}
  result.value = 'idle'
  hintStep.value = 0
}
function resetProgress() {
  if (!window.confirm('Reset all learning progress and start over?')) return
  try {
    localStorage.removeItem(PROGRESS_STORAGE_KEY)
  } catch {
    // Reset the in-memory state even when storage is unavailable.
  }
  level.value = 1
  score.value = 0
  streak.value = 0
  solved.value = 0
  round.value = makeRound(1)
  resetSelection()
}
function nextRound() {
  if (solved.value > 0 && solved.value % 3 === 0 && level.value < 4) level.value += 1
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

function isPointInside(element: HTMLElement | null, x: number, y: number) {
  if (!element) return false
  const rect = element.getBoundingClientRect()
  return x >= rect.left && x <= rect.right && y >= rect.top && y <= rect.bottom
}

function beginDrag(event: PointerEvent, money: Money, key: string, source: 'table' | 'tray') {
  if (result.value === 'correct' || event.button !== 0) return
  coinPreview.value = null
  const target = event.currentTarget as HTMLElement
  const targetRect = target.getBoundingClientRect()
  target.setPointerCapture(event.pointerId)
  dragState.value = {
    key,
    money,
    source,
    pointerId: event.pointerId,
    startX: event.clientX,
    startY: event.clientY,
    startCenterX: targetRect.left + targetRect.width / 2,
    startCenterY: targetRect.top + targetRect.height / 2,
    grabX: Math.max(0, Math.min(target.offsetWidth, event.offsetX)),
    grabY: Math.max(0, Math.min(target.offsetHeight, event.offsetY)),
    dx: 0,
    dy: 0,
    moved: false,
    overTray: source === 'tray',
  }
}

function moveDrag(event: PointerEvent) {
  if (!dragState.value || dragState.value.pointerId !== event.pointerId) return
  const dx = event.clientX - dragState.value.startX
  const dy = event.clientY - dragState.value.startY
  dragState.value.dx = dx
  dragState.value.dy = dy
  dragState.value.moved = dragState.value.moved || Math.hypot(dx, dy) > 7
  dragState.value.overTray = isPointInside(trayRef.value, event.clientX, event.clientY)
}

function showCoinPreview(event: PointerEvent, piece: WalletPiece) {
  if (piece.money.kind !== 'coin' || event.pointerType === 'touch' || dragState.value) return
  const previewWidth = 224
  const previewHeight = 264
  coinPreview.value = {
    piece,
    left: Math.max(12, Math.min(event.clientX + 24, window.innerWidth - previewWidth - 12)),
    top: Math.max(12, Math.min(event.clientY - previewHeight / 2, window.innerHeight - previewHeight - 12)),
  }
}

function moveMoneyPointer(event: PointerEvent, piece: WalletPiece) {
  if (dragState.value) moveDrag(event)
  else showCoinPreview(event, piece)
}

function hideCoinPreview(key: string) {
  if (coinPreview.value?.piece.key === key) coinPreview.value = null
}

function endDrag(event: PointerEvent) {
  const drag = dragState.value
  if (!drag || drag.pointerId !== event.pointerId) return
  const target = event.currentTarget as HTMLElement
  if (target.hasPointerCapture(event.pointerId)) target.releasePointerCapture(event.pointerId)

  if (drag.source === 'table') {
    if (!drag.moved) {
      bringPieceToFront(drag.key)
    } else if (isPointInside(trayRef.value, event.clientX, event.clientY)) {
      pickPiece(drag.money, drag.key)
    } else {
      const currentOffset = pieceOffsets.value[drag.key] ?? { x: 0, y: 0 }
      pieceOffsets.value[drag.key] = {
        x: currentOffset.x + drag.dx,
        y: currentOffset.y + drag.dy,
      }
      bringPieceToFront(drag.key)
    }
  } else if (!drag.moved) {
    returnPiece(drag.money, drag.key)
  } else if (
    !isPointInside(trayRef.value, event.clientX, event.clientY) &&
    isPointInside(tableRef.value, event.clientX, event.clientY) &&
    tableCanvasRef.value
  ) {
    const canvasRect = tableCanvasRef.value.getBoundingClientRect()
    const pieceIndex = Number(drag.key.slice(drag.key.lastIndexOf('-') + 1))
    const basePiece = makeWalletPiece(drag.money, pieceIndex)
    const canvasX = drag.startCenterX + drag.dx - canvasRect.left
    const canvasY = drag.startCenterY + drag.dy - canvasRect.top
    pieceOffsets.value[drag.key] = {
      x: canvasX - basePiece.x,
      y: canvasY - basePiece.y,
    }
    bringPieceToFront(drag.key)
    returnPiece(drag.money, drag.key)
  }
  dragState.value = null
}

function cancelDrag() {
  dragState.value = null
}

function pieceStyle(piece: WalletPiece) {
  const drag = dragState.value?.key === piece.key ? dragState.value : null
  const offset = pieceOffsets.value[piece.key] ?? { x: 0, y: 0 }
  const coinScale = piece.money.kind === 'coin' ? (piece.money.diameter ?? 24.26) / 24.26 : 1
  return {
    left: `${piece.x + offset.x}px`,
    top: `${piece.y + offset.y}px`,
    '--piece-x': `${piece.x + offset.x}px`,
    '--piece-y': `${piece.y + offset.y}px`,
    '--piece-rotation': `${piece.rotation}deg`,
    '--drag-x': `${drag?.dx ?? 0}px`,
    '--drag-y': `${drag?.dy ?? 0}px`,
    '--grab-x': drag ? `${drag.grabX}px` : '50%',
    '--grab-y': drag ? `${drag.grabY}px` : '50%',
    '--coin-scale': coinScale,
    '--piece-z': drag ? 280 : (pieceLayers.value[piece.key] ?? piece.index + 1),
  }
}

function selectedPieceStyle(piece: WalletPiece) {
  const drag = dragState.value?.key === piece.key ? dragState.value : null
  const coinScale = piece.money.kind === 'coin' ? (piece.money.diameter ?? 24.26) / 24.26 : 1
  return {
    left: `${piece.x}px`,
    top: `${piece.y}px`,
    '--tray-x': `${piece.x}px`,
    '--tray-y': `${piece.y}px`,
    '--piece-rotation': `${piece.rotation}deg`,
    '--drag-x': `${drag?.dx ?? 0}px`,
    '--drag-y': `${drag?.dy ?? 0}px`,
    '--grab-x': drag ? `${drag.grabX}px` : '50%',
    '--grab-y': drag ? `${drag.grabY}px` : '50%',
    '--coin-scale': coinScale,
    '--piece-z': drag ? 200 : piece.index + 1,
  }
}

function beginPan(event: PointerEvent) {
  if (event.button !== 0 || !tableRef.value) return
  const target = event.currentTarget as HTMLElement
  target.setPointerCapture(event.pointerId)
  panState.value = {
    pointerId: event.pointerId,
    startX: event.clientX,
    startY: event.clientY,
    scrollLeft: tableRef.value.scrollLeft,
    scrollTop: tableRef.value.scrollTop,
  }
}

function movePan(event: PointerEvent) {
  if (!panState.value || !tableRef.value || panState.value.pointerId !== event.pointerId) return
  tableRef.value.scrollLeft = panState.value.scrollLeft - (event.clientX - panState.value.startX)
  tableRef.value.scrollTop = panState.value.scrollTop - (event.clientY - panState.value.startY)
}

function endPan(event: PointerEvent) {
  if (!panState.value || panState.value.pointerId !== event.pointerId) return
  const target = event.currentTarget as HTMLElement
  if (target.hasPointerCapture(event.pointerId)) target.releasePointerCapture(event.pointerId)
  panState.value = null
}
</script>

<template>
  <div class="app-shell" :class="{ expert: level >= 3 }">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Cash Class home">
        <span class="brand-mark"><span>$</span></span>
        <span>Cash<span>Class</span></span>
      </a>
      <div class="level-picker">
        <button class="level-button" @click="showLevelMenu = !showLevelMenu">
          <span class="level-dot">{{ level }}</span>
          <span><small>LEARNING PATH</small>Level {{ level }} · {{ levelNames[level - 1] }}</span>
          <ChevronDown :size="16" />
        </button>
        <div v-if="showLevelMenu" class="level-menu">
          <button v-for="n in 4" :key="n" :class="{ active: level === n }" @click="changeLevel(n)">
            Level {{ n }} · {{ levelNames[n - 1] }}
          </button>
        </div>
      </div>
      <div class="header-actions">
        <div class="streak"><Zap :size="16" fill="currentColor" /> {{ streak }} streak</div>
        <button class="reset-progress-button" aria-label="Reset all progress" title="Reset all progress" @click="resetProgress">
          <RotateCcw :size="16" /><span>Reset progress</span>
        </button>
        <button class="icon-button" aria-label="Open money guide" @click="showGuide = true"><CircleHelp :size="20" /></button>
      </div>
    </header>

    <main>
      <section class="game-layout">
        <div class="wallet-panel">
          <div class="panel-heading">
            <div>
              <span>
                <strong>Choose your money</strong>
                <p>{{ walletInstructions }}</p>
              </span>
            </div>
            <button class="text-button" @click="resetSelection"><RotateCcw :size="15" /> Reset</button>
          </div>

          <div class="canvas-help">
            <span><Hand :size="16" /><strong>Click</strong> to bring forward · <strong>Double-click</strong> to pay</span>
            <span><Hand :size="15" /><strong>Drag money</strong> to move it or drop it in the payment area</span>
            <span><Move :size="15" /><strong>Drag the wood</strong> to move around the table</span>
            <span><RotateCcw :size="14" /><strong>Right-click</strong> a piece to flip it</span>
          </div>
          <div class="table-stage">
            <div ref="tableRef" class="money-table" :class="{ panning: panState }">
              <div
                ref="tableCanvasRef"
                class="table-canvas"
                @pointerdown.prevent="beginPan"
                @pointermove.prevent="movePan"
                @pointerup.prevent="endPan"
                @pointercancel="endPan"
              >
              <div class="table-grain"></div>

              <button
                v-for="piece in walletPieces"
                :key="piece.key"
                :class="[
                  'physical-money',
                  `kind-${piece.money.kind}`,
                  piece.money.id,
                  {
                    dragging: dragState?.key === piece.key,
                    'over-tray': dragState?.key === piece.key && dragState.overTray,
                    'showing-back': pieceFace(piece) === 'back',
                  },
                ]"
                :style="pieceStyle(piece)"
                :aria-label="`Pick up one ${piece.money.name}`"
                @pointerenter="showCoinPreview($event, piece)"
                @pointerdown.stop.prevent="beginDrag($event, piece.money, piece.key, 'table')"
                @pointermove.stop.prevent="moveMoneyPointer($event, piece)"
                @pointerleave="hideCoinPreview(piece.key)"
                @pointerup.stop.prevent="endDrag"
                @pointercancel.stop="cancelDrag"
                @dblclick.stop.prevent="pickPiece(piece.money, piece.key)"
                @contextmenu.stop.prevent="flipPiece(piece.key)"
                @keydown.enter.prevent="adjust(piece.money, 1)"
                @keydown.space.prevent="adjust(piece.money, 1)"
              >
                <img :src="pieceImage(piece)" :alt="`${piece.money.name}, ${pieceFace(piece)} side`" draggable="false">
                <span class="money-value-badge">{{ piece.money.shortName }}</span>
              </button>

                <div v-if="!walletPieces.length" class="table-empty">
                  <Check :size="22" /><strong>Everything is in the payment area</strong>
                </div>
              </div>
            </div>

            <section
              ref="trayRef"
              class="cash-zone"
              :class="{ 'drop-ready': dragState?.source === 'table' && dragState.overTray }"
            >
              <div class="cash-zone-heading">
                <span class="payment-label"><small>PAYMENT AREA</small><strong>Drop money here</strong></span>
                <span class="target-pill"><small>{{ targetLabel }}</small><strong>{{ formatMoney(round.target) }}</strong></span>
              </div>
              <div class="cash-zone-total">
                <span>SELECTED</span>
                <strong :class="{ over: remaining < 0, exact: remaining === 0 }">{{ formatMoney(selectedTotal) }}</strong>
                <div class="progress-track"><i :style="{ width: `${progress}%` }"></i></div>
                <small>{{ remaining > 0 ? `${formatMoney(remaining)} to go` : remaining < 0 ? `${formatMoney(-remaining)} over` : 'Exact amount!' }}</small>
              </div>

              <div v-if="selectedCount" class="cash-zone-pieces">
                <button
                  v-for="piece in selectedPieces"
                  :key="piece.key"
                  :class="[
                    'selected-money',
                    `kind-${piece.money.kind}`,
                    piece.money.id,
                    {
                      dragging: dragState?.key === piece.key,
                      'outside-tray': dragState?.key === piece.key && !dragState.overTray,
                      'showing-back': pieceFace(piece) === 'back',
                    },
                  ]"
                  :style="selectedPieceStyle(piece)"
                  :aria-label="`Return one ${piece.money.name} to the table`"
                  @pointerdown.stop.prevent="beginDrag($event, piece.money, piece.key, 'tray')"
                  @pointermove.stop.prevent="moveDrag"
                  @pointerup.stop.prevent="endDrag"
                  @pointercancel.stop="cancelDrag"
                  @contextmenu.stop.prevent="flipPiece(piece.key)"
                  @keydown.enter.prevent="returnPiece(piece.money, piece.key)"
                  @keydown.space.prevent="returnPiece(piece.money, piece.key)"
                >
                  <img :src="pieceImage(piece)" alt="" draggable="false">
                  <span>{{ piece.money.shortName }}</span>
                </button>
              </div>
              <div v-else class="cash-zone-empty">
                <Hand :size="25" />
                <strong>Place money inside this area</strong>
                <small>Drop a piece here or tap it on the table.</small>
              </div>

              <div v-if="feedback" :class="['feedback', result]">
                <span><Check v-if="result === 'correct'" :size="19" /><Lightbulb v-else :size="19" /></span>
                <p>{{ feedback }}</p>
              </div>
              <div v-if="hintStep" class="hint-box">
                <Lightbulb :size="17" /><span><small>HINT {{ hintStep }}</small>{{ currentHint }}</span>
              </div>

              <div class="cash-zone-footer">
                <div class="piece-count">
                  <span><strong>{{ selectedCount }}</strong> pieces used</span>
                  <span v-if="result === 'correct'"><Trophy :size="14" /> {{ level === 4 ? 'Exact!' : 'Optimal!' }}</span>
                </div>
                <button v-if="result !== 'correct'" class="primary-button" :disabled="!selectedCount" @click="checkAnswer">
                  Check my answer <ArrowRight :size="17" />
                </button>
                <button v-else class="primary-button success" @click="nextRound">Next challenge <ArrowRight :size="17" /></button>
                <button v-if="result !== 'correct'" class="hint-button" @click="useHint"><Lightbulb :size="15" /> Need a hint?</button>
              </div>
            </section>
          </div>
        </div>
      </section>

      <section class="stats-strip">
        <div><span class="stat-icon blue"><Trophy :size="18" /></span><span><small>SESSION SCORE</small><strong>{{ score.toLocaleString() }} pts</strong></span></div>
        <i></i>
        <div><span class="stat-icon amber"><Zap :size="18" /></span><span><small>CURRENT STREAK</small><strong>{{ streak }} correct</strong></span></div>
        <i></i>
        <div class="tip"><span class="stat-icon green"><Lightbulb :size="18" /></span><span><small>QUICK TIP</small><strong>{{ currentTip }}</strong></span></div>
      </section>
    </main>

    <footer>
      <span>CashClass</span> · Learn money by doing
      <button @click="showGuide = true">Money guide</button>
      <small>Bill imagery: U.S. Currency Education Program · Coin imagery: U.S. Mint</small>
    </footer>

    <div
      v-if="coinPreview"
      class="coin-preview"
      :style="{ left: `${coinPreview.left}px`, top: `${coinPreview.top}px` }"
      aria-hidden="true"
    >
      <div class="coin-preview-image">
        <img :src="pieceImage(coinPreview.piece)" alt="" draggable="false">
      </div>
      <span v-if="level < 3">
        <strong>{{ coinPreview.piece.money.name }}</strong>
        <small>{{ pieceSideLabel(coinPreview.piece) }} · magnified view</small>
      </span>
    </div>

    <div v-if="showGuide" class="modal-backdrop" @click.self="showGuide = false">
      <section class="guide-modal">
        <button class="modal-close" @click="showGuide = false"><X :size="20" /></button>
        <div class="eyebrow"><span></span> MONEY GUIDE</div>
        <h2>Meet U.S. money</h2>
        <p>Remember each name and value. The size of a coin does not always tell you what it is worth.</p>
        <div class="guide-list">
          <article v-for="money in denominations" :key="money.id">
            <span :class="['guide-token', money.kind, money.id]"><img :src="money.image" alt="" draggable="false"></span>
            <span><strong>{{ money.name }}</strong><small>{{ money.fact }}</small></span>
          </article>
        </div>
      </section>
    </div>
  </div>
</template>

<style src="./assets/app.css"></style>
