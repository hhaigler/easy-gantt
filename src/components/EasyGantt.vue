<script lang="ts">

// Gantt item props
export type GanttItem = {
    id: string;
    name: string;
    start: Date;
    end: Date;
    predecessor?: string;
    status?: GanttItemStatus;
}
export type RowOptions = {
    height: number;
    margin: number;
}
const defaultRowOptions: RowOptions = {
    height: 1.5,
    margin: .25,
}
export type ColumnOptions = {
    width: number;
}
const defaultColumnOptions: ColumnOptions = {
    width: 2
}
export type TableOptions = {
    units: 'px' | 'rem' | 'em' | '%';
    fill: string;
    minHeight: number;
    maxHeight: number;
}
const defaultTableOptions: TableOptions = {
    units: 'rem',
    fill: '#07549F',
    minHeight: 10,
    maxHeight: 20
}

const resetTimeOfDate = (date: Date): Date => {
    date.setHours(0, 0, 0, 0)
    return date
}

export enum GanttItemStatus {
    NORMAL,
    WARNING,
    URGENT,
    DONE
}

const dateToUTC = (date: Date): Date => {
    return new Date(date.getTime() + (date.getTimezoneOffset() * 60000))
}

const getMondayOfWeek = (date: Date): Date => {
    const day = date.getDay() > 0 ? date.getDay() : 7  // ADJUST FOR SUN FIRST DAY OF WEEK
    const mon = date.getDate() - day + 1;

    var copiedDate = new Date(date.toDateString());
    const monday = dateToUTC(new Date(copiedDate.setDate(mon)));

    return monday;
}

</script>

<script lang="ts" setup>
import { computed, nextTick, ref, toRefs, watch } from 'vue';

const props = withDefaults(defineProps<{
    items: GanttItem[],
    rowOptions?: RowOptions,
    tableOptions?: TableOptions,
    columnOptions?: ColumnOptions
}>(), { items: () => [], rowOptions: () => defaultRowOptions, tableOptions: () => defaultTableOptions, columnOptions: () => defaultColumnOptions })
const { items, rowOptions, tableOptions, columnOptions } = toRefs(props)

const selectedItem = ref<GanttItem>({} as GanttItem)
const emit = defineEmits<{
    select: [item: GanttItem]
}>()
const selectItem = (item: GanttItem, scroll?: boolean, emitItem?: boolean) => {
    selectedItem.value = item
    if (scroll) {
        scrollIntoView(item.id, 'smooth')
    }
    if (emitItem) {
        emit('select', item)
    }
}

const selectItemByID = (itemID: string) => {
    const item = items.value.find(item => item.id == itemID)
    if (!!!item) return
    selectItem(item, true)
}

defineExpose({
    selectItemByID
})

const zoomLevel = ref(1)
enum ZoomLevel {
    'day',
    'week',
    'month'
}
const columnLevel = ref<ZoomLevel>(ZoomLevel.week)
const setZoomLevel = (level: ZoomLevel) => {
    columnLevel.value = level
    if (level == ZoomLevel.day) {
        zoomLevel.value = 1
    } else if (level == ZoomLevel.week) {
        zoomLevel.value = 1.5
    } else {
        zoomLevel.value = 5
    }
}
setZoomLevel(ZoomLevel.week)

