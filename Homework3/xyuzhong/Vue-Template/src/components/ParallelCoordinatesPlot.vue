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
    artist_popularity: number
}

interface Dimension {
    name: string
    key: keyof TrackData
    scale: d3.ScaleLinear<number, number>
}

const data = ref<TrackData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 80, right: 80, top: 50, bottom: 60 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

async function loadData() {
    const rawData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            track_name: d.track_name,
            artist_name: d.artist_name,
            track_duration_ms: +d.track_duration_ms,
            track_popularity: +d.track_popularity,
            artist_followers: +d.artist_followers,
            artist_popularity: +d.artist_popularity
        }
    })

    data.value = rawData.filter(d => 
        !isNaN(d.track_duration_ms) && !isNaN(d.track_popularity) && 
        !isNaN(d.artist_followers) && !isNaN(d.artist_popularity) &&
        d.artist_followers > 0 && d.track_duration_ms > 0
    ) as TrackData[]
    
    // Sample data if too many (keep top 300 by popularity)
    if (data.value.length > 300) {
        data.value = data.value
            .sort((a, b) => b.track_popularity - a.track_popularity)
            .slice(0, 300)
    }
}

function onResize() {
    const target = container.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function initChart() {
    const svg = d3.select('#parallel-coordinates-svg')
    svg.selectAll('*').remove()

    if (!canRender.value || data.value.length === 0) return

    // Define dimensions
    const dimensions: Dimension[] = [
        {
            name: 'Duration (min)',
            key: 'track_duration_ms',
            scale: d3.scaleLinear()
                .domain(d3.extent(data.value, d => d.track_duration_ms / 1000 / 60) as [number, number])
                .range([size.value.height - margin.bottom, margin.top])
        },
        {
            name: 'Track Pop.',
            key: 'track_popularity',
            scale: d3.scaleLinear()
                .domain([0, 100])
                .range([size.value.height - margin.bottom, margin.top])
        },
        {
            name: 'Artist Pop.',
            key: 'artist_popularity',
            scale: d3.scaleLinear()
                .domain([0, 100])
                .range([size.value.height - margin.bottom, margin.top])
        },
        {
            name: 'Followers (M)',
            key: 'artist_followers',
            scale: d3.scaleLinear()
                .domain(d3.extent(data.value, d => d.artist_followers / 1000000) as [number, number])
                .range([size.value.height - margin.bottom, margin.top])
        }
    ]

    const xScale = d3.scalePoint<string>()
        .domain(dimensions.map(d => d.name))
        .range([margin.left, size.value.width - margin.right])

    const colorScale = d3.scaleLinear<string>()
        .domain([0, 100])
        .range(['#282828', '#1DB954'])

    // Add title
    svg.append('text')
        .attr('x', size.value.width / 2)
        .attr('y', margin.top - 15)
        .attr('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text('Multi-Dimensional Track Analysis (4 Dimensions)')

    // Add axes
    dimensions.forEach((dim) => {
        const x = xScale(dim.name)!

        // Vertical line for axis
        svg.append('line')
            .attr('x1', x)
            .attr('y1', margin.top)
            .attr('x2', x)
            .attr('y2', size.value.height - margin.bottom)
            .style('stroke', '#282828')
            .style('stroke-width', 2)

        // Axis label
        svg.append('text')
            .attr('x', x)
            .attr('y', size.value.height - margin.bottom + 30)
            .attr('text-anchor', 'middle')
            .style('font-size', '11px')
            .style('fill', '#b3b3b3')
            .text(dim.name)

        // Min/Max values
        const extent = dim.scale.domain() as [number, number]
        svg.append('text')
            .attr('x', x)
            .attr('y', size.value.height - margin.bottom + 10)
            .attr('text-anchor', 'middle')
            .style('font-size', '9px')
            .style('fill', '#666')
            .text(extent[1].toFixed(1))

        svg.append('text')
            .attr('x', x)
            .attr('y', margin.top - 5)
            .attr('text-anchor', 'middle')
            .style('font-size', '9px')
            .style('fill', '#666')
            .text(extent[0].toFixed(1))
    })

    // Add lines for each data point
    const lines = svg.selectAll('.line')
        .data(data.value)
        .enter()
        .append('g')
        .attr('class', 'line')

    lines.append('path')
        .attr('d', (d) => {
            const points = dimensions.map((dim) => {
                const x = xScale(dim.name)!
                let value = d[dim.key] as number
                
                // Convert duration to minutes
                if (dim.key === 'track_duration_ms') {
                    value = value / 1000 / 60
                }
                // Convert followers to millions
                else if (dim.key === 'artist_followers') {
                    value = value / 1000000
                }

                const y = dim.scale(value)
                return [x, y]
            })

            return d3.line()(points as [number, number][])
        })
        .style('stroke', d => colorScale(d.track_popularity))
        .style('stroke-width', 1.5)
        .style('opacity', 0.3)
        .style('fill', 'none')

    // Add interactivity
    lines
        .on('mouseover', function(event, d) {
            // Highlight this line
            d3.select(this).select('path')
                .style('stroke', '#000000')
                .style('stroke-width', 3)
                .style('opacity', 1)

            // Show tooltip
            const mousePos = d3.pointer(event)
            svg.append('text')
                .attr('class', 'tooltip')
                .attr('x', mousePos[0])
                .attr('y', mousePos[1] - 15)
                .style('font-size', '10px')
                .style('fill', '#000000')
                .style('font-weight', 'bold')
                .style('pointer-events', 'none')
                .text(d.track_name.length > 20 ? d.track_name.substring(0, 17) + '...' : d.track_name)

            svg.append('text')
                .attr('class', 'tooltip')
                .attr('x', mousePos[0])
                .attr('y', mousePos[1])
                .style('font-size', '9px')
                .style('fill', '#000000')
                .style('pointer-events', 'none')
                .text(d.artist_name.length > 20 ? d.artist_name.substring(0, 17) + '...' : d.artist_name)
        })
        .on('mouseout', function(event, d) {
            d3.select(this).select('path')
                .style('stroke', colorScale(d.track_popularity))
                .style('stroke-width', 1.5)
                .style('opacity', 0.3)

            svg.selectAll('.tooltip').remove()
        })
}

const debouncedOnResize = debounce(onResize, 200)

watch(
    () => canRender.value,
    () => {
        if (canRender.value) {
            initChart()
        }
    }
)

watch(
    () => size.value,
    () => {
        if (canRender.value) {
            initChart()
        }
    },
    { deep: true }
)

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
        <svg id="parallel-coordinates-svg" width="100%" height="100%"></svg>
    </div>
</template>

<style scoped>
.chart-container {
    width: 100%;
    height: 100%;
    background: #191414;
}
</style>
