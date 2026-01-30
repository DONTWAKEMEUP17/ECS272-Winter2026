<script setup lang="ts">
import TreemapView from './components/TreemapView.vue'
import TopArtistsBar from './components/TopArtistsBar.vue'
import BubbleChart from './components/BubbleChart.vue'
import SparklinesSummary from './components/SparklinesSummary.vue'

// Local components are available in the template automatically with `script setup`.
</script>

<!-- Using Vuetify components (Vuetify should be registered in the app entry). -->
<template>
  <VContainer id="main-container" class="d-flex flex-column" fluid>
    <!-- Visualization 1: Treemap (Context View) -->
    <VRow class="pa-0" style="height: 33%;">
      <VCol class="pa-0">
        <TreemapView />
      </VCol>
    </VRow>

    <!-- Visualization 2 & 3: Focus Views -->
    <VRow class="pa-0" style="height: 33%;">
      <VCol cols="6" class="pa-0">
        <TopArtistsBar />
      </VCol>
      <VCol cols="6" class="pa-0">
        <BubbleChart />
      </VCol>
    </VRow>

    <!-- Visualization 4: Sparklines Summary -->
    <VRow class="pa-0" style="height: 34%;">
      <VCol class="pa-0">
        <SparklinesSummary />
      </VCol>
    </VRow>
  </VContainer>
</template>

<style scoped>
#main-container{
  height: 100%;
}
</style>
