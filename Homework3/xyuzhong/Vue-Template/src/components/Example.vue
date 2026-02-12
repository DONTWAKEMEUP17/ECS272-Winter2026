
<script setup lang="ts">
import * as d3 from 'd3'
import dataFromJson from '../../data/demo.json'
import axios from 'axios'
import { isEmpty, debounce } from 'lodash'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'

import { Bar, ComponentSize, Margin } from '../types'

// A "extends" B means A inherits the properties and methods from B.
interface CategoricalBar extends Bar {
    category: string
}

const bars = ref<CategoricalBar[]>([])
const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 50, right: 20, top: 20, bottom: 80 }

const barContainer = ref<HTMLElement | null>(null)

// Re-render when there is data and the size is set
const canRender = computed(() => !isEmpty(bars.value) && size.value.width > 0 && size.value.height > 0)

// Initialize data (runs once)
if (!isEmpty(dataFromJson)) {
    // dataFromJson is expected to be an object with a `data` array like before
    // Keep this simple and synchronous for local example data
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    bars.value = (dataFromJson as any).data ?? []
}

async function read() {
    const readFromCSV = await d3.csv('../../data/demo.csv', (d: d3.DSVRowString<'category' | 'value'>) => {
        return { category: d.category, value: +d.value } as { category: string; value: number }
    })
    bars.value = readFromCSV as unknown as CategoricalBar[]
}

function onResize() {
    const target = barContainer.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function initChart() {
    const chartContainer = d3.select('#bar-svg')

    const yExtents = d3.extent(bars.value.map((d) => d.value as number)) as [number, number]
    const xCategories: string[] = [...new Set(bars.value.map((d) => d.category as string))]

    const xScale = d3
        .scaleBand<string>()
        .rangeRound([margin.left, size.value.width - margin.right])
        .domain(xCategories)
        .padding(0.1)

    const yScale = d3
        .scaleLinear()
        .range([size.value.height - margin.bottom, margin.top])
        .domain([0, yExtents[1]])

    chartContainer.append('g').attr('transform', `translate(0, ${size.value.height - margin.bottom})`).call(d3.axisBottom(xScale))

    chartContainer.append('g').attr('transform', `translate(${margin.left}, 0)`).call(d3.axisLeft(yScale))

    chartContainer
        .append('g')
        .attr('transform', `translate(${margin.left / 2}, ${size.value.height / 2}) rotate(-90)`)
        .append('text')
        .text('Value')
        .style('font-size', '.8rem')

    chartContainer
        .append('g')
        .attr('transform', `translate(${size.value.width / 2 - margin.left}, ${size.value.height - margin.top - 10})`)
        .append('text')
        .text('Categories')
        .style('font-size', '.8rem')

    chartContainer
        .append('g')
        .selectAll('rect')
        .data<CategoricalBar>(bars.value)
        .join('rect')
        .attr('x', (d) => (xScale(d.category) as number))
        .attr('y', (d) => yScale(d.value) as number)
        .attr('width', xScale.bandwidth())
        .attr('height', (d) => Math.abs(yScale(0) - yScale(d.value)))
        .attr('fill', 'teal')

    chartContainer
        .append('g')
        .append('text')
        .attr('transform', `translate(${size.value.width / 2}, ${size.value.height - margin.top + 5})`)
        .attr('dy', '0.5rem')
        .style('text-anchor', 'middle')
        .style('font-weight', 'bold')
        .text('Distribution of Demo Data')
}

// Watch for data and size changes to re-render
watch(
    [bars, size],
    ([barsVal, sizeVal]) => {
        if (!isEmpty(barsVal) && sizeVal.width > 0 && sizeVal.height > 0) {
            d3.select('#bar-svg').selectAll('*').remove()
            initChart()
        }
    },
    { deep: true }
)

// Debounced resize handler needs to be the same reference to remove later
const debouncedOnResize = debounce(onResize, 100)

onMounted(() => {
    window.addEventListener('resize', debouncedOnResize)
    // initial sizing
    onResize()
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', debouncedOnResize)
})

</script>

<!-- "ref" registers a reference to the HTML element so that we can access it via the reference in Vue.  -->
<!-- We use flex (d-flex) to arrange the layout-->
<template>
    <div class="chart-container d-flex" ref="barContainer">
        <svg id="bar-svg" width="100%" height="100%"></svg>
    </div>
</template>

<style scoped>
.chart-container {
    height: 100%;
}
</style>

