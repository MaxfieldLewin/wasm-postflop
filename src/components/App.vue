<template>
  <nav-bar />

  <div class="flex flex-row w-full mx-auto max-w-screen-xl">
    <side-bar />

    <main
      class="flex-grow my-4 px-6 py-1 overflow-y-auto"
      style="height: calc(100vh - 5.5rem)"
    >
      <div class="flex">
        <div
          :class="
            'mb-5 pl-2 pr-4 pb-1 text-xl font-bold border-l-8 border-b-2 ' +
            'border-blue-600 rounded rounded-br-none'
          "
        >
          {{
            store.mainView === "About"
              ? "About"
              : store.mainView === "OOPRange"
              ? "Range 1 (OOP)"
              : store.mainView === "IPRange"
              ? "Range 2 (IP)"
              : store.mainView === "Board"
              ? "Board"
              : store.mainView === "TreeConfig"
              ? "Tree Configuration"
              : store.mainView === "RunSolver"
              ? "Run Solver"
              : "Result"
          }}
        </div>
      </div>

      <div v-if="store.mainView === 'About'">
        <about-page />
      </div>
      <div v-show="store.mainView === 'OOPRange'">
        <range-editor :player="0" />
      </div>
      <div v-show="store.mainView === 'IPRange'">
        <range-editor :player="1" />
      </div>
      <div v-show="store.mainView === 'Board'">
        <board-selector />
      </div>
      <div v-show="store.mainView === 'TreeConfig'">
        <tree-config />
      </div>
      <div v-show="store.mainView === 'RunSolver'">
        <run-solver />
      </div>
      <div v-show="store.mainView === 'Result'">
        <result-viewer />
      </div>
    </main>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { useStore } from "../store";

import AboutPage from "./AboutPage.vue";
import BoardSelector from "./BoardSelector.vue";
import NavBar from "./NavBar.vue";
import RangeEditor from "./RangeEditor.vue";
import ResultViewer from "./ResultViewer.vue";
import RunSolver from "./RunSolver.vue";
import SideBar from "./SideBar.vue";
import TreeConfig from "./TreeConfig.vue";

export default defineComponent({
  components: {
    AboutPage,
    BoardSelector,
    NavBar,
    RangeEditor,
    ResultViewer,
    RunSolver,
    SideBar,
    TreeConfig,
  },

  setup() {
    return {
      store: useStore(),
    };
  },
});
</script>
