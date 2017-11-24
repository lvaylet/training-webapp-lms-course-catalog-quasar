<template>
  <q-layout>
    <q-toolbar slot="header" class="toolbar" style="height: 100px">
      <q-toolbar-title>
        Internal Course Catalog
        <div slot="subtitle">for Talend employees</div>
      </q-toolbar-title>
    </q-toolbar>

    <div class="layout-view">
      <div class="layout-padding">

        <div class="row sm-gutter">
          <div class="col-md-12 col-xl-9">

            <!-- Filters -->
            <div class="row sm-gutter">
              <div class="col-sm-12 col-md-4 col-lg-4 col-xl-3">
                <q-select
                  v-model="filters.product"
                  :options="products"
                  float-label="Product"
                  :filter="true"
                  :clearable="true"
                />
              </div>
              <div class="col-sm-12 col-md-4 col-lg-4 col-xl-3">
                <q-select
                  v-model="filters.learningPath"
                  :options="learningPaths"
                  float-label="Learning Path"
                  :filter="true"
                  :clearable="true"
                  class="col-3"
                />
              </div>
              <div class="col-sm-12 col-md-4 col-lg-4 col-xl-3">
                <q-select
                  v-model="filters.level"
                  :options="levels"
                  float-label="Level"
                  :filter="true"
                  :clearable="true"
                  class="col-3"
                />
              </div>
              <div class="col-sm-12 col-md-4 col-lg-4 col-xl-3">
                <q-select
                  v-model="filters.recommendedFor"
                  :options="recommendations"
                  float-label="Recommended For"
                  :filter="true"
                  :clearable="true"
                  class="col-3"
                />
              </div>
            </div>

            <q-data-table
              :data="tableDataFiltered"
              :config="config"
              :columns="columns"
              @refresh="loading"
              @rowclick="handleRowClick"
            >
            </q-data-table>
          </div>

          <div class="col-md-12 col-xl-3">
            <q-card>
              <q-card-title>
                <h5 v-if="Object.keys(courseSelected).length === 0">
                  Course Details
                </h5>
                <h5 v-else>
                  {{ courseSelected.name }}
                </h5>
              </q-card-title>
              <q-card-main>
                <div v-if="Object.keys(courseSelected).length === 0">
                  Please select a course to display its details...
                </div>
                <div v-else>
                  <p>{{ courseSelected.description }}</p>
                  <p>Learning Paths: <strong>{{ displayListAsString(courseSelected[getCustomFieldName('learningPaths')]) }}</strong></p>
                  <p>Recommended for: <strong>{{ displayListAsString(courseSelected[getCustomFieldName('recommendedFor')]) }}</strong></p>
                  <q-btn color="primary" icon="add_shopping_cart" @click="handleGetThisCourseClick(courseSelected.id)">Get this course</q-btn>
                </div>
              </q-card-main>
            </q-card>
          </div>
        </div>
      </div>
    </div>
  </q-layout>
</template>

<script>
import {
  QLayout,
  QToolbar,
  QToolbarTitle,
  QSelect,
  QDataTable,
  QCard,
  QCardTitle,
  QCardMain,
  QBtn
} from 'quasar'

import axios from 'axios'
import _ from 'lodash'

/* Constants */

const TRAINING_OPS_SERVER_IP = '10.42.100.179'
const HTTP_REST_API = axios.create({
  baseURL: `http://${TRAINING_OPS_SERVER_IP}:5050/api/`,
  headers: {
    'Access-Control-Allow-Origin': '*' // for CORS to work properly
  }
})

const CUSTOM_FIELDS_MAP = {
  'requireVM': 'custom_field_1',
  'product': 'custom_field_3',
  'learningPaths': 'custom_field_4',
  'level': 'custom_field_5',
  'recommendedFor': 'custom_field_6'
}

/* Helpers */

function logAxiosError (error) {
  if (error.response) {
    // The request was made and the server responded with a status code that
    // falls out of the range of 2xx
    console.log(error.response.data)
    console.log(error.response.status)
    console.log(error.response.headers)
  }
  else if (error.request) {
    // The request was made but no response was received. `error.request` is an
    // instance of XMLHttpRequest in the browser and an instance of
    // http.ClientRequest in node.js.
    console.log(error.request)
  }
  else {
    // Something happened in setting up the request that triggered an error
    console.log('Error', error.message)
  }
  console.log(error.config)
}

function generateOptionsFromFlatList (flatList) {
  return _.map(flatList, item => ({ label: item, value: item }))
}

function extractUniqueNonNullValues (arrayOfObjects, fieldname) {
  let uniqueValues = _.map(_.uniqBy(arrayOfObjects, fieldname), e => e[fieldname])
  let uniqueNonNullValues = _.without(uniqueValues, null)
  return uniqueNonNullValues.sort()
}

