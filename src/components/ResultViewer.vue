<template>
  <div v-if="!store.isSolverFinished" class="flex items-center">
    <span
      v-if="store.isSolverRunning || store.isFinalizing"
      class="spinner inline-block mr-3"
    ></span>
    {{
      !store.hasSolverRun
        ? "Solver has not run."
        : store.isSolverRunning
        ? "Solver running..."
        : store.isFinalizing
        ? "Finalizing..."
        : "Solver paused."
    }}
  </div>

  <div v-else>
    <div ref="resultNav" class="flex pl-0.5 overflow-x-auto">
      <div class="flex flex-col items-start">
        <button
          :class="
            'my-[0.1875rem] px-2 py-[0.0625rem] whitespace-nowrap border bg-white rounded-lg shadow select-none ' +
            (actionList.length === 1
              ? 'ring-1 ring-red-600 border-red-600 cursor-default'
              : 'border-black')
          "
          @click="moveResult(0, 0)"
        >
          <span class="inline-block mr-2 underline">
            {{ ["Flop", "Turn", "River"][board.length - 3] }}
          </span>
          <span
            v-for="item in board.map(cardText)"
            :key="item.rank + item.suit"
            :class="item.colorClass"
          >
            {{ item.rank + item.suit }}
          </span>
        </button>
      </div>

      <div
        v-for="item in actionList"
        :key="item.depth"
        class="flex flex-col ml-2 items-start"
      >
        <template v-if="item.type !== 'Player'">
          <button
            :class="
              'my-[0.1875rem] px-2 py-[0.0625rem] whitespace-nowrap border bg-white rounded-lg shadow select-none ' +
              (item.depth === actionList.length - 1
                ? 'ring-1 ring-red-600 border-red-600 '
                : 'border-black ') +
              (item.depth >= actionList.length - 1 ? 'cursor-default' : '')
            "
            @click="moveResult(item.depth, item.selectedIndex)"
          >
            <span class="inline-block mr-2 underline">
              {{ item.type }}
            </span>
            <span
              v-for="tmp in item.depth === actionList.length
                ? [{ rank: '?', suit: '', colorClass: 'text-black' }]
                : [cardText(item.selectedIndex)]"
              :key="tmp.rank + tmp.suit"
              :class="tmp.colorClass"
            >
              {{ tmp.rank + tmp.suit }}
            </span>
          </button>
        </template>
        <template v-else>
          <button
            v-for="action in item.actions"
            :key="action.str"
            :class="
              'w-full my-[0.1875rem] px-2 py-[0.0625rem] whitespace-nowrap border rounded-lg shadow text-center select-none ' +
              (!action.isSelected && item.depth < actionList.length
                ? 'opacity-40 '
                : '') +
              (action.isSelected && item.depth === actionList.length - 1
                ? 'ring-1 ring-red-600 border-red-600 cursor-default '
                : 'border-black ' +
                  (action.isTerminal ? 'cursor-default ' : '')) +
              (item.depth === actionList.length
                ? actionColorByStr[action.str]
                : 'bg-white')
            "
            @click="moveResult(item.depth, action.index)"
          >
            {{ action.str }}
          </button>
        </template>
      </div>
    </div>

    <div
      v-if="
        actionList.length === 0 ||
        actionList[actionList.length - 1].type === 'Player'
      "
    >
      <div class="flex mt-5 items-start">
        <div class="shrink-0">
          <table class="ml-1 bg-gray-200 shadow" @mouseleave="onMouseLeave">
            <tr v-for="row in 13" :key="row" class="h-9">
              <td
                v-for="col in 13"
                :key="col"
                :class="
                  'relative w-10 border-black select-none ' +
                  (row === col ? 'border-2' : 'border')
                "
                @mouseover="onMouseOver(row, col)"
              >
                <div
                  class="absolute bottom-0 left-0 w-full"
                  :style="{ height: weightPercent(row, col) }"
                >
                  <div
                    v-for="item in cellItems(row, col)"
                    :key="item.key"
                    :class="'absolute top-0 right-0 h-full ' + item.class"
                    :style="item.style"
                  ></div>
                </div>
                <div class="absolute -top-px left-px z-50 text-sm">
                  {{ cellText(row, col) }}
                </div>
                <div class="absolute -bottom-px right-px z-50 text-sm">
                  {{ cellAuxText(row, col) }}
                </div>
              </td>
            </tr>
          </table>

          <div class="mt-4">
            Player: {{ nodeInformation.player === 0 ? "OOP" : "IP" }} / Pot:
            {{ nodeInformation.pot }} / Stack: {{ nodeInformation.stack }}
            {{
              nodeInformation.toCall
                ? " / To Call: " + nodeInformation.toCall
                : ""
            }}
          </div>
        </div>

        <div
          class="ml-5 max-h-[30.25rem] border border-gray-500 rounded-md shadow overflow-x-auto overflow-y-scroll"
        >
          <table class="relative divide-y divide-gray-300">
            <thead class="sticky top-0 bg-gray-100 shadow">
              <tr>
                <th
                  v-for="text in headers"
                  :key="text"
                  scope="col"
                  :class="
                    'py-[0.3125rem] px-1 whitespace-nowrap text-sm font-bold cursor-pointer select-none ' +
                    (text === 'Hand'
                      ? 'min-w-[5rem]'
                      : text === 'EV'
                      ? 'min-w-[3.4rem]'
                      : 'min-w-[3.7rem]')
                  "
                  @click="sortBy(text)"
                >
                  <span
                    v-if="text === sortKey.key"
                    class="inline-block mr-0.5 text-xs"
                  >
                    {{ sortKey.order === "asc" ? "▲" : "▼" }}
                  </span>
                  {{
                    text
                      .replace("Bet", "B")
                      .replace("Raise", "R")
                      .replace("All-in", "A")
                  }}
                </th>
              </tr>
            </thead>

            <tbody class="bg-white divide-y divide-gray-300">
              <tr
                v-for="item in resultSorted"
                :key="item.card1 + '-' + item.card2"
                class="text-right text-sm"
              >
                <td class="px-2.5 py-[0.3125rem] text-center">
                  <template
                    v-for="card in [item.card1, item.card2].map(cardText)"
                    :key="card.rank + card.suit"
                  >
                    <span :class="card.colorClass">
                      {{ card.rank + card.suit }}
                    </span>
                  </template>
                </td>
                <td class="px-2.5 py-[0.3125rem]">
                  {{ percentStr(item.weight) }}
                </td>
                <td class="px-2.5 py-[0.3125rem]">
                  {{ percentStr(item.equity) }}
                </td>
                <td class="px-2.5 py-[0.3125rem]">
                  {{ trimMinusZero(item.expectedValue.toFixed(1)) }}
                </td>
                <td
                  v-for="i in item.strategy.length"
                  :key="i"
                  class="px-2.5 py-[0.3125rem]"
                >
                  {{ percentStr(item.strategy[i - 1]) }}
                </td>
              </tr>
            </tbody>

            <tfoot class="sticky bottom-0 font-bold bg-white shadow">
              <tr class="text-right text-sm">
                <th
                  scope="col"
                  class="px-2.5 py-[0.3125rem] text-center underline"
                >
                  {{ hoveredCellText }}
                </th>
                <th scope="col" class="px-2.5 py-[0.3125rem]">
                  {{ resultAverage.combos }}
                </th>
                <th scope="col" class="px-2.5 py-[0.3125rem]">
                  {{ resultAverage.equity }}
                </th>
                <th scope="col" class="px-2.5 py-[0.3125rem]">
                  {{ resultAverage.expectedValue }}
                </th>
                <th
                  v-for="i in resultAverage.strategy.length"
                  :key="i"
                  scope="col"
                  class="px-2.5 py-[0.3125rem]"
                >
                  {{ resultAverage.strategy[i - 1] }}
                </th>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>

    <div v-else class="mt-5">
      <div v-for="suit in 4" :key="suit" class="flex">
        <board-selector-card
          v-for="rank in 13"
          :key="rank"
          :class="
            'm-1 ' +
            (actionList[actionList.length - 1].actions[56 - 4 * rank - suit]
              .isTerminal
              ? 'opacity-40 cursor-default'
              : '')
          "
          :card-id="56 - 4 * rank - suit"
          @click="moveResult(actionList.length, 56 - 4 * rank - suit)"
        />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from "vue";
