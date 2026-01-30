<script setup lang="ts">
import TreemapView from './components/TreemapView.vue'
import TopArtistsBar from './components/TopArtistsBar.vue'
import BubbleChart from './components/BubbleChart.vue'
import SparklinesSummary from './components/SparklinesSummary.vue'
import ParallelCoordinatesPlot from './components/ParallelCoordinatesPlot.vue'

// Local components are available in the template automatically with `script setup`.
</script>

<!-- Using Vuetify components (Vuetify should be registered in the app entry). -->
<template>
  <VContainer id="main-container" class="d-flex flex-column" fluid style="background: #121212; min-height: 100vh; padding: 0; overflow-y: auto;">
    <!-- Header -->
    <VRow class="pa-0" style="background: #191414; padding: 24px 40px; border-bottom: 2px solid #1DB954; flex-shrink: 0;">
      <VCol class="pa-0" style="padding-left: 20px;">
        <h1 style="color: #1DB954; font-size: 32px; font-weight: bold; margin: 0; letter-spacing: 1px;">
          🎵 SPOTIFY TRACKS ANALYTICS
        </h1>
        <p style="color: #b3b3b3; margin: 8px 0 0 0; font-size: 14px; white-space: nowrap;">
          Explore artist and track popularity patterns across 8,780 Spotify tracks
        </p>
      </VCol>
    </VRow>

    <!-- Visualization 1: Dataset Summary -->
    <VRow class="pa-4" style="height: 250px; background: #121212; flex-shrink: 0;">
      <VCol class="pa-0" style="border: 1px solid #282828; border-radius: 8px; overflow: hidden;">
        <SparklinesSummary />
      </VCol>
    </VRow>

    <!-- Visualization 2: Treemap -->
    <VRow class="pa-4" style="height: 400px; background: #121212; flex-shrink: 0;">
      <VCol class="pa-0" style="border: 1px solid #282828; border-radius: 8px; overflow: hidden;">
        <TreemapView />
      </VCol>
    </VRow>

    <!-- Visualization 3: Top Artists Bar -->
    <VRow class="pa-4" style="height: 400px; background: #121212; flex-shrink: 0;">
      <VCol class="pa-0" style="border: 1px solid #282828; border-radius: 8px; overflow: hidden;">
        <TopArtistsBar />
      </VCol>
    </VRow>

    <!-- Visualization 4: Bubble Chart -->
    <VRow class="pa-4" style="height: 400px; background: #121212; flex-shrink: 0;">
      <VCol class="pa-0" style="border: 1px solid #282828; border-radius: 8px; overflow: hidden;">
        <BubbleChart />
      </VCol>
    </VRow>

    <!-- Visualization 5: Parallel Coordinates Plot -->
    <VRow class="pa-4" style="height: 450px; background: #121212; flex-shrink: 0;">
      <VCol class="pa-0" style="border: 1px solid #282828; border-radius: 8px; overflow: hidden;">
        <ParallelCoordinatesPlot />
      </VCol>
    </VRow>
  </VContainer>
</template>

<style scoped>
#main-container {
  background-color: #121212;
  overflow-y: auto;
}

h1 {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

p {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}
</style>
