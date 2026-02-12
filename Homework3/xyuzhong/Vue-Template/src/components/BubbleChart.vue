<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { isEmpty, debounce } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface TrackData {
    track_name: string
    artist_name: string
    track_duration_ms: number
    track_popularity: number
    artist_followers: number
    explicit: boolean
}

const data = ref<TrackData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 70, right: 30, top: 40, bottom: 70 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

const stats = computed(() => {
    if (isEmpty(data.value)) return null
    const explicitCount = data.value.filter(d => d.explicit).length
    const cleanCount = data.value.length - explicitCount
    return {
        totalTracks: data.value.length,
        explicitCount: explicitCount,
        cleanCount: cleanCount
    }
})

async function loadData() {
    const rawData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            track_name: d.track_name,
            artist_name: d.artist_name,
            track_duration_ms: +d.track_duration_ms,
            track_popularity: +d.track_popularity,
            artist_followers: +d.artist_followers,
            explicit: d.explicit === 'True'
        }
    })

    data.value = rawData.filter(d => 
        !isNaN(d.track_duration_ms) && !isNaN(d.track_popularity) && 
        !isNaN(d.artist_followers) && d.artist_followers > 0
    ) as TrackData[]
}

function onResize() {
    const target = container.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function initChart() {
    const chartContainer = d3.select('#bubble-chart-svg')
    chartContainer.selectAll('*').remove()

    const xExtent = d3.extent(data.value, d => d.track_duration_ms / 1000 / 60) as [number, number] // Convert to minutes
    const yExtent = d3.extent(data.value, d => d.track_popularity) as [number, number]
    const rExtent = d3.extent(data.value, d => d.artist_followers) as [number, number]

    const xScale = d3
        .scaleLinear()
        .domain([xExtent[0] - 0.5, xExtent[1] + 0.5])
        .range([margin.left, size.value.width - margin.right])

    const yScale = d3
        .scaleLinear()
        .domain([yExtent[0] - 5, yExtent[1] + 5])
        .range([size.value.height - margin.bottom, margin.top])

    const radiusScale = d3
        .scaleSqrt()
        .domain([rExtent[0], rExtent[1]])
        .range([2, 15])

    // Add title
    chartContainer
        .append('text')
        .attr('x', size.value.width / 2)
        .attr('y', margin.top - 5)
        .attr('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text(`Track Duration vs Popularity (${stats.value?.totalTracks || 0} tracks: ${stats.value?.cleanCount || 0} clean, ${stats.value?.explicitCount || 0} explicit)`)

    // X axis
    chartContainer
        .append('g')
        .attr('transform', `translate(0, ${size.value.height - margin.bottom})`)
        .call(d3.axisBottom(xScale))
        .append('text')
        .attr('x', size.value.width / 2)
        .attr('y', 50)
        .style('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#333')
        .text('Track Duration (minutes)')

    // Y axis
    chartContainer
        .append('g')
        .attr('transform', `translate(${margin.left}, 0)`)
        .call(d3.axisLeft(yScale))
        .append('text')
        .attr('transform', `translate(-50, ${size.value.height / 2}) rotate(-90)`)
        .style('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#333')
        .text('Track Popularity')

    // Bubbles
    chartContainer
        .append('g')
        .selectAll('circle')
        .data(data.value)
        .join('circle')
        .attr('cx', d => xScale(d.track_duration_ms / 1000 / 60))
        .attr('cy', d => yScale(d.track_popularity))
        .attr('r', d => radiusScale(d.artist_followers))
        .attr('fill', d => d.explicit ? '#FF6B6B' : '#1DB954')
        .attr('opacity', 0.5)
        .attr('stroke', '#282828')
        .attr('stroke-width', 0.5)

    // Legend for bubble sizes
    const legendX = margin.left
    const legendY = margin.top + 10
    const sizes = [1000000, 50000000, 100000000]
    const labels = ['1M', '50M', '100M']

    chartContainer
        .append('text')
        .attr('x', legendX)
        .attr('y', legendY - 10)
        .style('font-size', '12px')
        .style('font-weight', 'bold')
        .text('Artist Followers:')

    sizes.forEach((size, i) => {
        const r = radiusScale(size)
        const baseY = legendY + 30 + i * 40

        chartContainer
            .append('circle')
            .attr('cx', legendX + r + 5)
            .attr('cy', baseY)
            .attr('r', r)
            .attr('fill', '#1DB954')
            .attr('opacity', 0.5)
            .attr('stroke', '#282828')
            .attr('stroke-width', 0.5)

        chartContainer
            .append('text')
            .attr('x', legendX + r * 2 + 15)
            .attr('y', baseY + 4)
            .style('font-size', '11px')
            .style('fill', '#b3b3b3')
            .text(labels[i])
    })

    // Legend for explicit content
    const explicitLegendX = size.value.width - margin.right - 150
    const explicitLegendY = margin.top + 10

    chartContainer
        .append('text')
        .attr('x', explicitLegendX)
        .attr('y', explicitLegendY - 10)
        .style('font-size', '12px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text('Explicit Content:')

    chartContainer
        .append('circle')
        .attr('cx', explicitLegendX + 10)
        .attr('cy', explicitLegendY + 15)
        .attr('r', 4)
        .attr('fill', '#FF6B6B')
        .attr('opacity', 0.5)
        .attr('stroke', '#282828')
        .attr('stroke-width', 0.5)

    chartContainer
        .append('text')
        .attr('x', explicitLegendX + 25)
        .attr('y', explicitLegendY + 19)
        .style('font-size', '11px')
        .style('fill', '#b3b3b3')
        .text('Explicit')

    chartContainer
        .append('circle')
        .attr('cx', explicitLegendX + 10)
        .attr('cy', explicitLegendY + 35)
        .attr('r', 4)
        .attr('fill', '#1DB954')
        .attr('opacity', 0.5)
        .attr('stroke', '#282828')
        .attr('stroke-width', 0.5)

    chartContainer
        .append('text')
        .attr('x', explicitLegendX + 25)
        .attr('y', explicitLegendY + 39)
        .style('font-size', '11px')
        .style('fill', '#b3b3b3')
        .text('Clean')
}

watch(
    [data, size],
    ([dataVal, sizeVal]) => {
        if (!isEmpty(dataVal) && sizeVal.width > 0 && sizeVal.height > 0) {
            initChart()
        }
    },
    { deep: true }
)

const debouncedOnResize = debounce(onResize, 100)

onMounted(() => {
    window.addEventListener('resize', debouncedOnResize)
    onResize()
    loadData()
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', debouncedOnResize)
})
</script>

<template>
    <div class="chart-container d-flex" ref="container" style="background: #191414;">
        <svg id="bubble-chart-svg" width="100%" height="100%"></svg>
    </div>
</template>

<style scoped>
.chart-container {
    width: 100%;
    height: 100%;
    background: #191414;
}
</style>