import { cardText, useStore, useSavedConfigStore } from "../store";
import * as GlobalWorker from "../global-worker";

import BoardSelectorCard from "./BoardSelectorCard.vue";

const ranks = ["2", "3", "4", "5", "6", "7", "8", "9", "T", "J", "Q", "K", "A"];

export default defineComponent({
  components: {
    BoardSelectorCard,
  },

  setup() {
    const store = useStore();
    const savedConfig = useSavedConfigStore();

    const resultNav = ref(null as HTMLDivElement | null);

    const isSolverFinished = ref(false);
    const handCards = ref([new Uint16Array(), new Uint16Array()]);

    const actionList = ref(
      [] as {
        type: "Player" | "Turn" | "River";
        depth: number;
        selectedIndex: number;
        actions: {
          index: number;
          str: string;
          isSelected: boolean;
          isTerminal: boolean;
        }[];
      }[]
    );

    const result = ref(
      [] as {
        card1: number;
        card2: number;
        weight: number;
        weightNormalized: number;
        equity: number;
        expectedValue: number;
        strategy: number[];
      }[]
    );

    const resultCell = ref(
      [] as {
        count: number;
        weight: number;
        equity: number;
        expectedValue: number;
        strategy: number[];
      }[]
    );

    const nodeInformation = ref(
      {} as {
        player: number;
        pot: number;
        stack: number;
        toCall: number;
      }
    );

    const hoveredCell = ref({ row: 0, col: 0 });

    type Order = "asc" | "desc";
    const sortKey = ref({ key: "Hand", order: "desc" as Order });

    store.$subscribe(async (_, store) => {
      if (store.isSolverFinished !== isSolverFinished.value) {
        if ((isSolverFinished.value = store.isSolverFinished)) {
          await updateResult(0, true);
        } else {
          await clearResult();
        }
      }
    });

    const board = computed(() => savedConfig.board);

    const dealtTurn = computed(() => {
      const item = actionList.value.find((item) => item.type === "Turn");
      return item?.selectedIndex ?? -1;
    });

    const dealtRiver = computed(() => {
      const item = actionList.value.find((item) => item.type === "River");
      return item?.selectedIndex ?? -1;
    });

    const percentStr = (num: number) => {
      if (num >= 0.9995) {
        return "100%";
      } else {
        return Math.max(100 * num, 0).toFixed(1) + "%";
      }
    };

    const trimMinusZero = (str: string) => {
      if (Number(str) === 0 && str[0] === "-") {
        return str.slice(1);
      } else {
        return str;
      }
    };

    const sortBy = (key: string) => {
      if (sortKey.value.key === key) {
        sortKey.value.order = sortKey.value.order === "asc" ? "desc" : "asc";
      } else {
        sortKey.value = { key, order: "desc" };
      }
    };

    const clearResult = async () => {
      handCards.value = [new Uint16Array(), new Uint16Array()];
      actionList.value = [];
      result.value = [];
      resultCell.value = [];
    };

    const updateResult = async (depth: number, isFirstCall: boolean) => {
      const handler = await GlobalWorker.getHandler();

      if (isFirstCall) {
        const memory = await GlobalWorker.getMemory();
        const buffer = await memory.buffer;
        for (let player = 0; player < 2; ++player) {
          const cardsBuffer = await handler.privateHandCards(player);
          handCards.value[player] = new Uint16Array(
            buffer,
            cardsBuffer.ptr,
            cardsBuffer.byteLength / 2
          );
        }
      }

      const nextActions = (await handler.getActions()).split("/");
      const numActions = nextActions.length;
      const isChance = nextActions[0] === "Chance";

      if (isChance) {
        const isPossibleChance = await handler.isPossibleChance();

        actionList.value.splice(depth, actionList.value.length, {
          type: "River",
          selectedIndex: -1,
          depth: depth + 1,
          actions: Array.from({ length: 52 }, (_, i) => ({
            index: i,
            str: i.toString(),
            isSelected: false,
            isTerminal: !isPossibleChance[i],
          })),
        });

        if (dealtTurn.value === -1 && board.value.length === 3) {
          actionList.value[actionList.value.length - 1].type = "Turn";
        }

        return;
      }

      const isTerminalAction = await handler.isTerminalAction();
      actionList.value.splice(depth, actionList.value.length, {
        type: "Player",
        selectedIndex: -1,
        depth: depth + 1,
        actions: Array.from({ length: numActions }, (_, i) => ({
          index: i,
          str: nextActions[i],
          isSelected: false,
          isTerminal: !!isTerminalAction[i],
        })).reverse(),
      });

      let player = 0;
      let pot = [0, 0];
      let lastBet = 0;

      for (let i = 0; i < actionList.value.length - 1; ++i) {
        const item = actionList.value[i];
        if (item.type !== "Player") {
          player = 0;
          lastBet = 0;
          continue;
        }

        const action = item.actions[item.selectedIndex];

        if (action.str === "Call") {
          pot[player] = pot[1 - player];
        } else if (action.str.slice(0, 3) === "Bet") {
          const bet = Number(action.str.slice(4));
          pot[player] += bet;
          lastBet = bet;
        } else if (
          action.str.slice(0, 5) === "Raise" ||
          action.str.slice(0, 6) === "All-in"
        ) {
          const bet = Number(action.str.slice(6).trimStart());
          const potDiff = pot[1 - player] - pot[player];
          pot[player] += bet - lastBet + potDiff;
          lastBet = bet;
        }

        player = 1 - player;
      }

      nodeInformation.value = {
        player,
        pot: savedConfig.startingPot + pot[0] + pot[1],
        stack: savedConfig.effectiveStack - pot[player],
        toCall: pot[1 - player] - pot[player],
      };

      const currentPlayer = await handler.currentPlayer();
      const cards = handCards.value[currentPlayer];

      const results = await handler.getResults();
      const weights = results.subarray(0, cards.length);
      const weightsNormalized = results.subarray(
        cards.length,
        2 * cards.length
      );
      const expectedValues = results.subarray(
        2 * cards.length,
        3 * cards.length
      );
      const equity = results.subarray(3 * cards.length, 4 * cards.length);
      const strategy = results.subarray(4 * cards.length);

      if (resultNav.value) {
        const div = resultNav.value;
        div.scrollLeft = div.scrollWidth - div.clientWidth;
      }

      let factor = store.normalizer;
      if (dealtTurn.value !== -1) factor *= 45;
      if (dealtRiver.value !== -1) factor *= 44;

      result.value = Array.from({ length: cards.length }, (_, i) => ({
        card1: cards[i] >> 8,
        card2: cards[i] & 0xff,
        weight: weights[i],
        weightNormalized: weightsNormalized[i],
        equity: (equity[i] / weightsNormalized[i]) * factor + 0.5,
        expectedValue:
          (expectedValues[i] / weightsNormalized[i]) * factor +
          savedConfig.startingPot / 2 +
          pot[currentPlayer],
        strategy: Array.from(
          { length: numActions },
          (_, j) => strategy[j * cards.length + i]
        ).reverse(),
      }));

      for (let i = 0; i < numActions; ++i) {
        const action = actionList.value[actionList.value.length - 1].actions[i];
        if (action.isTerminal) continue;
        let invalid = true;
        for (let j = 0; j < cards.length; ++j) {
          if (
            weightsNormalized[j] > 0 &&
            weights[j] * result.value[j].strategy[i] >= 0.05 / 100
          ) {
            invalid = false;
            break;
          }
        }
        action.isTerminal = invalid;
      }

      const weightSumCell = Array.from({ length: 13 * 13 }, () => 0);
      const weightNormalizedSumCell = Array.from({ length: 13 * 13 }, () => 0);
      const countCell = Array.from({ length: 13 * 13 }, () => 0);
      const equitySumCell = Array.from({ length: 13 * 13 }, () => 0);
      const expectedValueSumCell = Array.from({ length: 13 * 13 }, () => 0);
      const strategySumCell = Array.from({ length: 13 * 13 }, () =>
        Array.from({ length: numActions }, () => 0)
      );

      const strategySum = Array.from({ length: numActions }, () => 0);

      for (let i = 0; i < cards.length; ++i) {
        const card1 = cards[i] >> 8;
        const card2 = cards[i] & 0xff;
        const rank1 = Math.floor(card1 / 4);
        const rank2 = Math.floor(card2 / 4);
        const suit1 = card1 % 4;
        const suit2 = card2 % 4;

        let row, col;
        if (rank1 === rank2) {
          row = 12 - rank1;
          col = 12 - rank1;
        } else if (suit1 === suit2) {
          row = 12 - rank1;
          col = 12 - rank2;
        } else {
          row = 12 - rank2;
          col = 12 - rank1;
        }

        const idx = row * 13 + col;

        if (weightsNormalized[i] > 0 && weights[i] >= 0.05 / 100) {
          weightSumCell[idx] += weights[i];
          weightNormalizedSumCell[idx] += weightsNormalized[i];
          countCell[idx] += 1;
          equitySumCell[idx] += equity[i] * factor;
          expectedValueSumCell[idx] += expectedValues[i] * factor;
          for (let j = 0; j < numActions; ++j) {
            const s = weightsNormalized[i] * result.value[i].strategy[j];
            strategySumCell[idx][j] += s;
            strategySum[j] += s;
          }
        }
      }

      resultCell.value = Array.from({ length: 13 * 13 }, (_, i) => ({
        weight: weightSumCell[i],
        count: countCell[i],
        equity: equitySumCell[i] / weightNormalizedSumCell[i] + 0.5,
        expectedValue:
          expectedValueSumCell[i] / weightNormalizedSumCell[i] +
          savedConfig.startingPot / 2 +
          pot[currentPlayer],
        strategy: Array.from(
          { length: numActions },
          (_, j) => strategySumCell[i][j] / weightNormalizedSumCell[i]
        ),
      }));
    };

    const moveResult = async (depth: number, index: number) => {
      if (depth === 0) {
        if (actionList.value.length === 1) return;
      } else if (index === -1) {
        return;
      } else {
        const item = actionList.value[depth - 1];
        const selectedIndex = item.actions.findIndex((a) => a.index === index);
        if (
          item.actions[selectedIndex].isTerminal ||
          (item.selectedIndex === selectedIndex &&
            depth === actionList.value.length - 1)
        ) {
          return;
        }
        item.selectedIndex = selectedIndex;
        item.actions.forEach((a) => (a.isSelected = a.index === index));
      }

      const history = new Uint32Array(
        actionList.value
          .slice(0, depth)
          .map((item) => item.actions[item.selectedIndex].index)
      );

      if (!["Hand", "Weight", "EQ", "EV"].includes(sortKey.value.key)) {
        sortKey.value = { key: "Hand", order: "desc" };
      }

      const handler = await GlobalWorker.getHandler();
      await handler.applyHistory(history);
      await updateResult(depth, false);
    };

    const nextActionsStr = computed(() => {
      if (actionList.value.length === 0) return [];
      const lastItem = actionList.value[actionList.value.length - 1];
      if (lastItem.type === "Player") {
        return lastItem.actions.map((a) => a.str);
      } else {
        return [];
      }
    });

    const actionColor = computed(() => {
      const actions = nextActionsStr.value;
      if (actions.length === 0) return [];

      const ret = [];
      const subtract = actions[actions.length - 1] === "Check" ? 1 : 2;

      for (let i = 0; i < actions.length; ++i) {
        let color;
        if (actions[i] === "Fold") {
          color = "bg-sky-300";
        } else if (actions[i] === "Check" || actions[i] === "Call") {
          color = "bg-green-300";
        } else {
          color = ["bg-amber-300", "bg-pink-300", "bg-violet-300"][
            (actions.length - subtract - i - 1) % 3
          ];
        }
        ret.push(color);
      }

      return ret;
    });

    const actionColorByStr = computed(() => {
      const ret = {} as { [key: string]: string };
      const actions = nextActionsStr.value;
      for (let i = 0; i < actions.length; ++i) {
        ret[actions[i]] = actionColor.value[i];
      }
      return ret;
    });

    const weightPercent = (row: number, col: number) => {
      if (resultCell.value.length === 0) return "0%";

      const r1 = 13 - row;
      const r2 = 13 - col;

      let denom = 0;
      for (let s1 = 0; s1 < 4; ++s1) {
        for (let s2 = 0; s2 < 4; ++s2) {
          if (
            (row === col && s1 < s2) ||
            (row !== col && row < col === (s1 === s2))
          ) {
            const c1 = r1 * 4 + s1;
            const c2 = r2 * 4 + s2;
            if (
              savedConfig.board.indexOf(c1) === -1 &&
              dealtTurn.value !== c1 &&
              dealtRiver.value !== c1 &&
              savedConfig.board.indexOf(c2) === -1 &&
              dealtTurn.value !== c2 &&
              dealtRiver.value !== c2
            ) {
              denom += 1;
            }
          }
        }
      }

      const idx = (row - 1) * 13 + col - 1;
      return percentStr(resultCell.value[idx].weight / denom);
    };

    const cellItems = (row: number, col: number) => {
      if (resultCell.value.length === 0) return [];
      const idx = (row - 1) * 13 + col - 1;
      const strategy = resultCell.value[idx].strategy;
      return strategy.map((_, i) => {
        const left = strategy.slice(0, i).reduce((a, b) => a + b, 0);
        return {
          key: i,
          class: actionColor.value[i],
          style: {
            width: percentStr(1 - left),
            zIndex: i,
          },
        };
      });
    };

    const cellText = (row: number, col: number) => {
      const r1 = 13 - Math.min(row, col);
      const r2 = 13 - Math.max(row, col);
      return ranks[r1] + ranks[r2] + ["s", "", "o"][Math.sign(row - col) + 1];
    };

    const cellAuxText = (row: number, col: number) => {
      const idx = (row - 1) * 13 + col - 1;
      if (resultCell.value.length === 0 || resultCell.value[idx].count === 0) {
        return "";
      }
      if (sortKey.value.key === "EQ") {
        return (100 * resultCell.value[idx].equity).toFixed(0) + "%";
      } else if (sortKey.value.key === "EV") {
        const ev = resultCell.value[idx].expectedValue;
        return -0.5 <= ev && ev < 0.5
          ? "0"
          : (ev >= 0 ? "+" : "") + ev.toFixed(0);
      } else {
        const i = nextActionsStr.value.indexOf(sortKey.value.key);
        if (i !== -1) {
          return (100 * resultCell.value[idx].strategy[i]).toFixed(0) + "%";
        }
      }
      return "";
    };

    const onMouseOver = (row: number, col: number) => {
      hoveredCell.value = { row, col };
    };

    const onMouseLeave = () => {
      hoveredCell.value = { row: 0, col: 0 };
    };

    const hoveredCellText = computed(() => {
      const { row, col } = hoveredCell.value;
      const idx = (row - 1) * 13 + col - 1;
      if (
        idx < 0 ||
        resultCell.value.length === 0 ||
        resultCell.value[idx].count === 0
      ) {
        return "All";
      } else {
        return cellText(row, col);
      }
    });

    const headers = computed(() => {
      return ["Hand", "Weight", "EQ", "EV"].concat(nextActionsStr.value);
    });

    const resultFiltered = computed(() => {
      const { row, col } = hoveredCell.value;
      const idx = (row - 1) * 13 + col - 1;
      if (
        idx < 0 ||
        resultCell.value.length === 0 ||
        resultCell.value[idx].count === 0
      ) {
        return result.value.filter(
          (r) => r.weightNormalized > 0 && r.weight >= 0.05 / 100
        );
      }
      const r1 = 13 - Math.min(row, col);
      const r2 = 13 - Math.max(row, col);
      const isSuited = row < col;
      return result.value.filter((r) => {
        return (
          r.weightNormalized > 0 &&
          r.weight >= 0.05 / 100 &&
          Math.floor(r.card1 / 4) === r1 &&
          Math.floor(r.card2 / 4) === r2 &&
          (r.card1 % 4 === r.card2 % 4) === isSuited
        );
      });
    });

    const resultSorted = computed(() => {
      const rankComparator = (
        a1: number,
        a2: number,
        b1: number,
        b2: number,
        order: Order
      ) => {
        const coef = order === "asc" ? 1 : -1;
        const ar1 = Math.floor(a1 / 4);
        const as1 = a1 % 4;
        const ar2 = Math.floor(a2 / 4);
        const as2 = a2 % 4;
        const br1 = Math.floor(b1 / 4);
        const bs1 = b1 % 4;
        const br2 = Math.floor(b2 / 4);
        const bs2 = b2 % 4;
        return (
          coef * (ar1 - br1) ||
          coef * (ar2 - br2) ||
          Number(bs1 === bs2) - Number(as1 === as2) ||
          bs1 - as1 ||
          bs2 - as2
        );
      };

      const round = (x: number) => Math.round(10 * x);

      return [...resultFiltered.value].sort((a, b) => {
        const coef = sortKey.value.order === "asc" ? 1 : -1;
        if (sortKey.value.key === "Hand") {
          return rankComparator(
            a.card1,
            a.card2,
            b.card1,
            b.card2,
            sortKey.value.order
          );
        } else {
          const fallback = rankComparator(
            a.card1,
            a.card2,
            b.card1,
            b.card2,
            "desc"
          );
          if (sortKey.value.key === "Weight") {
            return (
              coef * (round(100 * a.weight) - round(100 * b.weight)) || fallback
            );
          } else if (sortKey.value.key === "EQ") {
            return (
              coef * (round(100 * a.equity) - round(100 * b.equity)) || fallback
            );
          } else if (sortKey.value.key === "EV") {
            return (
              coef * (round(a.expectedValue) - round(b.expectedValue)) ||
              fallback
            );
          } else {
            const idx = nextActionsStr.value.indexOf(sortKey.value.key);
            return (
              coef *
                (round(100 * a.strategy[idx]) - round(100 * b.strategy[idx])) ||
              fallback
            );
          }
        }
      });
    });

    const resultAverage = computed(() => {
      const result = resultFiltered.value;
      if (result.length === 0) {
        return {
          combos: "-",
          equity: "-",
          expectedValue: "-",
          strategy: nextActionsStr.value.map(() => "-"),
        };
      }
      let weightSum = 0;
      let combos = 0;
      let equity = 0;
      let expectedValue = 0;
      let strategy = nextActionsStr.value.map(() => 0);
      for (const r of result) {
        weightSum += r.weightNormalized;
        combos += r.weight;
        equity += r.equity * r.weightNormalized;
        expectedValue += r.expectedValue * r.weightNormalized;
        for (let i = 0; i < nextActionsStr.value.length; i++) {
          strategy[i] += r.strategy[i] * r.weightNormalized;
        }
      }
      return {
        combos: combos.toFixed(1),
        equity: percentStr(equity / weightSum),
        expectedValue: trimMinusZero((expectedValue / weightSum).toFixed(1)),
        strategy: strategy.map((x) => percentStr(x / weightSum)),
      };
    });

    return {
      store,
      cardText,
      resultNav,
      board,
      actionList,
      sortKey,
      sortBy,
      percentStr,
      trimMinusZero,
      moveResult,
      actionColorByStr,
      weightPercent,
      cellItems,
      cellText,
      cellAuxText,
      onMouseOver,
      onMouseLeave,
      hoveredCellText,
      headers,
      resultSorted,
      resultAverage,
      nodeInformation,
    };
  },
});
</script>
