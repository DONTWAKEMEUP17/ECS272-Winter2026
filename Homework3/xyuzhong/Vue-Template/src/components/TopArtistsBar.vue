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
const chartType = ref<'bar' | 'pie'>('bar')

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

function drawBarChart() {
    const chartContainer = d3.select('#artists-bar-svg')
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
        .range(['#282828', '#1DB954'])

    const title = chartContainer.selectAll('.chart-title').data([null])
    title
        .join('text')
        .attr('class', 'chart-title')
        .attr('x', size.value.width / 2)
        .attr('y', margin.top - 10)
        .attr('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#333')
        .text(`Genre Distribution (Bar View)`)

    // Add X axis
    const xAxisGroup = chartContainer.selectAll('.x-axis').data([null])
    xAxisGroup
        .join('g')
        .attr('class', 'x-axis')
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
    const yAxisGroup = chartContainer.selectAll('.y-axis').data([null])
    yAxisGroup
        .join('g')
        .attr('class', 'y-axis')
        .attr('transform', `translate(${margin.left}, 0)`)
        .call(d3.axisLeft(yScale))
        .selectAll('text')
        .style('fill', '#333')

    // Add bars with smooth transitions
    chartContainer
        .selectAll('.bar')
        .data(data.value)
        .join(
            enter => enter.append('rect')
                .attr('class', 'bar')
                .attr('x', d => xScale(d.genre))
                .attr('y', size.value.height - margin.bottom)
                .attr('width', xScale.bandwidth())
                .attr('height', 0)
                .attr('fill', d => colorScale(d.avg_popularity))
                .attr('opacity', 0.8)
                .transition()
                .duration(600)
                .attr('y', d => yScale(d.track_count))
                .attr('height', d => size.value.height - margin.bottom - yScale(d.track_count)),
            update => update
                .transition()
                .duration(600)
                .attr('x', d => xScale(d.genre))
                .attr('y', d => yScale(d.track_count))
                .attr('height', d => size.value.height - margin.bottom - yScale(d.track_count))
                .attr('width', xScale.bandwidth()),
            exit => exit
                .transition()
                .duration(300)
                .attr('y', size.value.height - margin.bottom)
                .attr('height', 0)
                .remove()
        )
        .style('cursor', 'pointer')
        .on('mouseover', function(event, d) {
            d3.select(this)
                .transition()
                .duration(200)
                .attr('opacity', 1)
            
            chartContainer.selectAll('.bar')
                .transition()
                .duration(200)
                .attr('opacity', (bar: any) => bar === d ? 1 : 0.2)
            
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
            d3.select(this)
                .transition()
                .duration(200)
                .attr('opacity', 0.8)
            
            chartContainer.selectAll('.bar')
                .transition()
                .duration(200)
                .attr('opacity', 0.8)
            
            chartContainer.selectAll('.tooltip').remove()
        })

    // Add legend
    addLegend(chartContainer, colorScale)
}

