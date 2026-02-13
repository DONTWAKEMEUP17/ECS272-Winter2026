<script setup lang="ts">
import { ref } from 'vue'
import TreemapView from './components/TreemapView.vue'
import TopArtistsBar from './components/TopArtistsBar.vue'
import SankeyDiagram from './components/SankeyDiagram.vue'

// Shared state for filtering across views
const selectedArtist = ref<string | null>(null)

const handleArtistSelect = (artistName: string) => {
    selectedArtist.value = artistName
}

const clearSelection = () => {
    selectedArtist.value = null
}

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

    <!-- Row 1: Treemap (Full Width) -->
    <VRow class="pa-4" style="height: 380px; background: #ffffff; flex-shrink: 0;">
      <VCol class="pa-0" style="border: 1px solid #ddd; border-radius: 8px; overflow: hidden;">
        <TreemapView :on-artist-select="handleArtistSelect" />
      </VCol>
    </VRow>

    <!-- Interaction Tip -->
    <VRow class="pa-2" style="background: #e8f5e9; border-left: 5px solid #1DB954; flex-shrink: 0; margin: 0 16px;">
      <VCol class="pa-0">
        <p style="margin: 0; font-size: 12px; color: #2e7d32; display: flex; align-items: center; gap: 6px; line-height: 1.3;">
          <span style="font-size: 14px;">💡</span>
          <strong>Tip:</strong> Click on any artist box in the Treemap above to filter the Sankey diagram and see that artist's track popularity flow pattern.
        </p>
      </VCol>
    </VRow>

    <!-- Selection Indicator -->
    <VRow v-if="selectedArtist" class="pa-4" style="background: #f9f9f9; border-top: 1px solid #ddd; flex-shrink: 0; justify-content: space-between; align-items: center;">
      <VCol class="pa-0">
        <p style="margin: 0; font-size: 14px; color: #333;">
          <strong>Selected Artist:</strong> <span style="color: #1DB954; font-weight: bold;">{{ selectedArtist }}</span>
        </p>
      </VCol>
      <VCol class="pa-0" style="text-align: right;">
        <button @click="clearSelection" style="padding: 8px 16px; background: #1DB954; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; font-size: 13px;">
          ✕ Clear Selection
        </button>
      </VCol>
    </VRow>

    <!-- Row 2: Top Artists Bar and Sankey Diagram -->
    <VRow class="pa-4" style="height: 400px; background: #ffffff; flex-shrink: 0; gap: 16px;">
      <VCol class="pa-0" style="flex: 1; border: 1px solid #ddd; border-radius: 8px; overflow: hidden; min-width: 0;">
        <TopArtistsBar />
      </VCol>
      <VCol class="pa-0" style="flex: 1; border: 1px solid #ddd; border-radius: 8px; overflow: hidden; min-width: 0;">
        <SankeyDiagram :selected-artist="selectedArtist" />
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
