<script setup>
import { Timeline } from 'vis-timeline'
import { DataSet } from 'vis-data'
import $ from 'jQuery'

import { onMounted } from 'vue'

onMounted(() => {
  console.log('Mounting')
  var items = new DataSet()
  var opNames = ['Lizzie', 'Sukhitha', 'Juan', 'Roma']
  var delayNames = ['Hauling', 'Meal break', 'Digging', 'Doing Burn Outs']

  // create visualization
  var container = document.getElementById('visualization')
  var options = {
    start: '2024-01-01',
    end: '2024-01-02',
    tooltip: {
      followMouse: true
    },
    stack: false,
    multiselect: true,
    snap: function (date, scale, step) {
      var hour = 60 * 60 * 1000
      return date
    },
    groupOrder: 'content', // groupOrder can be a property name or a sorting function
    editable: {
      add: true,
      updateTime: true,
      updateGroup: false,
      remove: true
    },
    onAdd: function (item, callback) {
      let date = new Date(item.start.getTime() + 30 * 60000)
      item.end = date

      if (
        !(
          (item.group.includes('_operator') && opNames.includes(item.content)) ||
          (item.group.includes('_delay') && delayNames.includes(item.content))
        )
      ) {
        return
      }

      if (opNames.includes(item.content)) {
        setSubGroup(item, 'operator')
      } else {
        setSubGroup(item, 'delay')
      }

      // Check if dropped within a delay
      var overlappingItem = items.get({
        filter: (x) => {
          return (
            x.subgroup == item.subgroup &&
            x.group == item.group &&
            Date.parse(x.start) < Date.parse(item.start) &&
            Date.parse(x.end) > Date.parse(item.start)
          )
        }
      })[0]
      // If dropped within another delay, either replace or split
      if (overlappingItem != null) {
        console.log('OverlapppingItem', JSON.stringify(overlappingItem))
        if (overlappingItem.content == item.content) return
        let duration = Math.round(
          (Date.parse(overlappingItem.end) - Date.parse(overlappingItem.start)) / (1000 * 60)
        )
        if (duration < 120) {
          console.log('duration greater than 2 hours')
          overlappingItem.content = item.content
          setItemTooltip(overlappingItem)
          items.update(overlappingItem)
          return
        } else {
          console.log('duration less than 2 hours')
          let clonedOverlappingItem = JSON.parse(JSON.stringify(overlappingItem))
          overlappingItem.end = item.start
          setItemTooltip(overlappingItem)
          items.update(overlappingItem)
          const guid = generateGUID()
          clonedOverlappingItem.start = item.end
          clonedOverlappingItem.id = guid
          console.log('cloned item', JSON.stringify(clonedOverlappingItem))
          if (Date.parse(clonedOverlappingItem.start) < Date.parse(clonedOverlappingItem.end)) {
            setItemTooltip(clonedOverlappingItem)
            items.add(clonedOverlappingItem)
          }
          callback(item)
        }
      }

      // if not dropped within a delay, fill gap if appropriate
      let allItems = items.get({
        filter: (x) => {
          return x.subgroup == item.subgroup && x.group == item.group
        }
      })
      const pastDateObjects = allItems.filter((obj) => Date.parse(obj.end) < Date.parse(item.start))
      const futureDateObjects = allItems.filter(
        (obj) => Date.parse(obj.start) > Date.parse(item.start)
      )
      let nextDateObject
      let pastDateObject
      if (pastDateObjects.length > 0) {
        pastDateObject = pastDateObjects.reduce(function (r, a) {
          return r.end > a.end ? r : a
        })
      }
      if (futureDateObjects.length > 0) {
        nextDateObject = futureDateObjects.reduce(function (r, a) {
          return r.start < a.start ? r : a
        })
      }

      if (pastDateObject != null && nextDateObject != null) {
        var minDiff = Math.round((nextDateObject.start - pastDateObject.end) / 60000)
        if (minDiff < 120) {
          item.end = nextDateObject.start
          item.start = pastDateObject.end
        }
      }

      let duration = Math.round((item.end - item.start) / (1000 * 60))
      setItemTooltip(item)
      callback(item)
    },
    onMove: function (item, callback) {
      setItemTooltip(item)
      callback(item)
    }
  }

  var timeline = new Timeline(container)

  timeline.setOptions(options)
  setGroups(timeline)
  timeline.setItems(items)

  $('#assetSelection').on('change', function (e) {
    setGroups(timeline)
  })
  var dragItems = document.querySelectorAll('.items .item')
  for (var i = dragItems.length - 1; i >= 0; i--) {
    var item = dragItems[i]
    item.addEventListener('dragstart', handleDragStart.bind(this), false)
  }
})

