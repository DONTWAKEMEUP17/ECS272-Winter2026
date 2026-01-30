<script setup lang="ts">
import OverviewScatter from './components/OverviewScatter.vue'
import TopArtistsBar from './components/TopArtistsBar.vue'
import BubbleChart from './components/BubbleChart.vue'

// Local components are available in the template automatically with `script setup`.
</script>

<!-- Using Vuetify components (Vuetify should be registered in the app entry). -->
<template>
  <VContainer id="main-container" class="d-flex flex-column" fluid>
    <!-- Context View: Overview of the entire dataset -->
    <VRow class="pa-0" style="height: 45%;">
      <VCol class="pa-0">
        <OverviewScatter />
      </VCol>
    </VRow>

    <!-- Focus Views: Detailed exploration of specific dimensions -->
    <VRow class="pa-0" style="height: 55%;">
      <!-- Focus 1: Top Artists by Track Count -->
      <VCol cols="6" class="pa-0">
        <TopArtistsBar />
      </VCol>
      <!-- Focus 2: Advanced Bubble Chart -->
      <VCol cols="6" class="pa-0">
        <BubbleChart />
      </VCol>
    </VRow>
  </VContainer>
</template>

<style scoped>
#main-container{
  height: 100%;
}
</style>
