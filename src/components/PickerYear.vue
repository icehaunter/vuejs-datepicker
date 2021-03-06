<template>
  <div :class="[calendarClass, 'vdp-datepicker__calendar']" v-show="showYearView" :style="calendarStyle" @mousedown.prevent>
    <slot name="beforeCalendarHeader"></slot>
    <header>
      <span
        @click="isRtl ? nextDecade() : previousDecade()"
        class="prev"
        :class="{'disabled': isLeftNavDisabled}">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 6 8">
          <g fill="none" fill-rule="evenodd">
            <path stroke="none" d="M-9 16V-8h24v24z"/>
            <path stroke-linecap="round" stroke-linejoin="round" d="M5 0L1 4l4 4"/>
          </g>
        </svg>
      </span>
      <span>{{ getPageDecade }}</span>
      <span
        @click="isRtl ? previousDecade() : nextDecade()"
        class="next"
        :class="{'disabled': isRightNavDisabled}">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 6 8">
          <g fill="none" fill-rule="evenodd">
            <path stroke="none" d="M15-8v24H-9V-8z"/>
            <path stroke-linecap="round" stroke-linejoin="round" d="M1 8l4-4-4-4"/>
          </g>
        </svg>
      </span>
    </header>
    <div class="cell-block years">
      <span
        class="cell year"
        v-for="year in years"
        :key="year.timestamp"
        :class="{ 'selected': year.isSelected, 'disabled': year.isDisabled }"
        @click.stop="selectYear(year)"><span>{{ year.year }}</span></span>
    </div>
  </div>
</template>
<script>
import { makeDateUtils, rtlLangs, langYearSuffix } from '../utils/DateUtils'
export default {
  props: {
    showYearView: Boolean,
    selectedDate: Date,
    pageDate: Date,
    pageTimestamp: Number,
    disabledDates: Object,
    highlighted: Object,
    calendarClass: [String, Object, Array],
    calendarStyle: Object,
    language: Object,
    allowedToShowView: Function,
    useUtc: Boolean
  },
  watch: {
    language (newLanguage) {
      this.utils = makeDateUtils(this.useUtc, newLanguage)
    },
    useUtc (newUtc) {
      this.utils = makeDateUtils(newUtc, this.language)
    }
  },
  computed: {
    isRtl () {
      return rtlLangs.indexOf(this.language) !== -1
    },
    years () {
      const d = this.pageDate
      const years = []
      // set up a new date object to the beginning of the current 'page'7
      const dObj = this.useUtc
        ? new Date(Date.UTC(Math.floor(d.getUTCFullYear() / 10) * 10, d.getUTCMonth(), d.getUTCDate()))
        : new Date(Math.floor(d.getFullYear() / 10) * 10, d.getMonth(), d.getDate(), d.getHours(), d.getMinutes())
      for (let i = 0; i < 10; i++) {
        years.push({
          year: this.utils.getFullYear(dObj),
          timestamp: dObj.getTime(),
          isSelected: this.isSelectedYear(dObj),
          isDisabled: this.isDisabledYear(dObj)
        })
        this.utils.setFullYear(dObj, this.utils.getFullYear(dObj) + 1)
      }
      return years
    },
    /**
     * @return {String}
     */
    getPageDecade () {
      const decadeStart = Math.floor(this.utils.getFullYear(this.pageDate) / 10) * 10
      const decadeEnd = decadeStart + 9
      const yearSuffix = langYearSuffix[this.language] || ''
      return `${decadeStart} - ${decadeEnd}${yearSuffix}`
    },
    /**
     * Is the left hand navigation button disabled?
     * @return {Boolean}
     */
    isLeftNavDisabled () {
      return this.isRtl
        ? this.isNextDecadeDisabled(this.pageTimestamp)
        : this.isPreviousDecadeDisabled(this.pageTimestamp)
    },
    /**
     * Is the right hand navigation button disabled?
     * @return {Boolean}
     */
    isRightNavDisabled () {
      return this.isRtl
        ? this.isPreviousDecadeDisabled(this.pageTimestamp)
        : this.isNextDecadeDisabled(this.pageTimestamp)
    }
  },
  data () {
    const constructedDateUtils = makeDateUtils(this.useUtc, this.language)
    return {
      utils: constructedDateUtils
    }
  },
  methods: {
    selectYear (year) {
      if (year.isDisabled) {
        return false
      }
      this.$emit('selectYear', year)
    },
    changeYear (incrementBy) {
      const date = this.pageDate
      this.utils.setFullYear(date, this.utils.getFullYear(date) + incrementBy)
      this.$emit('changedDecade', date)
    },
    previousDecade () {
      if (this.isPreviousDecadeDisabled()) {
        return false
      }
      this.changeYear(-10)
    },
    isPreviousDecadeDisabled () {
      if (!this.disabledDates || !this.disabledDates.to) {
        return false
      }
      const disabledYear = this.utils.getFullYear(this.disabledDates.to)
      const lastYearInPreviousPage = Math.floor(this.utils.getFullYear(this.pageDate) / 10) * 10 - 1
      return disabledYear > lastYearInPreviousPage
    },
    nextDecade () {
      if (this.isNextDecadeDisabled()) {
        return false
      }
      this.changeYear(10)
    },
    isNextDecadeDisabled () {
      if (!this.disabledDates || !this.disabledDates.from) {
        return false
      }
      const disabledYear = this.utils.getFullYear(this.disabledDates.from)
      const firstYearInNextPage = Math.ceil(this.utils.getFullYear(this.pageDate) / 10) * 10
      return disabledYear < firstYearInNextPage
    },

    /**
     * Whether the selected date is in this year
     * @param {Date}
     * @return {Boolean}
     */
    isSelectedYear (date) {
      return this.selectedDate && this.utils.getFullYear(this.selectedDate) === this.utils.getFullYear(date)
    },
    /**
     * Whether a year is disabled
     * @param {Date}
     * @return {Boolean}
     */
    isDisabledYear (date) {
      let disabledDates = false
      if (typeof this.disabledDates === 'undefined' || !this.disabledDates) {
        return false
      }

      if (typeof this.disabledDates.to !== 'undefined' && this.disabledDates.to) {
        if (this.utils.getFullYear(date) < this.utils.getFullYear(this.disabledDates.to)) {
          disabledDates = true
        }
      }
      if (typeof this.disabledDates.from !== 'undefined' && this.disabledDates.from) {
        if (this.utils.getFullYear(date) > this.utils.getFullYear(this.disabledDates.from)) {
          disabledDates = true
        }
      }

      if (typeof this.disabledDates.customPredictor === 'function' && this.disabledDates.customPredictor(date)) {
        disabledDates = true
      }

      return disabledDates
    }
  }
}
// eslint-disable-next-line
;
</script>
