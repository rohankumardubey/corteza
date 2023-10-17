<template>
  <b-tab :title="$t('report.label')">
    <b-row>
      <b-col>
        <b-form-group
          :label="$t('report.label')"
          label-class="text-primary"
        >
          <c-input-select
            v-model="options.reportID"
            :options="reportOptions"
            :reduce="o => o.reportID"
            append-to-body
          />
        </b-form-group>
      </b-col>

      <b-col
        v-if="selectedReport && scenarioOptions.length > 1"
      >
        <b-form-group
          :label="$t('report.scenario.label')"
          label-class="text-primary"
        >
          <c-input-select
            v-model="options.scenarioID"
            :options="scenarioOptions"
            :reduce="o => o.scenarioID"
            append-to-body
          />
        </b-form-group>
      </b-col>
    </b-row>

    <b-form-group
      v-if="selectedReport"
      :label="$t('report.element.label')"
      :description="$t('report.element.description')"
      label-class="text-primary"
    >
      <c-input-select
        v-model="options.elementID"
        :options="elementOptions"
        :reduce="reduceElement"
        :get-option-label="getOptionLabel"
        append-to-body
      >
        <template #option="option">
          <div v-if="option.options">
            <strong>{{ option.label }}</strong>

            <ul class="list-unstyled">
              <li
                v-for="subOption in option.options"
                :key="subOption.name"
                class="ml-2"
              >
                {{ subOption.name }}
              </li>
            </ul>
          </div>
          <div v-else>
            {{ option.name }}
          </div>
        </template>
      </c-input-select>
    </b-form-group>
  </b-tab>
</template>
<script>
import base from '../base'
import { NoID } from '@cortezaproject/corteza-js'

export default {
  i18nOptions: {
    namespaces: 'block',
  },

  extends: base,

  data () {
    return {
      reports: [],
    }
  },

  computed: {
    reportOptions () {
      return [
        { reportID: NoID, label: this.$t('general:label.none') },
        ...this.reports.map(r => ({ ...r, label: r.meta.name || r.handle })),
      ]
    },

    selectedReport () {
      const { reportID = NoID } = this.options

      if (reportID !== NoID) {
        return this.reports.find(r => r.reportID === reportID)
      }

      return undefined
    },

    scenarioOptions () {
      const { scenarios = [] } = this.selectedReport || {}

      return [
        { scenarioID: NoID, label: this.$t('general:label.none') },
        ...scenarios,
      ]
    },

    elementOptions () {
      const elements = [{ elementID: NoID, name: this.$t('general:label.none') }]

      if (this.selectedReport) {
        this.selectedReport.blocks.forEach(b => {
          elements.push({
            label: b.title || `${this.$t('general:label.block')} ${b.key}`,
            options: b.elements.map(({ elementID, name, kind }) => ({ elementID, name: name || kind })),
          })
        })
      }

      return elements
    },
  },

  watch: {
    'options.blockID' () {
      this.options.elementID = NoID
    },
  },

  beforeDestroy () {
    this.setDefaultValues()
  },

  created () {
    this.fetchReports()
  },

  methods: {
    fetchReports () {
      this.$SystemAPI.reportList()
        .then(({ set = [] }) => {
          this.reports = set
        })
        .catch(this.toastErrorHandler(this.$t('notification:report.listFetchFailed')))
    },

    setDefaultValues () {
      this.reports = []
    },

    reduceElement (o) {
      if (o.elementID === '0') {
        return o.elementID
      }
      const element = this.elementOptions.find(e => e.label === o.label)
      if (element) {
        return element.options[0].elementID
      }
    },

    getOptionLabel (o) {
      if (o.elementID === '0') {
        return o.name
      }
      const element = this.elementOptions.find(e => e.label === o.label)
      if (element) {
        return element.options[0].name
      }
    },
  },
}
</script>
