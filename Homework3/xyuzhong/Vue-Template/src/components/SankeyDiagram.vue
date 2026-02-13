<script setup lang="ts">
import * as d3 from 'd3'
import { sankey, sankeyLinkHorizontal } from 'd3-sankey'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { isEmpty, debounce } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface TrackData {
    track_popularity: number
    artist_popularity: number
}

interface SankeyNode {
    name: string
    category?: string
}

interface SankeyLink {
    source: number
    target: number
    value: number
}

interface SankeyData {
    nodes: SankeyNode[]
    links: SankeyLink[]
}

const data = ref<TrackData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 120, right: 120, top: 50, bottom: 40 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

const stats = computed(() => {
    if (isEmpty(data.value)) return null
    return {
        totalTracks: data.value.length,
        avgPopularity: (data.value.reduce((sum, d) => sum + d.track_popularity, 0) / data.value.length).toFixed(1)
    }
})

async function loadData() {
    const rawData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            track_popularity: +d.track_popularity,
            artist_popularity: +d.artist_popularity
        }
    })

    data.value = rawData.filter(d => 
        !isNaN(d.track_popularity) && !isNaN(d.artist_popularity) && d.artist_popularity > 0
    ) as TrackData[]
}

function getPopularityLevel(popularity: number): string {
    if (popularity < 34) return 'Low'
    if (popularity < 67) return 'Medium'
    return 'High'
}

function prepareSankeyData(): SankeyData {
    const nodes: SankeyNode[] = [
        { name: 'Artist: Low', category: 'artist' },
        { name: 'Artist: Medium', category: 'artist' },
        { name: 'Artist: High', category: 'artist' },
        { name: 'Track: Low', category: 'track' },
        { name: 'Track: Medium', category: 'track' },
        { name: 'Track: High', category: 'track' }
    ]

    // Count flows
    const linkMap = new Map<string, number>()
    
    data.value.forEach(d => {
        const artistLevel = getPopularityLevel(d.artist_popularity)
        const trackLevel = getPopularityLevel(d.track_popularity)
        const key = `${artistLevel}→${trackLevel}`
        linkMap.set(key, (linkMap.get(key) || 0) + 1)
    })

    // Create links
    const links: SankeyLink[] = []
    const artistMap = { 'Low': 0, 'Medium': 1, 'High': 2 }
    const trackMap = { 'Low': 3, 'Medium': 4, 'High': 5 }
    
    linkMap.forEach((count, key) => {
        const [artistLevel, trackLevel] = key.split('→')
        links.push({
            source: (artistMap as any)[artistLevel],
            target: (trackMap as any)[trackLevel],
            value: count
        })
    })

    return { nodes, links }
}

function onResize() {
    const target = container.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function initChart() {
    const svg = d3.select('#sankey-svg')
    svg.selectAll('*').remove()

    if (!canRender.value) return

    const chartWidth = size.value.width - margin.left - margin.right
    const chartHeight = size.value.height - margin.top - margin.bottom

    // Title
    svg.append('text')
        .attr('x', size.value.width / 2)
        .attr('y', 25)
        .attr('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text(`Artist vs Track Popularity Flow (${stats.value?.totalTracks || 0} tracks, Avg: ${stats.value?.avgPopularity || 0})`)

    const sankeyData = prepareSankeyData()

    // Create sankey
    const layout = sankey<SankeyNode, SankeyLink>()
        .nodeWidth(25)
        .nodePadding(80)
        .extent([[0, 0], [chartWidth, chartHeight]])

    const graph = layout({
        nodes: sankeyData.nodes.map(d => ({ ...d })),
        links: sankeyData.links.map(d => ({ ...d }))
    })

    const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`)

    // Color schemes
    const artistColors: { [key: string]: string } = {
        'Artist: Low': '#FF6B6B',
        'Artist: Medium': '#FFD93D',
        'Artist: High': '#1DB954'
    }

    const trackColors: { [key: string]: string } = {
        'Track: Low': '#4ECDC4',
        'Track: Medium': '#45B7D1',
        'Track: High': '#FF6B9D'
    }

    // Draw links
    g.append('g')
        .attr('fill', 'none')
        .selectAll('path')
        .data(graph.links)
        .join('path')
        .attr('d', sankeyLinkHorizontal() as any)
        .attr('stroke', d => {
            const sourceName = sankeyData.nodes[(d.source as any).index].name
            return sourceName.includes('Artist') ? artistColors[sourceName] : trackColors[sourceName]
        })
        .attr('stroke-opacity', 0.4)
        .attr('stroke-width', d => Math.max(1, d.width))

    // Draw nodes
    g.append('g')
        .selectAll('rect')
        .data(graph.nodes)
        .join('rect')
        .attr('x', d => d.x0)
        .attr('y', d => d.y0)
        .attr('height', d => d.y1 - d.y0)
        .attr('width', d => d.x1 - d.x0)
        .attr('fill', d => {
            const name = d.name
            return name.includes('Artist') ? artistColors[name] : trackColors[name]
        })
        .attr('stroke', '#fff')
        .attr('stroke-width', 2)

    // Node labels
    g.append('g')
        .selectAll('text')
        .data(graph.nodes)
        .join('text')
        .attr('x', d => d.x0 < chartWidth / 2 ? d.x0 - 10 : d.x1 + 10)
        .attr('y', d => (d.y0 + d.y1) / 2)
        .attr('dy', '0.35em')
        .attr('text-anchor', d => d.x0 < chartWidth / 2 ? 'end' : 'start')
        .style('font-size', '12px')
        .style('font-weight', 'bold')
        .style('fill', '#333')
        .style('pointer-events', 'none')
        .text(d => d.name)

    // Link value labels
    g.append('g')
        .selectAll('text')
        .data(graph.links)
        .join('text')
        .attr('x', d => ((d.source as any).x1 + (d.target as any).x0) / 2)
        .attr('y', d => (d.y0 + d.y1) / 2 - 5)
        .attr('text-anchor', 'middle')
        .style('font-size', '10px')
        .style('fill', '#666')
        .style('pointer-events', 'none')
        .text(d => d.value)

    // Legend
    const legendY = chartHeight + 10
    svg.append('text')
        .attr('x', margin.left + 10)
        .attr('y', margin.top + legendY)
        .style('font-size', '11px')
        .style('fill', '#666')
        .text('Flow size represents number of tracks | Colors show popularity levels (Low → High)')
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

watch([data, size], () => {
    if (canRender.value) {
        initChart()
    }
}, { deep: true })
</script>

<template>
  <div
    ref="container"
    class="sankey-container"
    style="width: 100%; height: 100%; overflow: hidden; background: #ffffff"
  >
    <svg
      id="sankey-svg"
      :width="size.width"
      :height="size.height"
      style="background: #ffffff; border-bottom: 1px solid #e0e0e0"
    />
  </div>
</template>

<style scoped>
.sankey-container {
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

:deep(svg) {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}
</style>
