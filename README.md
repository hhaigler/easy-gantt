# Easy Gantt 

`easy-gantt` is a Vue.js component for creating simple, clean and customizable Gantt charts.  
This project is very simple right now. The chart is currently ***read-only***.  
There is no ability to drag and drop or change durations

## Installation

You can install `easy-gantt` via npm:

```sh
npm install easy-gantt
```

## Usage
Import { Gantt } directly into components.  
Import any other type if necessary / using Typescript.  
```javascript
import { Gantt, GanttItem, GanttItemStatus } from 'easy-gantt'
import 'easy-gantt/dist/style.css'
```
Populate items list
```javascript
const items = [
   {  
      id: '1',
      name: 'Task 1',
      start: new Date(2022, 0, 1),
      end: new Date(2022, 0, 5)
   },
   { 
      id: '2',
      name: 'Task 2',
      start: new Date(2022, 0, 6),
      end: new Date(2022, 0, 10)
   },
   // more items...
]
```
Pass items as prop
```html
<gantt :items="items" />
```

  
**DONE!**  


## Props
| Prop | Type | Required/Optional | Description |
| ---- | ---- | ---------------- | ----------- |
| `items` | [`GanttItem[]`](#ganttitem) | Required | The items to be displayed in the Gantt chart. Each item should be an object that matches the `GanttItem` type. |
| `rowOptions`  | [`RowOptions`](#rowoptions) | Optional | Options for customizing the rows in the Gantt chart. |
| `tableOptions`  | [`TableOptions`](#tableoptions) | Optional | Options for customizing the table part of the Gantt chart. |
| `columnOptions`  | [`ColumnOptions`](#columnoptions) | Optional | Options for customizing the columns in the Gantt chart. |

## Types

### GanttItem
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `string` | - | The unique identifier for the item. |
| `name` | `string` | - | The name of the item. |
| `start` | `Date` | - | The start date of the item. |
| `end` | `Date` | - | The end date of the item. |
| `predecessor` | `string` | `undefined` | The id of the predecessor item. |
| `status` | [`GanttItemStatus`](#ganttitemstatus) | `undefined` | The status of the item. |

### RowOptions
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `height` | `number` | `1.5` | The height of the rows in the Gantt chart. |
| `margin` | `number` | `0.25` | The margin of the rows in the Gantt chart. |

### ColumnOptions
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `width` | `number` | `2` | The width of the columns in the Gantt chart. |

### TableOptions
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `units` | `'px' \| 'rem' \| 'em' \| '%'` | `'rem'` | The units for the table dimensions. |
| `fill` | `string` | `'#07549F'` | The fill color for the table. |
| `minHeight` | `number` | `10` | The minimum height for the table. |
| `maxHeight` | `number` | `20` | The maximum height for the table. |

### GanttItemStatus
The `GanttItemStatus` enum has the following values:

| Value | Description |
| ----- | ----------- |
| `NORMAL` | The item is in a normal state. |
| `WARNING` | The item is in a warning state. |
| `URGENT` | The item is in an urgent state. |
| `DONE` | The item is in a done state. |