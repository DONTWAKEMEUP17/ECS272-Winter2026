<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
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
const margin: Margin = { left: 60, right: 30, top: 40, bottom: 60 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

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

function initChart() {
    const chartContainer = d3.select('#overview-scatter-svg')

    // Clear previous chart
    chartContainer.selectAll('*').remove()

    const xExtent = d3.extent(data.value, d => d.artist_popularity) as [number, number]
    const yExtent = d3.extent(data.value, d => d.track_popularity) as [number, number]

    const xScale = d3
        .scaleLinear()
        .domain([xExtent[0] - 5, xExtent[1] + 5])
        .range([margin.left, size.value.width - margin.right])

    const yScale = d3
        .scaleLinear()
        .domain([yExtent[0] - 5, yExtent[1] + 5])
        .range([size.value.height - margin.bottom, margin.top])

    const colorScale = d3
        .scaleLinear<string>()
        .domain(d3.extent(data.value, d => d.artist_followers) as [number, number])
        .range(['#FDE725', '#31688E'])

    // Add X axis
    chartContainer
        .append('g')
        .attr('transform', `translate(0, ${size.value.height - margin.bottom})`)
        .call(d3.axisBottom(xScale))

    // Add Y axis
    chartContainer
        .append('g')
        .attr('transform', `translate(${margin.left}, 0)`)
        .call(d3.axisLeft(yScale))

    // X axis label
    chartContainer
        .append('text')
        .attr('transform', `translate(${size.value.width / 2}, ${size.value.height - margin.bottom + 45})`)
        .style('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .text('Artist Popularity')

    // Y axis label
    chartContainer
        .append('text')
        .attr('transform', `translate(${margin.left - 45}, ${size.value.height / 2}) rotate(-90)`)
        .style('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .text('Track Popularity')

    // Title
    chartContainer
        .append('text')
        .attr('transform', `translate(${size.value.width / 2}, ${margin.top - 10})`)
        .style('text-anchor', 'middle')
        .style('font-size', '16px')
        .style('font-weight', 'bold')
        .text('Overview: Artist vs Track Popularity Distribution')

    // Add scatter points
    chartContainer
        .append('g')
        .selectAll('circle')
        .data(data.value)
        .join('circle')
        .attr('cx', d => xScale(d.artist_popularity))
        .attr('cy', d => yScale(d.track_popularity))
        .attr('r', 3)
        .attr('fill', d => colorScale(d.artist_followers))
        .attr('opacity', 0.6)
        .attr('stroke', 'white')
        .attr('stroke-width', 0.5)

    // Add legend for color
    const legendX = size.value.width - margin.right - 120
    const legendY = margin.top + 10

    chartContainer
        .append('text')
        .attr('x', legendX)
        .attr('y', legendY - 5)
        .style('font-size', '12px')
        .style('font-weight', 'bold')
        .text('Artist Followers')

    // Legend gradient
    const gradientId = 'legend-gradient'
    const defs = chartContainer.append('defs')
    const gradient = defs.append('linearGradient')
        .attr('id', gradientId)
        .attr('x1', '0%')
        .attr('x2', '100%')

    gradient.append('stop').attr('offset', '0%').attr('stop-color', '#FDE725')
    gradient.append('stop').attr('offset', '100%').attr('stop-color', '#31688E')

    chartContainer
        .append('rect')
        .attr('x', legendX)
        .attr('y', legendY)
        .attr('width', 100)
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
        .attr('x', legendX + 85)
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
        <svg id="overview-scatter-svg" width="100%" height="100%"></svg>
    </div>
</template>

<style scoped>
.chart-container {
    width: 100%;
    height: 100%;
}
</style>
