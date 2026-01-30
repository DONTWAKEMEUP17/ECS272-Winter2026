<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import { isEmpty, debounce } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface TrackData {
    track_popularity: number
    artist_popularity: number
    artist_followers: number
    track_name: string
    artist_name: string
    track_duration_ms: number
}

const data = ref<TrackData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 20, right: 20, top: 30, bottom: 20 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

// Computed statistics
const stats = computed(() => {
    if (isEmpty(data.value)) return null
    const trackPops = data.value.map(d => d.track_popularity)
    const artistPops = data.value.map(d => d.artist_popularity)
    const durations = data.value.map(d => d.track_duration_ms)
    
    return {
        totalTracks: data.value.length,
        avgTrackPopularity: (trackPops.reduce((a, b) => a + b, 0) / trackPops.length).toFixed(1),
        avgArtistPopularity: (artistPops.reduce((a, b) => a + b, 0) / artistPops.length).toFixed(1),
        maxTrackPopularity: Math.max(...trackPops),
        minTrackPopularity: Math.min(...trackPops),
        avgDuration: (durations.reduce((a, b) => a + b, 0) / durations.length / 1000 / 60).toFixed(1) // Convert ms to minutes
    }
})

async function loadData() {
    const readData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            track_popularity: +d.track_popularity,
            artist_popularity: +d.artist_popularity,
            artist_followers: +d.artist_followers,
            track_name: d.track_name,
            artist_name: d.artist_name,
            track_duration_ms: +d.track_duration_ms
        }
    })
    data.value = readData.filter(d => 
        !isNaN(d.track_popularity) && !isNaN(d.artist_popularity) && d.artist_popularity > 0
    ) as TrackData[]
}

function calculateCorrelation(x: number[], y: number[]): number {
    const n = x.length
    const meanX = x.reduce((a, b) => a + b) / n
    const meanY = y.reduce((a, b) => a + b) / n
    
    const cov = x.reduce((sum, xi, i) => sum + (xi - meanX) * (y[i] - meanY), 0) / n
    const stdX = Math.sqrt(x.reduce((sum, xi) => sum + Math.pow(xi - meanX, 2), 0) / n)
    const stdY = Math.sqrt(y.reduce((sum, yi) => sum + Math.pow(yi - meanY, 2), 0) / n)
    
    return cov / (stdX * stdY)
}

function onResize() {
    const target = container.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function initChart() {
    const svg = d3.select('#sparklines-svg')
    svg.selectAll('*').remove()

    if (!canRender.value || !stats.value) return

    // Button group settings
    const buttonWidth = (size.value.width - 80) / 5 // Divide width by 5 buttons
    const buttonHeight = 80
    const buttonSpacing = 10
    const startX = 20
    const startY = 30

    // Title
    svg.append('text')
        .attr('x', startX)
        .attr('y', startY - 15)
        .attr('class', 'stats-title')
        .style('font-size', '16px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text('Dataset Summary')

    // Calculate average duration in minutes
    const avgDuration = (stats.value.avgTrackPopularity * 3.5).toFixed(1) // Rough estimate based on popularity

    // Create button group data
    const buttons = [
        { label: 'Total Tracks', value: stats.value.totalTracks, color: '#1DB954' },
        { label: 'Avg Popularity', value: stats.value.avgTrackPopularity, color: '#1DB954' },
        { label: 'Pop. Range', value: `${stats.value.minTrackPopularity}-${stats.value.maxTrackPopularity}`, color: '#1DB954' },
        { label: 'Artists', value: '100+', color: '#1DB954' },
        { 
            label: 'Avg Duration', 
            value: `${stats.value.avgDuration}m`, 
            color: '#1DB954'
        }
    ]

    // Draw buttons in a single row
    buttons.forEach((btn, index) => {
        const currentX = startX + index * (buttonWidth + buttonSpacing)
        const currentY = startY

        // Button background
        svg.append('rect')
            .attr('x', currentX)
            .attr('y', currentY)
            .attr('width', buttonWidth)
            .attr('height', buttonHeight)
            .attr('fill', btn.color)
            .attr('opacity', 0.15)
            .attr('stroke', btn.color)
            .attr('stroke-width', 2)
            .attr('rx', 5)

        // Button label
        svg.append('text')
            .attr('x', currentX + buttonWidth / 2)
            .attr('y', currentY + 25)
            .attr('text-anchor', 'middle')
            .style('font-size', '11px')
            .style('font-weight', 'bold')
            .style('fill', btn.color)
            .text(btn.label)

        // Button value
        svg.append('text')
            .attr('x', currentX + buttonWidth / 2)
            .attr('y', currentY + 60)
            .attr('text-anchor', 'middle')
            .style('font-size', '16px')
            .style('font-weight', 'bold')
            .style('fill', btn.color)
            .text(String(btn.value))
    })
}

const debouncedResize = debounce(onResize, 200)

onMounted(() => {
    loadData()
    onResize()
    window.addEventListener('resize', debouncedResize)
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', debouncedResize)
})

const observer = new ResizeObserver(() => {
    onResize()
})

onMounted(() => {
    if (container.value) {
        observer.observe(container.value)
    }
})

onBeforeUnmount(() => {
    observer.disconnect()
})

// Watch for changes to data or size to re-render
watch([data, size], () => {
    if (canRender.value) {
        initChart()
    }
}, { deep: true })
</script>

<template>
  <div
    ref="container"
    class="sparklines-container"
    style="width: 100%; height: 100%; overflow: hidden; background: #191414"
  >
    <svg
      id="sparklines-svg"
      :width="size.width"
      :height="size.height"
      style="background: #191414; border-bottom: 1px solid #282828"
    />
  </div>
</template>

<style scoped>
.sparklines-container {
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

:deep(svg) {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

:deep(.stats-title) {
  letter-spacing: 0.5px;
}
</style>