function drawPieChart() {
    const chartContainer = d3.select('#artists-bar-svg')
    
    const colorScale = d3
        .scaleLinear<string>()
        .domain([0, 100])
        .range(['#282828', '#1DB954'])

    // Leave space for legend and title
    const availableHeight = size.value.height - margin.top - 40
    const availableWidth = size.value.width - 180
    const radius = Math.min(availableWidth, availableHeight) / 2.2
    const centerX = availableWidth / 2
    const centerY = availableHeight / 2 + margin.top

    const pie = d3.pie<GenreData>().value(d => d.track_count).sort(null)
    const arc = d3.arc<d3.PieArcDatum<GenreData>>().innerRadius(0).outerRadius(radius)

    const title = chartContainer.selectAll('.chart-title').data([null])
    title
        .join('text')
        .attr('class', 'chart-title')
        .attr('x', size.value.width / 2)
        .attr('y', 25)
        .attr('text-anchor', 'middle')
        .style('font-size', '14px')
        .style('font-weight', 'bold')
        .style('fill', '#333')
        .text(`Genre Distribution (Pie View)`)

    // Create pie group
    const pieGroup = chartContainer
        .selectAll('.pie-group')
        .data([null])
        .join('g')
        .attr('class', 'pie-group')
        .attr('transform', `translate(${centerX}, ${centerY})`)

    // Add slices with smooth transitions
    pieGroup
        .selectAll('.pie-slice')
        .data(pie(data.value), (d: any, i: number) => i)
        .join(
            enter => enter.append('path')
                .attr('class', 'pie-slice')
                .attr('fill', (d: any) => colorScale(d.data.avg_popularity))
                .attr('stroke', '#fff')
                .attr('stroke-width', 2)
                .attr('d', (d: any) => arc(d) as string)
                .attr('opacity', 0.8)
                .attr('fill-opacity', 0)
                .transition()
                .duration(600)
                .attr('fill-opacity', 1),
            update => update
                .transition()
                .duration(600)
                .attr('d', (d: any) => arc(d) as string)
                .attr('fill', (d: any) => colorScale(d.data.avg_popularity))
                .attr('fill-opacity', 1),
            exit => exit
                .transition()
                .duration(300)
                .attr('fill-opacity', 0)
                .remove()
        )
        .style('cursor', 'pointer')
        .on('mouseover', function(event, d: any) {
            d3.select(this)
                .transition()
                .duration(200)
                .attr('opacity', 1)
                .attr('stroke-width', 3)
            
            pieGroup.selectAll('.pie-slice')
                .transition()
                .duration(200)
                .attr('opacity', (slice: any) => slice === d ? 1 : 0.2)
            
            // Create tooltip using HTML div (similar to Treemap)
            const tooltip = d3.select('body').append('div')
                .attr('class', 'pie-tooltip-box')
                .style('position', 'absolute')
                .style('background-color', '#f5f5f5')
                .style('border', '2px solid #1DB954')
                .style('border-radius', '6px')
                .style('padding', '12px')
                .style('font-size', '12px')
                .style('box-shadow', '0 2px 8px rgba(0,0,0,0.15)')
                .style('z-index', '10000')
                .style('pointer-events', 'none')
                .style('left', (event.pageX + 10) + 'px')
                .style('top', (event.pageY - 10) + 'px')
                .html(`
                    <div style="font-weight: bold; color: #1DB954; margin-bottom: 8px; font-size: 13px;">${d.data.genre}</div>
                    <div style="color: #333; line-height: 1.4;">
                        <div>🎵 ${d.data.track_count} tracks</div>
                        <div>⭐ Avg Pop: ${d.data.avg_popularity.toFixed(1)}</div>
                    </div>
                `)
        })
        .on('mouseout', function() {
            d3.select(this)
                .transition()
                .duration(200)
                .attr('opacity', 0.8)
                .attr('stroke-width', 2)
            
            pieGroup.selectAll('.pie-slice')
                .transition()
                .duration(200)
                .attr('opacity', 1)
            
            d3.selectAll('.pie-tooltip-box').remove()
        })
        
        .on('mousemove', function(event) {
            d3.selectAll('.pie-tooltip-box')
                .style('left', (event.pageX + 10) + 'px')
                .style('top', (event.pageY - 10) + 'px')
        })

    // Add genre labels on pie slices
    pieGroup
        .selectAll('.pie-label')
        .data(pie(data.value), (d: any, i: number) => i)
        .join(
            enter => enter.append('text')
                .attr('class', 'pie-label')
                .attr('transform', (d: any) => {
                    const centroid = arc.centroid(d)
                    // Adjust distance based on slice size to avoid overlap
                    const distance = d.endAngle - d.startAngle > Math.PI / 6 ? 1.5 : 1.8
                    return `translate(${centroid[0] * distance}, ${centroid[1] * distance})`
                })
                .attr('text-anchor', 'middle')
                .attr('dominant-baseline', 'middle')
                .style('font-size', '10px')
                .style('font-weight', 'bold')
                .style('fill', '#333')
                .style('pointer-events', 'none')
                .style('opacity', 0)
                .style('text-shadow', '0 0 3px rgba(255,255,255,0.8)')
                .transition()
                .duration(600)
                .style('opacity', 1),
            update => update
                .transition()
                .duration(600)
                .attr('transform', (d: any) => {
                    const centroid = arc.centroid(d)
                    const distance = d.endAngle - d.startAngle > Math.PI / 6 ? 1.5 : 1.8
                    return `translate(${centroid[0] * distance}, ${centroid[1] * distance})`
                }),
            exit => exit
                .transition()
                .duration(300)
                .style('opacity', 0)
                .remove()
        )
        .text((d: any) => {
            const percentage = ((d.value / d3.sum(pie(data.value), (item: any) => item.value)) * 100).toFixed(0)
            // Only show full label for larger slices
            if (d.endAngle - d.startAngle > Math.PI / 6) {
                return `${d.data.genre} ${percentage}%`
            } else {
                return `${percentage}%`
            }
        })

    // Add legend
    addLegend(chartContainer, colorScale)
}