function setGroups(timelineObject) {
  var selected = new Array()
  $('#assetSelection option:selected').each(function () {
    selected.push($(this).val())
  })
  var groups = new DataSet()
  for (var g = 0; g < selected.length; g++) {
    groups.add({
      id: selected[g],
      content: selected[g],
      nestedGroups: [selected[g] + '_operator', selected[g] + '_delays']
    })
    groups.add({
      id: selected[g] + '_operator',
      content: 'Operator'
    })
    groups.add({
      id: selected[g] + '_delays',
      content: 'Delays'
    })
  }
  timelineObject.setGroups(groups)
}

//NORMAL DRAG
function handleDragStart(event) {
  console.log('Drag started')
  event.dataTransfer.effectAllowed = 'move'
  const guid = generateGUID()
  var item = {
    id: guid,
    type: 'range',
    content: event.target.innerHTML
  }
  event.dataTransfer.setData('text', JSON.stringify(item))
}

function setItemTooltip(item) {
  let duration = Math.round((item.end - item.start) / (1000 * 60))
  item.title = `<span>Device: ${item.group}</span><br>
        <span>Delay: ${item.content}</span><br>
        <span>Start: ${item.start}</span><br>
        <span>End: ${item.end}</span><br>
        <span>Duration: ${duration} mins</span>`
}

function setSubGroup(item, subgroupName) {
  item.subgroup = subgroupName
  item.className = subgroupName
}

function generateGUID() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
    const r = (Math.random() * 16) | 0
    const v = c === 'x' ? r : (r & 0x3) | 0x8
    return v.toString(16)
  })
}
</script>

<template>
  <div class="container-fluid">
    <div class="row timeline-controls">
      <div class="col-4">
        <h3>Device Selection</h3>
        <select
          id="assetSelection"
          class="form-select"
          multiple
          aria-label="multiple select example"
        >
          <option selected value="RD336">RD336</option>
          <option value="RD337">RD337</option>
          <option value="RD338">RD338</option>
          <option value="WL405">WL405</option>
          <option value="DZ799">DZ799</option>
        </select>
      </div>
      <div class="col-4">
        <h3>Delays</h3>
        <ul class="items">
          <li draggable="true" class="item">Hauling</li>
          <li draggable="true" class="item">Meal break</li>
          <li draggable="true" class="item">Digging</li>
          <li draggable="true" class="item">Doing Burn Outs</li>
        </ul>
      </div>
      <div class="col-4">
        <h3>Operators</h3>
        <ul class="items">
          <li draggable="true" class="item">Sukhitha</li>
          <li draggable="true" class="item">Lizzie</li>
          <li draggable="true" class="item">Juan</li>
          <li draggable="true" class="item">Roma</li>
        </ul>
      </div>
    </div>
    <div class="row timeline-view">
      <div id="visualization"></div>
    </div>
  </div>
</template>

<style>
body,
html {
  font-family: arial, sans-serif;
  font-size: 18px;
}

.col-3 {
  text-align: -webkit-center;
}

#visualization {
  box-sizing: border-box;
  width: 100%;
  height: 250px;
  margin: 10px;
}

.vis-item.delay {
  background-color: gold;
  border-color: grey;
}

.vis-item.vis-selected.delay {
  background-color: gold;
  border-color: black;
}

.vis-item.operator {
  background-color: lightskyblue;
  border-color: grey;
}

.vis-item.vis-selected.operator {
  /* custom colors for selected orange items */
  background-color: lightskyblue;
  border-color: black;
}

.timeline-controls {
  margin: 30px;
}

.form-select {
  margin: 10px;
  width: 200px;
}

div.vis-editable,
div.vis-editable.vis-selected {
  /* custom styling for editable items... */
}

div.vis-readonly,
div.vis-readonly.vis-selected {
  /* custom styling for readonly items... */
  background-color: #ff4500;
  border-color: red;
  color: white;
}

li.item {
  list-style: none;
  width: 150px;
  color: #1a1a1a;
  background-color: #d5ddf6;
  border: 1px solid #97b0f8;
  border-radius: 2px;
  margin-bottom: 5px;
  padding: 5px 12px;
}

li.object-item {
  list-style: none;
  width: 150px;
  color: #1a1a1a;
  background-color: #d5ddf6;
  border: 1px solid #97b0f8;
  border-radius: 2px;
  margin-bottom: 5px;
  padding: 5px 12px;
}
</style>
