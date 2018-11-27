<template lang="pug">
  .vuecal__cell(:class="[cssClass, splits.length ? 'splitted' : '']" :style="cellStyles")
    .vuecal__cell-content(:class="splits.length && `vuecal__cell-split ${splits[i - 1].class}`" v-for="i in (splits.length || 1)")
      .split-label(v-if="splits.length" v-html="splits[i - 1].label")
      div(v-if="content" v-html="content")
      div(v-else)
        .vuecal__no-event(v-if="!cellEvents.length") {{ texts.noEvent }}
        .vuecal__event(:class="{ [event.class]: true, background: event.background, overlapping: event.overlapping, overlapped: event.overlapped, split2: event.split2, split3: event.split3, 'split-middle': event.splitm }"
                       v-else
                       v-for="(event, j) in (splits.length ? splitEvents[i] : cellEvents)" :key="j"
                       :style="event.startTime && { top: `${event.top}px`, height: ($parent.resizeEvent || 1) && `${event.height}px` }")
          .vuecal__event-title(v-if="event.title") {{ event.title }}
          .vuecal__event-time(v-if="event.startTime")
            | {{ event.startTime }}
            span(v-if="event.endTime") &nbsp;- {{ event.endTime }}
          .vuecal__event-content(v-if="event.content" v-html="event.content")
          .vuecal__event-resize-handle(v-if="event.startTime && $parent.resizableEvents"
                                       @mousedown="$parent.resizableEvents && $parent.time && onMouseDown($event, event)"
                                       @touchstart="$parent.resizableEvents && $parent.time && onMouseDown($event, event)")
</template>

<script>
export default {
  props: {
    cssClass: {
      type: String,
      default: ''
    },
    date: {
      type: Date,
      required: true
    },
    events: {
      type: Array,
      default: () => []
    },
    content: {
      type: [String, Number],
      default: ''
    },
    splits: {
      type: Array,
      default: () => []
    }
  },
  data: () => ({
    splitEvents: [],
    // For each event, compare with others if overlapping.
    // Keep track of what is already check in this indexed array not to redo the check.
    comparedEvents: {}
  }),

  methods: {
    eventPosition (event) {
      if (!event.startTime) return
      let minutesFromTop = event.startTimeMinutes - this.$parent.timeFrom
      const top = Math.round(minutesFromTop * this.timeCellHeight / this.timeStep)

      minutesFromTop = event.endTimeMinutes - this.$parent.timeFrom
      const bottom = Math.round(minutesFromTop * this.timeCellHeight / this.timeStep)

      event.top = top
      event.height = bottom - top
      event.minHeight = event.height
    },

    checkOverlappingEvents (event, comparedEvents) {
      (this.splits.length && event.split ? this.splitEvents[event.split] : this.events).forEach(event2 => {
        // Don't compare with itself or with already compared item.
        const eventsAreDifferent = event.id !== event2.id
        const eventNeverCompared = comparedEvents[event.id].indexOf(event2.id) === -1 && (comparedEvents[event2.id] || []).indexOf(event.id) === -1
        const eventIsBackground = event.background || event2.background

        if (eventsAreDifferent && eventNeverCompared && !eventIsBackground) {
          const event1startsFirst = event.startTimeMinutes < event2.startTimeMinutes
          const event1overlapsEvent2 = event1startsFirst && event.endTimeMinutes > event2.startTimeMinutes
          const event2overlapsEvent1 = !event1startsFirst && event2.endTimeMinutes > event.startTimeMinutes

          if (event1overlapsEvent2 || event2overlapsEvent1) {
            event.overlapped = event1startsFirst
            event.overlapping = !event1startsFirst

            event2.overlapped = !event1startsFirst
            event2.overlapping = event1startsFirst

            // If up to 3 events start at the same time.
            if (event.startTimeMinutes === event2.startTimeMinutes) {
              event.split3 = event.split2
              event.split2 = true
              event2.split3 = event2.split2
              event2.split2 = true
              event2.splitm = event.split2 && !event2.class.middle
            }
          }
        }

        comparedEvents[event.id].push(event2.id)
      })

      return comparedEvents
    },

    resizeEvent (event, amount) {
      event.height = Math.max(event.originalHeight + amount, 10)

      this.updateEndTimeOnResize(event)
    },

    updateEndTimeOnResize (event) {
      const bottom = event.top + event.height
      const endTime = bottom / this.timeCellHeight + this.timeFrom / 60
      const hours = parseInt(endTime)
      const minutes = parseInt((endTime - hours) * 60)
      event.endTimeMinutes = endTime * 60
      event.endTime = `${hours}:${(minutes < 10 ? '0' : '') + minutes}`
    },

    onMouseDown (e, event) {
      const start = 'ontouchstart' in window ? e.touches[0].clientY : e.clientY
      event.originalHeight = event.height
      this.$parent.resizeEvent = { start, event, resizeHandler: this.resizeEvent }
    }
  },

  computed: {
    texts () {
      return this.$parent.texts
    },
    timeCellHeight () {
      return parseInt(this.$parent.timeCellHeight)
    },
    timeFrom () {
      return parseInt(this.$parent.timeFrom)
    },
    timeStep () {
      return parseInt(this.$parent.timeStep)
    },
    cellStyles () {
      return { minWidth: `${this.$parent.minCellWidth}px` || null }
    },
    cellEvents () {
      let comparedEvents = {}
      // eslint-disable-next-line
      this.splitEvents = []

      return this.events.map(event => {
        comparedEvents[event.id] = []

        // Only for splits.
        if (this.splits.length && event.split) {
          // eslint-disable-next-line
          if (!this.splitEvents[event.split]) this.splitEvents[event.split] = []
          // eslint-disable-next-line
          this.splitEvents[event.split].push(event)
        }

        if (!event.background) {
          comparedEvents = this.checkOverlappingEvents(event, comparedEvents)
        }

        this.eventPosition(event)

        return event
      })
    }
  }
}
</script>