function addLegend(chartContainer: d3.Selection<SVGSVGElement, unknown, HTMLElement, any>, colorScale: d3.ScaleLinear<string, string>) {
    const legendX = size.value.width - 160
    const legendY = margin.top + 20
    const legendHeight = 80

    // Legend background
    const legend = chartContainer.selectAll('.legend').data([null])
    legend
        .join('rect')
        .attr('class', 'legend')
        .attr('x', legendX - 10)
        .attr('y', legendY - 10)
        .attr('width', 150)
        .attr('height', legendHeight)
        .attr('fill', '#f5f5f5')
        .attr('opacity', 0.95)
        .attr('stroke', '#1DB954')
        .attr('stroke-width', 1)

    // Legend title
    const legendTitle = chartContainer.selectAll('.legend-title').data([null])
    legendTitle
        .join('text')
        .attr('class', 'legend-title')
        .attr('x', legendX + 65)
        .attr('y', legendY + 10)
        .attr('text-anchor', 'middle')
        .style('font-size', '11px')
        .style('font-weight', 'bold')
        .style('fill', '#1DB954')
        .text('Avg Popularity')

    // Legend color bar
    const legendBar = chartContainer.selectAll('.legend-bar').data([null])
    legendBar
        .join('rect')
        .attr('class', 'legend-bar')
        .attr('x', legendX)
        .attr('y', legendY + 25)
        .attr('width', 120)
        .attr('height', 12)
        .attr('fill', 'url(#genreGradient)')

    // Add gradient
    const defs = chartContainer.selectAll('defs').data([null]).join('defs')
    const gradient = defs.selectAll('#genreGradient').data([null]).join('linearGradient')
        .attr('id', 'genreGradient')
        .attr('x1', '0%')
        .attr('x2', '100%')

    gradient.selectAll('stop').remove()
    gradient
        .append('stop')
        .attr('offset', '0%')
        .attr('stop-color', '#282828')

    gradient
        .append('stop')
        .attr('offset', '100%')
        .attr('stop-color', '#1DB954')

    // Legend labels
    const lowLabel = chartContainer.selectAll('.legend-low').data([null])
    lowLabel
        .join('text')
        .attr('class', 'legend-low')
        .attr('x', legendX)
        .attr('y', legendY + 50)
        .style('font-size', '9px')
        .style('fill', '#333')
        .text('Low (0)')

    const highLabel = chartContainer.selectAll('.legend-high').data([null])
    highLabel
        .join('text')
        .attr('class', 'legend-high')
        .attr('x', legendX + 120)
        .attr('y', legendY + 50)
        .attr('text-anchor', 'end')
        .style('font-size', '9px')
        .style('fill', '#333')
        .text('High (100)')
}

function initChart() {
    const chartContainer = d3.select('#artists-bar-svg')
    chartContainer.selectAll('*').remove()

    if (chartType.value === 'bar') {
        drawBarChart()
    } else {
        drawPieChart()
    }
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

watch(
    () => chartType.value,
    () => {
        if (canRender.value) {
            initChart()
        }
    }
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
    <div class="chart-wrapper" style="width: 100%; height: 100%; display: flex; flex-direction: column; background: #ffffff;">
        <!-- Toggle Button -->
        <div style="display: flex; gap: 8px; padding: 12px 16px; border-bottom: 1px solid #ddd;">
            <button 
                @click="chartType = 'bar'"
                :style="{
                    padding: '6px 14px',
                    background: chartType === 'bar' ? '#1DB954' : '#f5f5f5',
                    color: chartType === 'bar' ? 'white' : '#333',
                    border: chartType === 'bar' ? '2px solid #1DB954' : '1px solid #ddd',
                    borderRadius: '4px',
                    cursor: 'pointer',
                    fontSize: '12px',
                    fontWeight: 'bold',
                    transition: 'all 200ms ease'
                }"
            >
                📊 Bar Chart
            </button>
            <button 
                @click="chartType = 'pie'"
                :style="{
                    padding: '6px 14px',
                    background: chartType === 'pie' ? '#1DB954' : '#f5f5f5',
                    color: chartType === 'pie' ? 'white' : '#333',
                    border: chartType === 'pie' ? '2px solid #1DB954' : '1px solid #ddd',
                    borderRadius: '4px',
                    cursor: 'pointer',
                    fontSize: '12px',
                    fontWeight: 'bold',
                    transition: 'all 200ms ease'
                }"
            >
                🥧 Pie Chart
            </button>
        </div>
        <!-- Chart Container -->
        <div ref="container" style="flex: 1; overflow: hidden;">
            <svg id="artists-bar-svg" width="100%" height="100%"></svg>
        </div>
    </div>
</template>

<style scoped>
.chart-wrapper {
    width: 100%;
    height: 100%;
    background: #ffffff;
    display: flex;
    flex-direction: column;
}

button {
    transition: all 200ms ease;
}

button:hover {
    transform: translateY(-1px);
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

button:active {
    transform: translateY(0);
}
</style>
