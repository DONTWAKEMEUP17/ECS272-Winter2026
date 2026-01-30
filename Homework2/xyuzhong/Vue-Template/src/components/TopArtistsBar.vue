<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { isEmpty, debounce } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface ArtistData {
    artist_name: string
    track_count: number
    avg_popularity: number
    avg_followers: number
}

const data = ref<ArtistData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 60, right: 30, top: 40, bottom: 100 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

async function loadData() {
    const rawData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            track_name: d.track_name,
            artist_name: d.artist_name,
            track_popularity: +d.track_popularity,
            artist_popularity: +d.artist_popularity,
            artist_followers: +d.artist_followers
        }
    })

    // Group by artist and calculate statistics
    const groupedMap = new Map<string, any[]>()
    rawData.forEach((d: any) => {
        if (!groupedMap.has(d.artist_name)) {
            groupedMap.set(d.artist_name, [])
        }
        groupedMap.get(d.artist_name)!.push(d)
    })

    const aggregated: ArtistData[] = []
    groupedMap.forEach((tracks, artistName) => {
        aggregated.push({
            artist_name: artistName,
            track_count: tracks.length,
            avg_popularity: d3.mean(tracks, d => d.track_popularity) || 0,
            avg_followers: d3.mean(tracks, d => d.artist_followers) || 0
        })
    })

    // Sort by track count and get top 15
    aggregated.sort((a, b) => b.track_count - a.track_count)
    data.value = aggregated.slice(0, 15)
}

function onResize() {
    const target = container.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function initChart() {
    const chartContainer = d3.select('#artists-bar-svg')
    chartContainer.selectAll('*').remove()

    const yMax = d3.max(data.value, d => d.track_count) as number

    const xScale = d3
        .scaleBand<string>()
        .domain(data.value.map(d => d.artist_name))
        .range([margin.left, size.value.width - margin.right])
        .padding(0.15)

    const yScale = d3
        .scaleLinear()
        .domain([0, yMax * 1.1])
        .range([size.value.height - margin.bottom, margin.top])

    const colorScale = d3
        .scaleLinear<string>()
        .domain(d3.extent(data.value, d => d.avg_popularity) as [number, number])
        .range(['#D7191C', '#ABD9E9'])

    // X axis
    chartContainer
        .append('g')
        .attr('transform', `translate(0, ${size.value.height - margin.bottom})`)
        .call(d3.axisBottom(xScale))
        .selectAll('text')
        .style('text-anchor', 'end')
        .attr('dx', '-.8em')
        .attr('dy', '.15em')
        .attr('transform', 'rotate(-45)')
        .style('font-size', '11px')

    // Y axis
    chartContainer
        .append('g')
        .attr('transform', `translate(${margin.left}, 0)`)
        .call(d3.axisLeft(yScale))

    // X axis label
    chartContainer
        .append('text')
        .attr('transform', `translate(${size.value.width / 2}, ${size.value.height - margin.bottom + 85})`)
        .style('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .text('Artist Name')

    // Y axis label
    chartContainer
        .append('text')
        .attr('transform', `translate(${margin.left - 45}, ${size.value.height / 2}) rotate(-90)`)
        .style('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .text('Number of Tracks')

    // Title
    chartContainer
        .append('text')
        .attr('transform', `translate(${size.value.width / 2}, ${margin.top - 10})`)
        .style('text-anchor', 'middle')
        .style('font-size', '16px')
        .style('font-weight', 'bold')
        .text('Focus: Top Artists by Track Count (Colored by Avg Track Popularity)')

    // Bars
    chartContainer
        .append('g')
        .selectAll('rect')
        .data(data.value)
        .join('rect')
        .attr('x', d => xScale(d.artist_name) as number)
        .attr('y', d => yScale(d.track_count))
        .attr('width', xScale.bandwidth())
        .attr('height', d => size.value.height - margin.bottom - yScale(d.track_count))
        .attr('fill', d => colorScale(d.avg_popularity))
        .attr('stroke', '#333')
        .attr('stroke-width', 0.5)

    // Legend
    const legendX = size.value.width - margin.right - 140
    const legendY = margin.top + 10

    chartContainer
        .append('text')
        .attr('x', legendX)
        .attr('y', legendY - 5)
        .style('font-size', '12px')
        .style('font-weight', 'bold')
        .text('Avg Track Popularity')

    const gradientId = 'artists-legend-gradient'
    const defs = chartContainer.append('defs')
    const gradient = defs.append('linearGradient')
        .attr('id', gradientId)
        .attr('x1', '0%')
        .attr('x2', '100%')

    gradient.append('stop').attr('offset', '0%').attr('stop-color', '#D7191C')
    gradient.append('stop').attr('offset', '100%').attr('stop-color', '#ABD9E9')

    chartContainer
        .append('rect')
        .attr('x', legendX)
        .attr('y', legendY)
        .attr('width', 120)
        .attr('height', 15)
        .style('fill', `url(#${gradientId})`)

    chartContainer
        .append('text')
        .attr('x', legendX)
        .attr('y', legendY + 25)
        .style('font-size', '10px')
        .text('Low')

    chartContainer
        .append('text')
        .attr('x', legendX + 105)
        .attr('y', legendY + 25)
        .style('font-size', '10px')
        .text('High')
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
    <div class="chart-container d-flex" ref="container">
        <svg id="artists-bar-svg" width="100%" height="100%"></svg>
    </div>
</template>

<style scoped>
.chart-container {
    width: 100%;
    height: 100%;
}
</style>
