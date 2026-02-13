<script setup lang="ts">
import * as d3 from 'd3'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { isEmpty, debounce } from 'lodash'
import { ComponentSize, Margin } from '../types'

interface GenreData {
    genre: string
    track_count: number
    avg_popularity: number
}

const data = ref<GenreData[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 80, right: 30, top: 40, bottom: 150 }

const container = ref<HTMLElement | null>(null)
const canRender = computed(() => !isEmpty(data.value) && size.value.width > 0 && size.value.height > 0)

const stats = computed(() => {
    if (isEmpty(data.value)) return null
    return {
        genreCount: data.value.length,
        avgPopularity: (data.value.reduce((sum, d) => sum + d.avg_popularity, 0) / data.value.length).toFixed(1)
    }
})

async function loadData() {
    const rawData = await d3.csv('../../data/track_data_final.csv', (d: any) => {
        return {
            artist_genres: d.artist_genres,
            track_popularity: +d.track_popularity
        }
    })

    // Parse genres and aggregate data
    const genreMap = new Map<string, { count: number; popularities: number[] }>()
    
    rawData.forEach((d: any) => {
        if (!d.artist_genres || d.artist_genres.trim() === '') return
        
        // Parse genres - handle formats like ['genre1' / 'genre2'] or comma-separated
        let genreString = d.artist_genres
            .replace(/\[/g, '')  // Remove [
            .replace(/\]/g, '')  // Remove ]
            .replace(/'/g, '')   // Remove quotes
            .replace(/\//g, ',') // Replace / with comma
        
        const genres = genreString
            .split(',')
            .map((g: string) => g.trim())
            .filter((g: string) => g.length > 0 && g !== '[]')
        
        genres.forEach((genre: string) => {
            if (!genreMap.has(genre)) {
                genreMap.set(genre, { count: 0, popularities: [] })
            }
            const entry = genreMap.get(genre)!
            entry.count += 1
            entry.popularities.push(d.track_popularity)
        })
    })

    // Convert to array and calculate averages, then sort by count
    const genreArray: GenreData[] = Array.from(genreMap.entries())
        .map(([genre, data]) => ({
            genre,
            track_count: data.count,
            avg_popularity: data.popularities.reduce((a, b) => a + b, 0) / data.popularities.length
        }))
        .sort((a, b) => b.track_count - a.track_count)
        .slice(0, 25) // Top 25 genres

    data.value = genreArray
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
        .domain(data.value.map(d => d.genre))
        .range([margin.left, size.value.width - margin.right])
        .padding(0.15)

    const yScale = d3
        .scaleLinear()
        .domain([0, yMax * 1.1])
        .range([size.value.height - margin.bottom, margin.top])

    const colorScale = d3
        .scaleLinear<string>()
        .domain([0, 100])
        .range(['#282828', '#1DB954']) // Dark gray to Spotify green

    // Add title
    chartContainer
        .append('text')
        .attr('x', size.value.width / 2)
        .attr('y', margin.top - 10)
        .attr('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text(`Genre Distribution (Top ${stats.value?.genreCount || 0}, Avg popularity: ${stats.value?.avgPopularity || 0})`)

    // Add X axis
    chartContainer
        .append('g')
        .attr('transform', `translate(0, ${size.value.height - margin.bottom})`)
        .call(d3.axisBottom(xScale))
        .selectAll('text')
        .style('text-anchor', 'end')
        .attr('dx', '-.8em')
        .attr('dy', '.15em')
        .attr('transform', 'rotate(-45)')
        .style('font-size', '10px')
        .style('fill', '#333')

    // Add Y axis
    chartContainer
        .append('g')
        .attr('transform', `translate(${margin.left}, 0)`)
        .call(d3.axisLeft(yScale))
        .selectAll('text')
        .style('fill', '#333')

    // Y axis label
    chartContainer
        .append('text')
        .attr('transform', `translate(${margin.left - 50}, ${size.value.height / 2}) rotate(-90)`)
        .style('text-anchor', 'middle')
        .style('fill', '#333')
        .style('font-size', '12px')
        .text('Number of Tracks')

    // X axis label
    chartContainer
        .append('text')
        .attr('x', size.value.width / 2)
        .attr('y', size.value.height - 5)
        .attr('text-anchor', 'middle')
        .style('fill', '#333')
        .style('font-size', '12px')
        .text('Genre')

    // Add bars
    chartContainer
        .selectAll('.bar')
        .data(data.value)
        .enter()
        .append('rect')
        .attr('class', 'bar')
        .attr('x', d => xScale(d.genre))
        .attr('y', d => yScale(d.track_count))
        .attr('width', xScale.bandwidth())
        .attr('height', d => size.value.height - margin.bottom - yScale(d.track_count))
        .attr('fill', d => colorScale(d.avg_popularity))
        .attr('opacity', 0.8)

    // Add hover tooltip
    chartContainer
        .selectAll('.bar')
        .on('mouseover', function(event, d) {
            d3.select(this).attr('opacity', 1)
            
            chartContainer
                .append('text')
                .attr('class', 'tooltip')
                .attr('x', +d3.select(this).attr('x') + xScale.bandwidth() / 2)
                .attr('y', +d3.select(this).attr('y') - 10)
                .attr('text-anchor', 'middle')
                .style('font-size', '11px')
                .style('fill', '#1DB954')
                .style('font-weight', 'bold')
                .text(`${d.track_count} tracks`)
            
            chartContainer
                .append('text')
                .attr('class', 'tooltip')
                .attr('x', +d3.select(this).attr('x') + xScale.bandwidth() / 2)
                .attr('y', +d3.select(this).attr('y') - 25)
                .attr('text-anchor', 'middle')
                .style('font-size', '10px')
                .style('fill', '#b3b3b3')
                .text(`Pop: ${d.avg_popularity.toFixed(1)}`)
        })
        .on('mouseout', function() {
            d3.select(this).attr('opacity', 0.8)
            chartContainer.selectAll('.tooltip').remove()
        })

    // Add legend
    const legendX = size.value.width - margin.right - 150
    const legendY = margin.top + 20
    const legendHeight = 80

    // Legend background
    chartContainer
        .append('rect')
        .attr('x', legendX - 10)
        .attr('y', legendY - 10)
        .attr('width', 150)
        .attr('height', legendHeight)
        .attr('fill', '#f5f5f5')
        .attr('opacity', 0.95)
        .attr('stroke', '#1DB954')
        .attr('stroke-width', 1)

    // Legend title
    chartContainer
        .append('text')
        .attr('x', legendX + 65)
        .attr('y', legendY + 10)
        .attr('text-anchor', 'middle')
        .style('font-size', '11px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text('Avg Popularity')

    // Legend gradient scale (0-100)
    const minPop = 0
    const maxPop = 100

    // Legend color bar
    chartContainer
        .append('rect')
        .attr('x', legendX)
        .attr('y', legendY + 25)
        .attr('width', 120)
        .attr('height', 12)
        .attr('fill', 'url(#genreGradient)')

    // Add gradient definition
    const defs = chartContainer.append('defs')
    const gradient = defs
        .append('linearGradient')
        .attr('id', 'genreGradient')
        .attr('x1', '0%')
        .attr('x2', '100%')

    gradient
        .append('stop')
        .attr('offset', '0%')
        .attr('stop-color', '#282828')

    gradient
        .append('stop')
        .attr('offset', '100%')
        .attr('stop-color', '#1DB954')

    // Legend labels (0-100 scale)
    chartContainer
        .append('text')
        .attr('x', legendX)
        .attr('y', legendY + 50)
        .style('font-size', '9px')
        .style('fill', '#333')
        .text('Low (0)')

    chartContainer
        .append('text')
        .attr('x', legendX + 120)
        .attr('y', legendY + 50)
        .attr('text-anchor', 'end')
        .style('font-size', '9px')
        .style('fill', '#333')
        .text('High (100)')
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
    <div class="chart-container d-flex" ref="container" style="background: #ffffff;">
        <svg id="artists-bar-svg" width="100%" height="100%"></svg>
    </div>
</template>

<style scoped>
.chart-container {
    width: 100%;
    height: 100%;
    background: #ffffff;
}
</style>