function extractUniqueNonNullValuesFromNestedListsOfStrings (arrayOfObjects, fieldname) {
  // TODO Refactor with `extractUniqueNonNullValues` above

  let rawValues = _.map(arrayOfObjects, fieldname)

  // At this point, `rawValues` is a list of list of strings:
  // recommendations = [
  //   null,
  //   null,
  //   ['ALL', 'CSA', 'SUPPORT'],
  //   null,
  //   'SUPPORT',
  //   null
  // ]

  // Remove null values
  let nonNullValues = _.without(rawValues, null)
  // Flatten list of list of strings
  let flattenedValues = _.flatten(nonNullValues)
  // Extract unique values
  let uniqueValues = _.uniq(flattenedValues)
  // Return sorted values
  return uniqueValues.sort()
}

export default {
  name: 'catalog',
  components: {
    QLayout,
    QToolbar,
    QToolbarTitle,
    QSelect,
    QDataTable,
    QCard,
    QCardTitle,
    QCardMain,
    QBtn
  },
  data: () => ({
    loading: false,
    config: {
      rowHeight: '50px',
      pagination: {
        rowsPerPage: 10,
        options: [5, 10, 15, 30, 50, 100]
      }
    },
    columns: [
      {
        label: 'Name',
        field: 'name',
        width: '300px',
        filter: true,
        sort: true,
        type: 'string'
      },
      {
        label: 'Require VM?',
        field: CUSTOM_FIELDS_MAP['requireVM'],
        width: '100px',
        filter: true,
        sort: true,
        type: 'string'
      },
      {
        label: 'Product',
        field: CUSTOM_FIELDS_MAP['product'],
        width: '100px',
        filter: true,
        sort: true,
        type: 'string'
      },
      {
        label: 'Level',
        field: CUSTOM_FIELDS_MAP['level'],
        width: '100px',
        filter: true,
        sort: true,
        type: 'string'
      }
    ],
    allCoursesWithFullDetails: {}, // object with key = course ID, value = course details
    filters: {
      product: ''
    },
    courseSelected: {}
  }),
  computed: {
    tableDataRaw () {
      return _.values(this.allCoursesWithFullDetails)
    },
    tableDataFiltered () {
      // Apply filters on top of tableDataRaw (if any)
      let result = this.tableDataRaw
      if (this.filters.product) {
        result = _.filter(result, course => course[CUSTOM_FIELDS_MAP['product']] === this.filters.product)
      }
      if (this.filters.learningPath) {
        result = _.filter(result, course => _.includes(course[CUSTOM_FIELDS_MAP['learningPaths']], this.filters.learningPath))
      }
      if (this.filters.level) {
        result = _.filter(result, course => course[CUSTOM_FIELDS_MAP['level']] === this.filters.level)
      }
      if (this.filters.recommendedFor) {
        result = _.filter(result, course => _.includes(course[CUSTOM_FIELDS_MAP['recommendedFor']], this.filters.recommendedFor))
      }
      return result
    },
    products () {
      return generateOptionsFromFlatList(
        extractUniqueNonNullValues(
          this.tableDataRaw,
          CUSTOM_FIELDS_MAP['product']
        )
      )
    },
    learningPaths () {
      return generateOptionsFromFlatList(
        extractUniqueNonNullValuesFromNestedListsOfStrings(
          this.tableDataRaw,
          CUSTOM_FIELDS_MAP['learningPaths']
        )
      )
    },
    levels () {
      return generateOptionsFromFlatList(
        extractUniqueNonNullValues(
          this.tableDataRaw,
          CUSTOM_FIELDS_MAP['level']
        )
      )
    },
    recommendations () {
      return generateOptionsFromFlatList(
        extractUniqueNonNullValuesFromNestedListsOfStrings(
          this.tableDataRaw,
          CUSTOM_FIELDS_MAP['recommendedFor']
        )
      )
    }
  },
  methods: {
    handleRowClick (course) {
      this.courseSelected = course
    },

    handleGetThisCourseClick (courseId) {
      window.open(`http://employeelearning.talentlms.com/catalog/info/id:${courseId}`)
    },

    displayListAsString (listOfStrings, separator = '; ') {
      return listOfStrings ? _.join(listOfStrings, separator) : 'N/A'
    },

    getCustomFieldName (fieldName) {
      return CUSTOM_FIELDS_MAP[fieldName]
    }
  },
  created () {
    // Fetch and populate list of courses on startup
    this.loading = true
    HTTP_REST_API.get('/talentlms/courses?discard_inactive=true&discard_hidden=true')
      .then(response => {
        this.loading = false
        this.allCoursesWithFullDetails = response.data
      })
      .catch(e => {
        this.loading = false
        logAxiosError(e)
      })
  }
}
</script>

<style lang="stylus">

</style>