const orderedItems = computed(() => {
    return [...items.value].sort((a, b) => {
        return a.start.getTime() - b.start.getTime()
    })
})
const startDate = computed(() => {
    if (orderedItems.value.length == 0) return new Date()
    const start = orderedItems.value[0].start
    if (columnLevel.value == ZoomLevel.day) {
        const startOffset = 7 * 24 * 60 * 60 * 1000
        return new Date(start.getTime() - startOffset)
    } else if (columnLevel.value == ZoomLevel.week) {
        const startOffset = 21 * 24 * 60 * 60 * 1000
        const date = new Date(start.getTime() - startOffset)
        const firstMonday = getMondayOfWeek(date)
        return firstMonday
    } else {
        const date = new Date(start)
        return new Date(date.getFullYear(), date.getMonth() - 1, 1)
    }
})
const endDate = computed(() => {
    if (orderedItems.value.length == 0) return new Date()
    const end = orderedItems.value[orderedItems.value.length - 1].end
    if (columnLevel.value == ZoomLevel.day) {
        const endOffset = 7 * 24 * 60 * 60 * 1000
        return new Date(end.getTime() + endOffset)
    } else if (columnLevel.value == ZoomLevel.week) {
        const date = new Date(end)
        return new Date(date.getFullYear(), date.getMonth() + 2, -1)
    } else {
        const date = new Date(end)
        return new Date(date.getFullYear(), date.getMonth() + 2, -1)
    }
})
const monthDiff = (d1: Date, d2: Date): number => {
    var months;
    months = (d2.getFullYear() - d1.getFullYear()) * 12;
    months -= d1.getMonth();
    months += d2.getMonth();
    return months <= 0 ? 0 : months;
}
const getXOffset = (date: Date, columnWidth: number, unit: string, offset?: number): string => {
    offset = offset || 0
    const start = startDate.value
    const diff = resetTimeOfDate(date).getTime() - resetTimeOfDate(start).getTime()
    const diffDays = diff / (1000 * 60 * 60 * 24)
    if (columnLevel.value == ZoomLevel.day) {
        const x = (diffDays * columnWidth * zoomLevel.value + offset)
        return `${x}${unit}`
    } else if (columnLevel.value == ZoomLevel.week) {
        const diffWeeks = Math.floor(diffDays / 7)
        const x = (diffWeeks * columnWidth * zoomLevel.value + offset)
        return `${x}${unit}`
    } else {
        const diffMonths = monthDiff(start, date)
        const daysInMonth = new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate()
        const dayOfMonth = date.getDate()
        const monthOffset = (dayOfMonth - 1) / daysInMonth
        const x = ((diffMonths + monthOffset) * columnWidth * zoomLevel.value + offset)
        return `${x}${unit}`
    }
}
const getYOffset = (index: number, rowHeight: number, rowMargin: number, unit: string, offset?: number): string => {
    offset = offset || 0
    const y = ((rowHeight * index + (rowMargin * index)) + offset)
    return `${y}${unit}`
}

const getWidth = (start: Date, end: Date, columnWidth: number, units: string, offset?: number): string => {
    offset = offset || 1
    if (columnLevel.value == ZoomLevel.day) {
        const diff = end.getTime() - start.getTime()
        const diffDays = diff / (1000 * 60 * 60 * 24)
        const width = (diffDays + offset) * columnWidth * zoomLevel.value
        return `${width}${units}`
    } else if (columnLevel.value == ZoomLevel.week) {
        const diff = end.getTime() - start.getTime()
        const diffDays = Math.round(diff / (1000 * 60 * 60 * 24 * 7))
        const width = (diffDays + offset) * columnWidth * zoomLevel.value
        return `${width}${units}`
    } else {
        offset--
        const diff = end.getTime() - start.getTime()
        const diffDays = diff / (1000 * 60 * 60 * 24)
        const diffMonths = diffDays / 30
        const width = (diffMonths + offset) * columnWidth * zoomLevel.value
        return `${width}${units}`
    }
}
const ganttHeight = computed((): string => {
    const height = (rowOptions.value.height * orderedItems.value.length) + (rowOptions.value.margin * orderedItems.value.length) + 1
    return Math.max(tableOptions.value.minHeight, height) + tableOptions.value.units
})

const datesBetweenStartAndEnd = computed((exlcudeWeekends?: boolean): Date[] => {
    const dates: Date[] = []
    const currentDate = new Date(startDate.value)
    while (currentDate <= endDate.value) {
        if (exlcudeWeekends) {
            if (currentDate.getDay() != 0 && currentDate.getDay() != 6) {
                dates.push(new Date(currentDate))
            }
        } else {
            dates.push(new Date(currentDate))
        }
        currentDate.setDate(currentDate.getDate() + 1)
    }
    return dates
})
const dateToMonthName = (date: Date): string => {
    const monthNames = [
        "Jan", "Feb", "Mar",
        "Apr", "May", "Jun", "Jul",
        "Aug", "Sep", "Oct",
        "Nov", "Dec"
    ]
    return monthNames[date.getMonth()]
}

type HeaderItem = {
    date: Date;
    width: number;
}

const minorHeaderItems = computed<HeaderItem[]>((): HeaderItem[] => {
    const items: HeaderItem[] = []
    let currentDate = new Date(startDate.value)
    if (columnLevel.value == ZoomLevel.day) {
        while (currentDate <= endDate.value) {
            items.push({
                date: new Date(currentDate),
                width: columnOptions.value.width * zoomLevel.value
            })
            currentDate.setDate(currentDate.getDate() + 1)
        }
    } else if (columnLevel.value == ZoomLevel.week) {
        // pad end date 
        // offset in milliseconds
        while (currentDate <= endDate.value) {
            items.push({
                date: new Date(currentDate),
                width: columnOptions.value.width * zoomLevel.value * 7
            })
            currentDate.setDate(currentDate.getDate() + 7)
        }
    } else if (columnLevel.value == ZoomLevel.month) {
        while (currentDate <= endDate.value) {
            const month = new Date(currentDate.getFullYear(), currentDate.getMonth() + 1, 0)
            const daysInMonth = month.getDate()
            if (month.getMonth() == startDate.value.getMonth() && month.getFullYear() == startDate.value.getFullYear()) {
                items.push({
                    date: new Date(currentDate),
                    width: columnOptions.value.width * zoomLevel.value * (daysInMonth - (startDate.value.getDate() - 1))
                })
            } else if (month.getMonth() == endDate.value.getMonth() && month.getFullYear() == endDate.value.getFullYear()) {
                items.push({
                    date: new Date(currentDate),
                    width: columnOptions.value.width * zoomLevel.value * endDate.value.getDate()
                })
            } else {
                items.push({
                    date: new Date(currentDate),
                    width: columnOptions.value.width * zoomLevel.value * daysInMonth
                })
            }
            currentDate.setMonth(currentDate.getMonth() + 1)
        }
    }
    return items
})

