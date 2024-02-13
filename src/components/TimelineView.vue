<script setup>
import { Timeline } from 'vis-timeline'
import { DataSet } from 'vis-data'
import $ from 'jQuery'
import { onMounted } from 'vue'
import TimelineDataService from '../services/timelineDataService'

const timelineDataService = new TimelineDataService()

var timeline
var rightClickedItemId
var itemToCopy
var copyDetails

var opNames = ['Lizzie', 'Sukhitha', 'Juan', 'Roma']
var delayNames = ['Hauling', 'Meal break', 'Digging', 'Doing Burn Outs']
onMounted(() => {
  var root = document.getElementsByTagName('html')[0] // '0' to assign the first (and only `HTML` tag)
  var items = new DataSet(JSON.parse(localStorage.getItem('myData')))

  root.setAttribute('class', 'light-theme')
  console.log('Mounting')
  //var items = new DataSet()

  // create visualization
  var container = document.getElementById('visualization')
  var options = {
    min: '2024-01-01T00:00:00.000Z',
    start: '2024-01-01T00:00:00.000Z',
    max: '2024-01-01T08:00:00.000Z',
    end: '2024-01-01T08:00:00.000Z',
    tooltip: {
      followMouse: true
    },
    maxHeight: 700,
    verticalScroll: true,
    stack: false,
    preferZoom: true,
    multiselect: true,
    snap: function (date, scale, step) {
      var fiveMins = 5 * 60 * 1000
      return Math.round(date / fiveMins) * fiveMins
    },
    groupOrder: function (a, b) {
      return a.value - b.value
    },
    editable: {
      add: true,
      updateTime: true,
      updateGroup: false,
      remove: true
    },

    onAdd: function (item, callback) {
      console.log('Adding item')
      if (
        !(
          (item.group.includes('_operator') && opNames.includes(item.content)) ||
          (item.group.includes('_delay') && delayNames.includes(item.content))
        )
      )
        return

      let date = new Date(item.start.getTime() + 30 * 60000)
      item.end = date
      var parentGroup = item.group.split('_')[0]
      var groupCategory = item.group.split('_')[1]

      setSubGroup(item, groupCategory)
      //if in shared timeline, replicate change to other timelines
      if (parentGroup == 'Selected Devices') {
        console.log('Dropped item on shared timeline')
        const matchingGroups = timeline.groupsData.get({
          filter: (x) => {
            let groupCat = x.id.split('_')[1]
            let groupParent = x.id.split('_')[0]
            return groupCat == groupCategory && groupParent != 'Selected Devices'
          }
        })
        console.log(matchingGroups)
        matchingGroups.forEach((x) => {
          let clonedItem = JSON.parse(JSON.stringify(item))
          clonedItem.group = x.id
          setSubGroup(clonedItem, groupCategory)
          setItemTooltip(clonedItem)
          clonedItem.id = generateGUID()
          timeline.itemsData.add(clonedItem)
          console.log('Added item from shared timeline', clonedItem)
        })
        console.log(item)
        item.id = generateGUID()
        setItemTooltip(item)
      }

      // Check if dropped within a delay
      var overlappingItem = timeline.itemsData.get({
        filter: (x) => {
          return (
            x.group == item.group &&
            x.type != 'background' &&
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
          timeline.itemsData.update(overlappingItem)
          return
        } else {
          console.log('duration less than 2 hours')
          let clonedOverlappingItem = JSON.parse(JSON.stringify(overlappingItem))
          overlappingItem.end = item.start
          setItemTooltip(overlappingItem)
          timeline.itemsData.update(overlappingItem)
          const guid = generateGUID()
          clonedOverlappingItem.start = item.end
          clonedOverlappingItem.id = guid
          console.log('cloned item', JSON.stringify(clonedOverlappingItem))
          if (Date.parse(clonedOverlappingItem.start) < Date.parse(clonedOverlappingItem.end)) {
            setItemTooltip(clonedOverlappingItem)
            timeline.itemsData.add(clonedOverlappingItem)
          }
          callback(item)
        }
      }

      // if not dropped within a delay, fill gap if appropriate
      let allItems = timeline.itemsData.get({
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

      setItemTooltip(item)
      callback(item)
    },
    onMove: function (item, callback) {
      console.log('Moving item')
      setItemTooltip(item)
      callback(item)
    }
  }

  timeline = new Timeline(container)
  var groups = new DataSet()
  timeline.setOptions(options)
  setGroups(timeline, groups)
  //var items = new DataSet(localStorage.getItem('myData'))
  timeline.setItems(items)

  $('#assetSelection').on('change', function (e) {
    setGroups(timeline, groups)
  })

  $('#nextShift').on('click', function (e) {
    changeTimelineBoundaries(timeline, 8)
  })

  $('#previousShift').on('click', function (e) {
    changeTimelineBoundaries(timeline, -8)
  })

  timeline.on('dragover', function (props) {
    console.log('Dragging over ', props)
  })

  timeline.on('contextmenu', function (props) {
    console.log(props)
    props.event.preventDefault()

    if (props.item != null) {
      rightClickedItemId = props.item
      // Show contextmenu
      $('.custom-menu')
        .finish()
        .toggle(100)
        // In the right position (the mouse)
        .css({
          top: props.event.pageY + 'px',
          left: props.event.pageX + 'px'
        })
    } else {
      $('.custom-menu-timeline')
        .finish()
        .toggle(100)
        // In the right position (the mouse)
        .css({
          top: props.event.pageY + 'px',
          left: props.event.pageX + 'px'
        })
      copyDetails = props
    }
  })
  // If the document is clicked somewhere
  $(document).bind('mouseup', function (e) {
    $('.custom-menu-timeline').hide(100)
    $('.custom-menu').hide(100)
  })
  var dragItems = document.querySelectorAll('.items .item')
  for (var i = dragItems.length - 1; i >= 0; i--) {
    var item = dragItems[i]
    item.addEventListener('dragstart', handleDragStart.bind(this), false)
    item.addEventListener('dragend', handleDragEnd.bind(this), false)
  }
})

function setGroups(timelineObject, groupDataSet) {
  var selected = new Array()
  $('#assetSelection option:selected').each(function () {
    selected.push($(this).val())
  })
  groupDataSet = new DataSet()
  if (selected.length > 1) addGroup(groupDataSet, 'Selected Devices')

  for (var g = 0; g < selected.length; g++) {
    addGroup(groupDataSet, selected[g])
  }

  timelineObject.setGroups(groupDataSet)
  addSummaryRows()
}

function addGroup(groupDataSet, groupName) {
  let className = 'main-group'
  if (groupName == 'Selected Devices') className = 'selected-devices-group'
  groupDataSet.add({
    id: groupName,
    content: groupName,
    nestedGroups: [groupName + '_operator', groupName + '_delays'],
    value: 1,
    className: className
  })
  groupDataSet.add({
    id: groupName + '_operator',
    content: 'Operator',
    className: 'operator-row'
  })
  groupDataSet.add({
    id: groupName + '_delays',
    content: 'Delays',
    className: 'delay-row'
  })
}

function addSummaryRows() {
  var summaryDiv =
    '<div class="summaryStuff"><img src="../src/assets/yellow-square.png"> 1.4h <img src="../src/assets/green-square.png"> 5.6h</div>'
  $('.vis-foreground .vis-group.main-group').append(summaryDiv)
  console.log($('.vis-foreground .vis-group.main-group'))
}

//NORMAL DRAG
function handleDragStart(event) {
  let tileName = event.target.innerHTML
  // if tilename =
  if (opNames.includes(tileName)) {
    $('.operator-row').attr('highlight', true)
  } else {
    $('.delay-row').attr('highlight', true)
  }

  console.log('Drag started')
  event.dataTransfer.effectAllowed = 'move'
  const guid = generateGUID()
  var item = {
    id: guid,
    type: 'range',
    content: tileName
  }
  event.dataTransfer.setData('text', JSON.stringify(item))
}

function handleDragEnd(event) {
  console.log('Drag end')
  $('.delay-row').attr('highlight', false)
  $('.operator-row').attr('highlight', false)
}

function changeTimelineBoundaries(timeline, hours) {
  let start = new Date(timeline.options.start)
  let end = new Date(timeline.options.end)
  start.setHours(start.getHours() + hours)
  end.setHours(end.getHours() + hours)
  console.log(end.toISOString())
  const options = {
    max: end,
    min: start,
    start: start,
    end: end
  }
  timeline.setOptions(options)
}

function setItemTooltip(item) {
  let duration = Math.round((Date.parse(item.end) - Date.parse(item.start)) / (1000 * 60))
  var parentGroup = item.group.split('_')[0]

  item.title = `<span>Device: ${parentGroup}</span><br>
        <span>Delay: ${item.content}</span><br>
        <span>Start: ${item.start}</span><br>
        <span>End: ${item.end}</span><br>
        <span>Duration: ${duration} mins</span>`
}

function setSubGroup(item, subgroupName) {
  //item.subgroup = subgroupName
  item.className = subgroupName
}

function generateGUID() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
    const r = (Math.random() * 16) | 0
    const v = c === 'x' ? r : (r & 0x3) | 0x8
    return v.toString(16)
  })
}

function saveTimelineData() {
  localStorage.setItem('myData', JSON.stringify(timeline.itemsData.get()))
  console.log(localStorage.getItem('myData'))
}

function resetData() {
  var defaultData = timelineDataService.getTimelineData()
  timeline.setItems(defaultData)
}

function deleteSelected() {
  console.log(timeline.getSelection())
  let selectedIds = timeline.getSelection()
  timeline.itemsData.remove(selectedIds)
}

function deleteRightClicked() {
  timeline.itemsData.remove(rightClickedItemId)
}

function copyItem() {
  itemToCopy = rightClickedItemId
}

function pasteItem() {
  console.log(copyDetails)
  let templateItem = timeline.itemsData.get(itemToCopy)
  let clonedItem = JSON.parse(JSON.stringify(templateItem))
  let duration = Math.round(Date.parse(clonedItem.end) - Date.parse(clonedItem.start))
  clonedItem.start = copyDetails.time
  clonedItem.end = new Date(Date.parse(copyDetails.time) + duration)
  clonedItem.group = copyDetails.group
  clonedItem.id = generateGUID()
  console.log(clonedItem)
  timeline.itemsData.add(clonedItem)
}

function editItem() {
  var item = timeline.itemsData.get(rightClickedItemId)
  alert('EDITING ITEM: ' + JSON.stringify(item))
}
</script>

<template>
  <div class="container-fluid">
    <div class="row timeline-controls">
      <div class="col-3">
        <h3>Date selection</h3>
        <button type="button" class="iv-btn iv-btn-secondary iv-btn-tertiary" id="nextShift">
          Next Shift
        </button>
        <button type="button" class="iv-btn iv-btn-secondary iv-btn-tertiary" id="previousShift">
          Previous Shift
        </button>
      </div>

      <div class="col-3">
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
      <div class="col-3">
        <h3>Delays</h3>
        <ul class="items">
          <li draggable="true" class="item">Hauling</li>
          <li draggable="true" class="item">Meal break</li>
          <li draggable="true" class="item">Digging</li>
          <li draggable="true" class="item">Doing Burn Outs</li>
        </ul>
      </div>
      <div class="col-3">
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
  <button type="button" @click="saveTimelineData" class="iv-btn iv-btn-primary" id="nextShift">
    Save Data
  </button>
  <button
    type="button"
    @click="resetData"
    class="iv-btn iv-btn-secondary iv-btn-tertiary"
    id="nextShift"
  >
    Reset Data
  </button>
  <button
    type="button"
    class="iv-btn iv-btn-secondary iv-btn-tertiary"
    @click="deleteSelected"
    id="nextShift"
  >
    Delete Selected
  </button>
  <ul class="custom-menu">
    <li data-action="first" @click="deleteRightClicked">Delete</li>
    <li data-action="second" @click="copyItem">Copy</li>
    <li data-action="third" @click="editItem">Edit Item</li>
  </ul>

  <ul class="custom-menu-timeline">
    <li data-action="second" @click="pasteItem">Paste</li>
  </ul>
</template>

<style>
body,
html {
  font-family: arial, sans-serif;
  font-size: 18px;
}

.vis-group {
  background: #ffffff;
}

.main-group {
  background: #c6c6c6;
}

.selected-devices-group {
  background: #ffc600;
}

.col-3 {
  text-align: -webkit-center;
}

#visualization {
  box-sizing: border-box;
  width: 100%;
  margin: 10px;
  overflow: auto;
  border-radius: 4px;
  border: var(--faded-grey) solid 1px;
  background: var(--light-background-color);
}

.operator-row[highlight='true'],
.delay-row[highlight='true'] {
  background: #d7f3ff85;
}

.vis-item.vis-background.working {
  background-color: rgba(105, 255, 98, 0.7);
}

.vis-item.vis-background.idle {
  background-color: rgba(255, 251, 0, 0.7);
}

.summaryStuff {
  position: absolute;
  right: 16px;
  top: 6px;
}

.vis-item.delays {
  background-color: gold;
  border-color: grey;
}

.vis-item.vis-selected.delays {
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

/* CSS3 */

/* The whole thing */
.custom-menu,
.custom-menu-timeline {
  display: none;
  z-index: 1000;
  position: absolute;
  overflow: hidden;
  border: 1px solid #ccc;
  white-space: nowrap;
  font-family: sans-serif;
  background: #fff;
  color: #333;
  border-radius: 5px;
  padding: 0;
}

/* Each of the items in the list */
.custom-menu li,
.custom-menu-timeline li {
  padding: 8px 12px;
  cursor: pointer;
  list-style-type: none;
  transition: all 0.3s ease;
  user-select: none;
}

.custom-menu li:hover,
.custom-menu-timeline li:hover {
  background-color: #def;
}
</style>
