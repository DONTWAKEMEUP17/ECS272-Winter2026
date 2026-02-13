<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import { isEmpty, debounce, groupBy } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface TrackData {
    track_popularity: number
    artist_popularity: number
    artist_followers: number
    track_name: string
    artist_name: string
}

interface TreeNode {
    name: string
    value: number
    popularity: number
}

interface TreeHierarchy extends d3.HierarchyRectangularNode<TreeNode> {
    children?: TreeHierarchy[]
}

const data = ref<TrackData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 0, right: 0, top: 30, bottom: 0 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

const stats = computed(() => {
    if (isEmpty(data.value)) return null
    const grouped = groupBy(data.value, 'artist_name')
    let children: TreeNode[] = Object.entries(grouped).map(([artist, tracks]) => ({
        name: artist,
        value: tracks.length,
        popularity: (tracks.reduce((sum: number, t: any) => sum + t.track_popularity, 0) / tracks.length)
    }))
    children = children
        .sort((a, b) => b.value - a.value)
        .slice(0, 40)
    
    const avgPopularity = (children.reduce((sum, n) => sum + n.popularity, 0) / children.length).toFixed(1)
    const totalTracks = children.reduce((sum, n) => sum + n.value, 0)
    return {
        artistCount: Object.keys(grouped).length,
        topArtistCount: children.length,
        totalTracks: totalTracks,
        avgPopularity: avgPopularity
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

function onResize() {
    const target = container.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function prepareTreemapData() {
    // Group by artist and count tracks, calculate average popularity
    const grouped = groupBy(data.value, 'artist_name')
    let children: TreeNode[] = Object.entries(grouped).map(([artist, tracks]) => ({
        name: artist,
        value: tracks.length, // Size by track count
        popularity: (tracks.reduce((sum: number, t: any) => sum + t.track_popularity, 0) / tracks.length)
    }))

    // Sort by track count and take top 40 artists
    children = children
        .sort((a, b) => b.value - a.value)
        .slice(0, 40)

    return {
        name: 'root',
        value: 0,
        popularity: 0,
        children: children
    }
}

function initChart() {
    const svg = d3.select('#treemap-svg')
    svg.selectAll('*').remove()

    if (!canRender.value) return

    // Add title
    svg.append('text')
        .attr('x', 10)
        .attr('y', 20)
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#333')
        .text(`Artist Tracks numbers & popularity (Top 40: ${stats.value?.topArtistCount || 0} artists, ${stats.value?.totalTracks || 0} total tracks, Avg: ${stats.value?.avgPopularity || 0})`)

    const chartWidth = size.value.width
    const chartHeight = size.value.height - margin.top

    // Prepare data
    const hierarchyData = prepareTreemapData()

    // Create hierarchy
    const root = d3.hierarchy(hierarchyData)
        .sum(d => d.value)
        .sort((a, b) => (b.value || 0) - (a.value || 0))

    // Create treemap layout
    const treemap = d3.treemap<TreeNode>()
        .size([chartWidth, chartHeight])
        .paddingTop(margin.top)
        .paddingRight(10)
        .paddingBottom(10)
        .paddingLeft(10)

    const treemapRoot = treemap(root) as TreeHierarchy

    // Color scale based on popularity
    const colorScale = d3.scaleLinear<string>()
        .domain([0, 100])
        .range(['#E8F4F8', '#0066CC']) // Light blue to deep blue

    // Create cells
    const cells = svg.selectAll('g')
        .data(treemapRoot.leaves() as TreeHierarchy[])
        .enter()
        .append('g')
        .attr('transform', (d: TreeHierarchy) => `translate(${d.x0},${d.y0})`)

    // Draw rectangles
    cells.append('rect')
        .attr('width', (d: TreeHierarchy) => d.x1 - d.x0)
        .attr('height', (d: TreeHierarchy) => d.y1 - d.y0)
        .attr('fill', (d: TreeHierarchy) => colorScale(d.data.popularity))
        .attr('stroke', '#fff')
        .attr('stroke-width', 2)
        .style('opacity', 0.85)

    // Add text labels (only if rectangle is large enough)
    cells.append('text')
        .attr('x', 4)
        .attr('y', 20)
        .style('font-size', '11px')
        .style('font-weight', 'bold')
        .style('fill', '#000')
        .style('pointer-events', 'none')
        .text((d: TreeHierarchy) => {
            const width = d.x1 - d.x0
            const height = d.y1 - d.y0
            if (width > 60 && height > 40) {
                return (d.data.name as string).length > 15 
                    ? (d.data.name as string).substring(0, 12) + '...'
                    : (d.data.name as string)
            }
            return ''
        })

    // Add count labels
    cells.append('text')
        .attr('x', 4)
        .attr('y', 35)
        .style('font-size', '10px')
        .style('fill', '#333')
        .style('pointer-events', 'none')
        .text((d: TreeHierarchy) => {
            const width = d.x1 - d.x0
            const height = d.y1 - d.y0
            if (width > 60 && height > 40) {
                return `${d.data.value} tracks`
            }
            return ''
        })

    // Add popularity labels
    cells.append('text')
        .attr('x', 4)
        .attr('y', 50)
        .style('font-size', '9px')
        .style('fill', '#666')
        .style('pointer-events', 'none')
        .text((d: TreeHierarchy) => {
            const width = d.x1 - d.x0
            const height = d.y1 - d.y0
            if (width > 60 && height > 40) {
                return `Pop: ${d.data.popularity.toFixed(0)}`
            }
            return ''
        })

    // Add interactivity
    cells.on('mouseover', function() {
        d3.select(this).select('rect')
            .style('opacity', 1)
            .attr('stroke-width', 3)
    })
    .on('mouseout', function() {
        d3.select(this).select('rect')
            .style('opacity', 0.85)
            .attr('stroke-width', 2)
    })

    // Add legend at the bottom
    const legendX = 20
    const legendY = chartHeight - 50
    const legendHeight = 70

    // Legend background
    svg.append('rect')
        .attr('x', legendX)
        .attr('y', legendY)
        .attr('width', 320)
        .attr('height', legendHeight)
        .attr('fill', '#f5f5f5')
        .attr('opacity', 0.95)
        .attr('stroke', '#0066CC')
        .attr('stroke-width', 1)

    // First row: Color legend
    svg.append('text')
        .attr('x', legendX + 10)
        .attr('y', legendY + 15)
        .style('font-size', '10px')
        .style('font-weight', 'bold')
        .style('fill', '#0066CC')
        .text('Color: Artist Avg Popularity')

    // Legend color bar
    svg.append('rect')
        .attr('x', legendX + 220)
        .attr('y', legendY + 8)
        .attr('width', 80)
        .attr('height', 12)
        .attr('fill', 'url(#treemapGradient)')

    // Add gradient definition
    const defs = svg.append('defs')
    const gradient = defs
        .append('linearGradient')
        .attr('id', 'treemapGradient')
        .attr('x1', '0%')
        .attr('x2', '100%')

    gradient
        .append('stop')
        .attr('offset', '0%')
        .attr('stop-color', '#E8F4F8')

    gradient
        .append('stop')
        .attr('offset', '100%')
        .attr('stop-color', '#0066CC')

    // Legend labels for colors with values
    svg.append('text')
        .attr('x', legendX + 220)
        .attr('y', legendY + 28)
        .style('font-size', '9px')
        .style('fill', '#333')
        .text('Low (0)')

    svg.append('text')
        .attr('x', legendX + 300)
        .attr('y', legendY + 28)
        .attr('text-anchor', 'end')
        .style('font-size', '9px')
        .style('fill', '#333')
        .text('High (100)')

    // Second row: Size legend
    svg.append('text')
        .attr('x', legendX + 10)
        .attr('y', legendY + 55)
        .style('font-size', '10px')
        .style('font-weight', 'bold')
        .style('fill', '#0066CC')
        .text('Size: Number of Tracks')
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
    class="treemap-container"
    style="width: 100%; height: 100%; overflow: hidden; background: #ffffff"
  >
    <svg
      id="treemap-svg"
      :width="size.width"
      :height="size.height"
      style="background: #ffffff; border-bottom: 1px solid #e0e0e0"
    />
  </div>
</template>

<style scoped>
.treemap-container {
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

:deep(svg) {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

:deep(rect) {
  cursor: pointer;
  transition: opacity 0.2s;
}
</style>