const majorHeaderItems = computed<HeaderItem[]>((): HeaderItem[] => {
    const items: HeaderItem[] = []
    const dates = minorHeaderItems.value
    for (let index = 0; index < dates.length; index++) {
        const date = dates[index];
        if (items.length == 0 || items[items.length - 1].date.getMonth() != date.date.getMonth()) {
            items.push({
                date: new Date(date.date),
                width: 1
            })
        } else {
            items[items.length - 1].width += 1
        }
    }
    return items
})

const getTextWidth = (inputText: string, fontSize: number, fontFamily: string, unit: 'px' | 'rem' | 'em' | '%', padding: number): string => {
    const canvas = document.createElement("canvas");
    const context = canvas.getContext("2d");
    if (!context) {
        return '0px'
    }
    const font = `${fontSize}${unit} ${fontFamily}`
    context.font = font;
    const width = context.measureText(inputText).width;
    const formattedWidth = Math.ceil(width) + padding + "px";

    return formattedWidth
}

const text = (item: GanttItem, fontSize: number) => {
    return getTextWidth(item.name, fontSize, 'poppins', tableOptions.value.units, 12)
}

let itemRefs: { [key: string]: any } = {}
const setItemRef = (id: string, el: any) => {
    if (!!!el) return
    if (el) {
        itemRefs[id] = el
    }
}

const scrollIntoView = async (id: string, behavior: "smooth" | "instant") => {
    nextTick(() => {
        const el = itemRefs[id]
        if (!!!el) return
        el.scrollIntoView({ behavior: behavior, block: 'nearest', inline: 'nearest' });
    })
}

const getColorFromStatus = (status: GanttItemStatus | undefined): string => {
    if (status == undefined) return tableOptions.value.fill
    if (status == GanttItemStatus.NORMAL) {
        return tableOptions.value.fill
    } else if (status == GanttItemStatus.WARNING) {
        return '#e6ab22'
    } else if (status == GanttItemStatus.URGENT) {
        return '#D22525'
    } else if (status == GanttItemStatus.DONE) {
        return '#079F61'
    }
    return tableOptions.value.fill
}

// Follow selectedScheduleItem and scroll to it when it changes
watch(selectedItem, (newVal, oldVal) => {
    if (newVal.id != oldVal.id || newVal.start == oldVal.start && newVal.end == newVal.end) return
    scrollIntoView(newVal.id, 'smooth')
}, { deep: true })

</script>

