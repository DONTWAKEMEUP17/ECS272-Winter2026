<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import { isEmpty, debounce } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface TrackData {
    track_popularity: number
    artist_popularity: number
    artist_followers: number
    track_name: string
    artist_name: string
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
    
    return {
        totalTracks: data.value.length,
        avgTrackPopularity: (trackPops.reduce((a, b) => a + b, 0) / trackPops.length).toFixed(1),
        avgArtistPopularity: (artistPops.reduce((a, b) => a + b, 0) / artistPops.length).toFixed(1),
        maxTrackPopularity: Math.max(...trackPops),
        minTrackPopularity: Math.min(...trackPops),
        correlation: calculateCorrelation(artistPops, trackPops).toFixed(2)
    }
})

async function loadData() {
    const readData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            track_popularity: +d.track_popularity,
            artist_popularity: +d.artist_popularity,
            artist_followers: +d.artist_followers,
            track_name: d.track_name,
            artist_name: d.artist_name
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

    const sparklineWidth = (size.value.width - 40) / 2
    const sparklineHeight = 60

    // Sparkline 1: Track Popularity Distribution
    drawSparkline(
        svg,
        data.value.map(d => d.track_popularity),
        20,
        20,
        sparklineWidth,
        sparklineHeight,
        'Track Popularity Distribution',
        '#3498DB'
    )

    // Sparkline 2: Artist Popularity Distribution
    drawSparkline(
        svg,
        data.value.map(d => d.artist_popularity),
        sparklineWidth + 40,
        20,
        sparklineWidth,
        sparklineHeight,
        'Artist Popularity Distribution',
        '#E74C3C'
    )

    // Summary Stats
    const statsY = 100
    const statsX = 20
    const lineHeight = 28

    // Title
    svg.append('text')
        .attr('x', statsX)
        .attr('y', statsY)
        .attr('class', 'stats-title')
        .style('font-size', '16px')
        .style('font-weight', 'bold')
        .text('Dataset Summary')

    // Stats
    svg.append('text')
        .attr('x', statsX)
        .attr('y', statsY + lineHeight)
        .style('font-size', '12px')
        .text(`Total Tracks: ${stats.value.totalTracks}`)

    svg.append('text')
        .attr('x', statsX)
        .attr('y', statsY + lineHeight * 2)
        .style('font-size', '12px')
        .text(`Avg Track Popularity: ${stats.value.avgTrackPopularity}`)

    svg.append('text')
        .attr('x', statsX)
        .attr('y', statsY + lineHeight * 3)
        .style('font-size', '12px')
        .text(`Track Range: ${stats.value.minTrackPopularity} - ${stats.value.maxTrackPopularity}`)

    svg.append('text')
        .attr('x', statsX)
        .attr('y', statsY + lineHeight * 4)
        .style('font-size', '12px')
        .text(`Avg Artist Popularity: ${stats.value.avgArtistPopularity}`)

    svg.append('text')
        .attr('x', statsX)
        .attr('y', statsY + lineHeight * 5)
        .style('font-size', '12px')
        .style('font-weight', 'bold')
        .style('fill', stats.value.correlation > 0.3 ? '#27AE60' : '#E67E22')
        .text(`Correlation: ${stats.value.correlation} ${stats.value.correlation > 0.3 ? '↑' : '→'}`)
}

function drawSparkline(
    svg: any,
    values: number[],
    x: number,
    y: number,
    width: number,
    height: number,
    label: string,
    color: string
) {
    // Sort values for distribution visualization
    const sortedValues = [...values].sort((a, b) => a - b)
    
    const xScale = d3.scaleLinear()
        .domain([0, sortedValues.length - 1])
        .range([0, width])

    const yScale = d3.scaleLinear()
        .domain([Math.min(...sortedValues), Math.max(...sortedValues)])
        .range([height, 0])

    const line = d3.line<number>()
        .x((d, i) => xScale(i))
        .y(d => yScale(d))

    // Draw label
    svg.append('text')
        .attr('x', x)
        .attr('y', y - 5)
        .style('font-size', '11px')
        .style('font-weight', 'bold')
        .text(label)

    // Create group for sparkline
    const g = svg.append('g')
        .attr('transform', `translate(${x}, ${y})`)

    // Draw line
    g.append('path')
        .datum(sortedValues)
        .attr('d', line)
        .attr('stroke', color)
        .attr('stroke-width', 1.5)
        .attr('fill', 'none')

    // Draw area under line
    const area = d3.area<number>()
        .x((d, i) => xScale(i))
        .y0(height)
        .y1(d => yScale(d))

    g.append('path')
        .datum(sortedValues)
        .attr('d', area)
        .attr('fill', color)
        .attr('opacity', 0.1)
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
import { watch } from 'vue'
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
    style="width: 100%; height: 100%; overflow: hidden; background: #fafafa"
  >
    <svg
      id="sparklines-svg"
      :width="size.width"
      :height="size.height"
      style="background: white; border-bottom: 1px solid #e0e0e0"
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