<style lang="scss">
.vuecal__cell {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  position: relative;
  .vuecal--month-view &,
  .vuecal--week-view &,
  .vuecal--day-view & {
    width: 14.2857%;
  }

  .vuecal--hide-weekends.vuecal--month-view &,
  .vuecal--hide-weekends.vuecal--week-view &,
  .vuecal--hide-weekends.vuecal--day-view & {
    width: 20%;
  }

  .vuecal--years-view & {width: 20%;}
  .vuecal--year-view & {width: 33.33%;}
  .vuecal--day-view & {flex: 1;}

  .vuecal--click-to-navigate & {cursor: pointer;}
  .vuecal--view-with-time &,
  .vuecal--week-view.vuecal--no-time &,
  .vuecal--day-view.vuecal--no-time & {display: block;}

  &.splitted {
    flex-direction: row;
    display: flex;
  }

  .vuecal__cell-split {
    display: flex;
    flex-grow: 1;
    flex-direction: column;
    height: 100%;
    position: relative;
  }

  &:before {
    position: absolute;
    top: 0;
    left: 0;
    right: -1px;
    bottom: -1px;
    border: 1px solid #ddd;
    content: '';

    .vuecal--day-view & {left: 1px;right: 0;bottom: 0;}
    .vuecal--view-with-time.vuecal--day-view & {left: 0;}
  }

  &.today,
  &.current {
    background-color: rgba(240, 240, 255, 0.4);
    z-index: 1;
  }

  &.selected {
    background-color: rgba(235, 255, 245, 0.4);
    z-index: 2;

    // .vuecal--day-view &:before {background: none;border: 1px solid rgba(235, 255, 245, 0.4);}
    .vuecal--day-view & {background: none;}
  }

  &.out-of-scope {
    color: #ccc;
  }
}

.vuecal--split-days.vuecal--week-view .vuecal__cell.splitted {
  overflow: hidden;
}

.vuecal__no-event {
  padding-top: 1em;
  color: #aaa;
  user-select: none;
}

.vuecal__event {
  color: #666;
  background-color: #f8f8f8;
  position: relative;
  z-index: 1;

  // Reactivate user selection in events.
  .vuecal__cell & * {user-select: auto;}

  .vuecal--view-with-time & {
    position: absolute;
    left: 0;
    right: 0;
    overflow: hidden;

    // &:hover {height: auto !important;}
  }

  &.overlapped {right: 20%;}
  &.overlapping:not(.split2):not(.split3) {left: 30%;box-shadow: 0 0 5px rgba(#000, 0.2);}
  &.overlapped.split2 {right: 50%;}
  &.overlapping.split2 {left: 50%;}
  &.overlapped.split3 {right: 66.66%;}
  &.overlapping.split3 {left: 66.66%;}
  &.overlapping.split3.split-middle {left: 33.33%;right: 33.33%;}
  &.background {z-index: 0;}
}

.vuecal__event-resize-handle {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 1em;
  background-color: rgba(255, 255, 255, 0.3);
  opacity: 0;
  transition: 0.3s;
  cursor: ns-resize;

  .vuecal__event:hover & {opacity: 1;}
}
</style>