<template>
    <div id="gantt-chart">
        <div id="zoom-level-selector">
            <button @click="setZoomLevel(ZoomLevel.day)" :class="{ 'selected': columnLevel == ZoomLevel.day }">d</button>
            <button @click="setZoomLevel(ZoomLevel.week)" :class="{ 'selected': columnLevel == ZoomLevel.week }">w</button>
            <button @click="setZoomLevel(ZoomLevel.month)"
                :class="{ 'selected': columnLevel == ZoomLevel.month }">m</button>
        </div>
        <div id="gantt-chart-wrapper" :style="{ 'max-height': tableOptions.maxHeight + tableOptions.units }">
            <div id="gantt-header" v-show="items.length > 0">
                <div>
                    <div v-for="item in majorHeaderItems" :key="item.date.toString()" class="header-item"
                        :style="{ width: item.width * columnOptions.width * zoomLevel + tableOptions.units }">
                        <div class="header-item-sticky">
                            {{ dateToMonthName(item.date) + ' ' + item.date.getFullYear() }}
                        </div>
                    </div>
                </div>
                <div v-show="columnLevel != ZoomLevel.month">
                    <div v-for="item in minorHeaderItems" :key="item.date.toString()" class="header-item"
                        :style="{ width: columnOptions.width * zoomLevel + tableOptions.units }">
                        {{ item.date.getDate() }}
                    </div>
                </div>
            </div>
            <svg :height="ganttHeight" width="100%" xmlns="http://www.w3.org/2000/svg">
                <g> <!-- TODAY -->
                    <line y1="0rem" :y2="ganttHeight" :x1="getXOffset(new Date(), columnOptions.width, tableOptions.units)"
                        :x2="getXOffset(new Date(), columnOptions.width, tableOptions.units)" :stroke="tableOptions.fill"
                        width="2px"></line>
                </g>
                <g v-for="(_, index) in datesBetweenStartAndEnd "> <!-- DATE LINES -->
                    <line y1="0rem" :y2="ganttHeight" :x1="index * columnOptions.width * zoomLevel + tableOptions.units"
                        :x2="index * columnOptions.width * zoomLevel + tableOptions.units" stroke="#ccc" width="1px"></line>
                </g>
                <g v-for="(item, index) in items" @click="selectItem(item, false, true)">
                    <rect class="text-wrapper" :x="getXOffset(item.start, columnOptions.width, tableOptions.units, .3)"
                        :y="getYOffset(index, rowOptions.height, rowOptions.margin, tableOptions.units, .25)" rx="5" ry="5"
                        :width="text(item, rowOptions.height / 2)" :height="rowOptions.height / 1.5 + tableOptions.units"
                        :fill="'#ccc'">
                    </rect>
                    <rect :x="0" class="hightlight"
                        :y="getYOffset(index, rowOptions.height, rowOptions.margin, tableOptions.units, -.15)"
                        v-show="selectedItem.id == item.id"
                        :width="getWidth(startDate, endDate, columnOptions.width, tableOptions.units)"
                        :height="rowOptions.height + .3 + tableOptions.units"
                        :fill="getColorFromStatus(item.status) + '19'" />
                    <rect class="bar" :x="getXOffset(item.start, columnOptions.width, tableOptions.units)"
                        :y="getYOffset(index, rowOptions.height, rowOptions.margin, tableOptions.units)"
                        :width="getWidth(item.start, item.end, columnOptions.width, tableOptions.units)"
                        :height="rowOptions.height + tableOptions.units" :fill="getColorFromStatus(item.status)" rx="5"
                        ry="5" :ref="e => setItemRef(item.id, e)" />
                    <text :x="getXOffset(item.start, columnOptions.width, tableOptions.units, .75)"
                        :y="getYOffset(index, rowOptions.height, rowOptions.margin, tableOptions.units, rowOptions.height / 1.5)"
                        :font-size="rowOptions.height / 2 + tableOptions.units" fill="#fff">
                        {{ item.name }}
                    </text>
                    <svg v-show="item.predecessor" width="20" height="20" viewBox="0 0 24 24" fill="none"
                        stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"
                        class="feather feather-corner-left-up"
                        :x="getXOffset(item.start, columnOptions.width, tableOptions.units, -1.5)"
                        :y="getYOffset(index, rowOptions.height, rowOptions.margin, tableOptions.units)">
                        <polyline points="14 9 9 4 4 9"></polyline>
                        <path d="M20 20h-7a4 4 0 0 1-4-4V4"></path>
                    </svg>
                </g>
            </svg>
        </div>
    </div>
</template>

<style scoped>
#gantt-chart {
    position: relative;
    background: inherit;
}

#gantt-chart-wrapper {
    width: max-content;
    overflow: auto;
    width: 100%;

    border-radius: .5rem;
    background: white;
}

#gantt-header {
    width: max-content;
    position: sticky;
    top: 0;
    background: white;
    min-width: 100%;
}

#zoom-level-selector {
    position: absolute;
    bottom: 2rem;
    left: 1rem;
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    z-index: 1;
    background-color: white;
    border-radius: .75rem;
    box-shadow: 0 0 5px rgba(0, 0, 0, .2);
    padding: .25rem .25rem;
    gap: .25rem
}

#zoom-level-selector button {
    padding: .3rem .5rem;
    border: none;
    background: none;
    cursor: pointer;
    outline: none;
    font-size: .8rem;
    color: #222;
    border-radius: .6rem;
}

#zoom-level-selector button:hover {
    background: #eee;
}

#zoom-level-selector button.selected {
    background: #07549F;
    color: white
}

.header-item {
    color: #222;
    text-align: center;
    border-left: 1px solid #ccc;
    display: inline-flex;
    justify-content: center;
    align-items: center;
}

.header-item:first-child {
    border: none;
}

.header-item-sticky {
    position: sticky;
    left: 0;
    right: 0;
    padding: 0 1rem;
    background: white;
    font-size: .9rem;
    white-space: nowrap;
}

svg {
    overflow: visible;
}

rect,
text {
    cursor: pointer;
}

.bar {
    scroll-margin: 8rem 8rem;
}
</style>