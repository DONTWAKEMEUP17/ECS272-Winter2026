<script setup lang="ts">
import TreemapView from './components/TreemapView.vue'
import TopArtistsBar from './components/TopArtistsBar.vue'
import BubbleChart from './components/BubbleChart.vue'
import SankeyDiagram from './components/SankeyDiagram.vue'

// Local components are available in the template automatically with `script setup`.
</script>

<!-- Using Vuetify components (Vuetify should be registered in the app entry). -->
<template>
  <VContainer id="main-container" class="d-flex flex-column" fluid style="background: #ffffff; min-height: 100vh; padding: 0; overflow-y: auto;">
    <!-- Header -->
    <VRow class="pa-0" style="background: #f5f5f5; padding: 24px 40px; border-bottom: 2px solid #1DB954; flex-shrink: 0;">
      <VCol class="pa-0" style="padding-left: 60px;">
        <h1 style="color: #1DB954; font-size: 32px; font-weight: bold; margin: 0; letter-spacing: 1px;">
          🎵 SPOTIFY TRACKS ANALYTICS
        </h1>
        <p style="color: #666; margin: 8px 0 0 0; font-size: 14px; white-space: nowrap;">
          Explore artist and track popularity patterns across 8,780 Spotify tracks
        </p>
      </VCol>
    </VRow>

    <!-- Visualizations Grid -->
    <!-- Row 1: Treemap and Bar Chart -->
    <VRow class="pa-4" style="height: 400px; background: #ffffff; flex-shrink: 0; gap: 16px;">
      <VCol class="pa-0" style="flex: 1; border: 1px solid #ddd; border-radius: 8px; overflow: hidden; min-width: 0;">
        <TreemapView />
      </VCol>
      <VCol class="pa-0" style="flex: 1; border: 1px solid #282828; border-radius: 8px; overflow: hidden; min-width: 0;">
        <TopArtistsBar />
      </VCol>
    </VRow>

    <!-- Row 2: Bubble Chart and Sankey Diagram -->
    <VRow class="pa-4" style="height: 450px; background: #ffffff; flex-shrink: 0; gap: 16px;">
      <VCol class="pa-0" style="flex: 1; border: 1px solid #ddd; border-radius: 8px; overflow: hidden; min-width: 0;">
        <BubbleChart />
      </VCol>
      <VCol class="pa-0" style="flex: 1; border: 1px solid #282828; border-radius: 8px; overflow: hidden; min-width: 0;">
        <SankeyDiagram />
      </VCol>
    </VRow>
  </VContainer>
</template>

<style scoped>
#main-container {
  background-color: #ffffff;
  overflow-y: auto;
}

h1 {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

p {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}
</style>
