<template>
  <div :class="['good-table', styleClass, rtl ? rtl : '']">
    <div :class="{'responsive': responsive}">
      <div v-if="title" class="table-header clearfix">
        <h2 class="table-title pull-left">{{title}}</h2>
        <div class="actions pull-right">
        </div>
      </div>

      <table ref="table" class="table">
        <thead>
          <tr v-if="globalSearch && externalSearchQuery == null">
            <td :colspan="lineNumbers ? columns.length + 1: columns.length">
              <div class="global-search">
                <input type="text" class="form-control global-search-input" :placeholder="globalSearchPlaceholder" v-model="globalSearchTerm" @keyup.enter="searchTable()" />
              </div>
            </td>
          </tr>
          <tr>
            <th v-if="lineNumbers" class="line-numbers"></th>
            <th v-for="(column, index) in columns"
              :key="column.field"
              @click="sort(index)"
              :class="getHeaderClasses(column, index)"
              :style="{width: column.width ? column.width : 'auto'}"
              v-if="!column.hidden">
              <slot name="table-column" :column="column">
                <span>{{column.label}}</span>
              </slot>
            </th>
            <slot name="thead-tr"></slot>
          </tr>
          <tr v-if="hasFilterRow">
            <th v-if="lineNumbers"></th>
            <th v-for="(column, index) in columns"
              :key="column.field"
              v-if="!column.hidden">
              <div v-if="column.filterable" 
                :class="getHeaderClasses(column, index)">
                <input v-if="!column.filterDropdown"
                  type="text"
                  class=""
                  :placeholder="getPlaceholder(column)"
                  :value="columnFilters[column.field]"
                  v-on:input="updateFilters(column, $event.target.value)" />

                <!-- options are a list of primitives -->
                <select v-if="column.filterDropdown && typeof(column.filterOptions[0]) !== 'object'"
                  class=""
                  :value="columnFilters[column.field]"
                  v-on:input="updateFilters(column, $event.target.value)">
                    <option value="">{{ getPlaceholder(column) }}</option>
                    <option
                      v-for="option in column.filterOptions"
                      :key="option"
                      :value="option">
                      {{ option }}
                    </option>
                </select>

                <!-- options are a list of objects with text and value -->
                <select v-if="column.filterDropdown && typeof(column.filterOptions[0]) === 'object'"
                  class=""
                  :value="columnFilters[column.field]"
                  v-on:input="updateFilters(column, $event.target.value)">
                  <option value="">{{ getPlaceholder(column) }}</option>
                  <option 
                    v-for="option in column.filterOptions"
                    :key="option"
                    :value="option.value">{{ option.text }}</option>
                </select>
              </div>
            </th>
          </tr>
        </thead>

        <tbody>
          <tr 
            v-for="(row, index) in paginated"
            :key="index"
            :class="getRowStyleClass(row)" 
            @click="click(row, index)">
            <th v-if="lineNumbers" class="line-numbers">{{ getCurrentIndex(index) }}</th>
            <slot name="table-row-before" :row="row" :index="index"></slot>
            <slot name="table-row" :row="row" :formattedRow="formattedRow(row)" :index="index">
              <td 
                v-for="(column, i) in columns"
                :key="column.field" 
                :class="getClasses(i, 'td')"
                v-if="!column.hidden && column.field">
                <span v-if="!column.html">{{ collectFormatted(row, column) }}</span>
                <span v-if="column.html" v-html="collect(row, column.field)"></span>
              </td>
            </slot>
            <slot name="table-row-after" :row="row" :index="index"></slot>
          </tr>
          <tr v-if="processedRows.length === 0">
            <td :colspan="columns.length">
              <slot name="emptystate">
                <div class="center-align text-disabled">
                  No data for table.
                </div>
              </slot>
            </td>
          </tr>
        </tbody>
      </table>

      <vue-good-pagination
        v-if="paginate"
        :perPage="perPage"
        :rtl="rtl"
        :total="processedRows.length"
        @page-changed="pageChanged"
        @per-page-changed="perPageChanged"></vue-good-pagination>
    </div>
  </div>
</template>

<script>
  import {format, parse, compareAsc, isValid} from 'date-fns/esm'
  import VueGoodPagination from './Pagination.vue'
  
  export default {
    name: 'vue-good-table',
    components: {
      VueGoodPagination
    },
    props: {
      styleClass: {default: 'table table-bordered'},
      title: '',
      columns: {},
      rows: {},
      onClick: {},
      perPage: {},
      sortable: {default: true},
      paginate: {default: false},
      lineNumbers: {default: false},
      defaultSortBy: {default: null},
      responsive: {default: true},
      rtl: {default: false},
      rowStyleClass: {default: null, type: [Function, String]},

      // search
      globalSearch: {default: false},
      searchTrigger: {default: null},
      externalSearchQuery: {default: null},

      // text options
      globalSearchPlaceholder: {default: 'Search ...'}
    },

    data: () => ({
      currentPage: 1,
      currentPerPage: 10,
      sortColumn: -1,
      sortType: 'asc',
      globalSearchTerm: '',
      columnFilters: {},
      filteredRows: [],
      timer: null,
      forceSearch: false,
      sortChanged: false,
    }),

    methods: {

      pageChanged(pagination) {
        this.currentPage = pagination.currentPage;
        this.$emit('pageChanged', {currentPage: this.currentPage, total: Math.floor(this.rows.length / this.currentPerPage)});
      },

      perPageChanged(pagination) {
        this.currentPerPage = pagination.currentPerPage;
      },

      sort(index) {
        if (!this.isSortableColumn(index))
          return;
        if (this.sortColumn === index) {
          this.sortType = this.sortType === 'asc' ? 'desc' : 'asc';
        } else {
          this.sortType = 'asc';
          this.sortColumn = index;
        }
        this.sortChanged = true;
      },

      click(row, index) {
        if (this.onClick)
          this.onClick(row, index);
      },

      searchTable() {
        if(this.searchTrigger == 'enter') {
          this.forceSearch = true;
          this.sortChanged = true;
        }
      },

      // field can be:
      // 1. function
      // 2. regular property - ex: 'prop'
      // 3. nested property path - ex: 'nested.prop'
      collect(obj, field) {

        //utility function to get nested property
        function dig(obj, selector) {
          var result = obj;
          const splitter = selector.split('.');
          for (let i = 0; i < splitter.length; i++)
            if (typeof(result) === 'undefined')
              return undefined;
            else
              result = result[splitter[i]];
          return result;
        }

        if (typeof(field) === 'function')
          return field(obj);
        else if (typeof(field) === 'string')
          return dig(obj, field);
        else
          return undefined;
      },

      collectFormatted(obj, column) {
        //helper functions within collect
        function formatDecimal(v) {
          return parseFloat(Math.round(v * 100) / 100).toFixed(2);
        }

        function formatPercent(v) {
          return parseFloat(v * 100).toFixed(2) + '%';
        }

        function formatDate(v) {
          // convert to date
          return format(parse(v, column.inputFormat, new Date()), column.outputFormat);
        }

        var value = this.collect(obj, column.field);

        if (value === undefined) return '';
        //lets format the resultant data
        switch(column.type) {
          case 'decimal':
            return formatDecimal(value);
          case 'percentage':
            return formatPercent(value);
          case 'date':
            return formatDate(value);
          default:
            return value;
        }
      },

      formattedRow(row) {
        var formattedRow = {};
        for (const col of this.columns) {
          if (col.field) {
            formattedRow[col.field] = this.collectFormatted(row, col);
          }
        }
        return formattedRow;
      },

      //Check if a column is sortable.
      isSortableColumn(index) {
        const sortable = this.columns[index].sortable;
        const isSortable = typeof sortable === 'boolean' ? sortable : this.sortable;
        return isSortable;
      },

      //Get classes for the given header column.
      getHeaderClasses(column, index) {
        const isSortable = this.isSortableColumn(index);
        const classes = Object.assign({}, this.getClasses(index, 'th'), {
          'sorting': isSortable,
          'sorting-desc': isSortable && this.sortColumn === index && this.sortType === 'desc',
          'sorting-asc': isSortable && this.sortColumn === index && this.sortType === 'asc',
        });
        return classes;
      },

      //Get classes for the given column index & element.
      getClasses(index, element) {
        const { type, [element + 'Class']: custom } = this.columns[index];
        let isRight = ['number', 'percentage', 'decimal', 'date'].includes(type);
        if (this.rtl) isRight = true;
        const classes = {
          'right-align': isRight,
          'left-align': !isRight,
          [custom]: !!custom
        };
        return classes;
      },

      //since vue doesn't detect property addition and deletion, we
      // need to create helper function to set property etc
      updateFilters(column, value) {
        const _this = this;
        if (this.timer) clearTimeout(this.timer);
        this.timer = setTimeout(function(){
          _this.$set(_this.columnFilters, column.field, value)
        }, 400);

      },

      //method to filter rows
      filterRows() {
        var computedRows = JSON.parse(JSON.stringify(this.rows));
        // we need to preserve the original index of rows so lets do that
        for(const [index, row] of computedRows.entries()) {
          row.originalIndex = index;
        }

        if(this.hasFilterRow) {
          for (var col of this.columns){
            if (col.filterable && this.columnFilters[col.field]) {
              computedRows = computedRows.filter(row => {

                // If column has a custom filter, use that.

                if (col.filter) {
                    return col.filter(this.collect(row, col.field), this.columnFilters[col.field])
                }

                // Use default filters

                switch(col.type) {
                  case 'number':
                  case 'percentage':
                  case 'decimal':
                    //in case of numeric value we need to do an exact
                    //match for now`
                    return this.collect(row, col.field) == this.columnFilters[col.field];
                  default:
                    //text value lets test starts with
                    return this.collect(row, col.field)
                      .toLowerCase()
                      .startsWith(
                        (this.columnFilters[col.field]).toLowerCase()
                      );
                }
              });
            }
          }
        }
        this.filteredRows = computedRows;
      },

      //get column's defined placeholder or default one
      getPlaceholder(column) {
        const placeholder = column.placeholder || 'Filter ' + column.label
        return placeholder
      },

      getCurrentIndex(index) {
        return (this.currentPage - 1) * this.currentPerPage + index + 1;
      },

      getRowStyleClass(row) {
        let classes = '';
        this.onClick ? classes += 'clickable' : '';
        let rowStyleClasses;
        if (typeof this.rowStyleClass === 'function') {
          rowStyleClasses = this.rowStyleClass(row);
        } else {
          rowStyleClasses = this.rowStyleClass;
        }
        if (rowStyleClasses) {
          classes += ' ' + rowStyleClasses;
        }
        return classes;
      }
    },

    watch: {
      columnFilters: {
          handler: function(newObj){
            this.filterRows();
          },
          deep: true
      },
      rows: {
        handler: function(newObj){
          this.filterRows();
        },
        deep: true
      }

    },

    computed: {

      searchTerm(){
        return (this.externalSearchQuery != null) ? this.externalSearchQuery : this.globalSearchTerm;
      },

      //
      globalSearchAllowed() {
        if (this.globalSearch
          && !!this.globalSearchTerm
          && this.searchTrigger != 'enter'){
          return true;
        }

        if (this.externalSearchQuery != null
           && this.searchTrigger != 'enter'){
          return true;
        }

        if (this.forceSearch){
          this.forceSearch = false;
          return true;
        }

        return false;
      },

      // to create a filter row, we need to
      // make sure that there is atleast 1 column
      // that requires filtering
      hasFilterRow(){
        if (!this.globalSearch) {
          for(var col of this.columns){
            if(col.filterable){
              return true;
            }
          }
        }
        return false;
      },

      // this is done everytime sortColumn
      // or sort type changes
      //----------------------------------------
      processedRows() {
       var computedRows = this.filteredRows;

        // take care of the global filter here also
        if (this.globalSearchAllowed) {
          var filteredRows = [];
          for (var row of this.rows) {
            for(var col of this.columns) {
              if (String(this.collectFormatted(row, col)).toLowerCase()
                  .search(this.searchTerm.toLowerCase()) > -1) {
                filteredRows.push(row);
                break;
              }
            }
          }
          computedRows = filteredRows;
        }

        //taking care of sort here only if sort has changed
        if (this.sortColumn !== -1 && this.isSortableColumn(this.sortColumn) &&

          // if search trigger is enter then we only sort
          // when enter is hit
          (this.searchTrigger !== 'enter' || this.sortChanged)) {

          this.sortChanged = false;

          computedRows = computedRows.sort((x,y) => {
            if (!this.columns[this.sortColumn])
              return 0;

            const cook = (d) => {
              d = this.collect(d, this.columns[this.sortColumn].field);

              //take care of dates too.
              if (this.columns[this.sortColumn].type === 'date') {
                d = parse(d + '', this.columns[this.sortColumn].inputFormat, new Date());
              } else if (typeof(d) === 'string') {
                d = d.toLowerCase();
                if (this.columns[this.sortColumn].type === 'number')
                  d = d.indexOf('.') >= 0 ? parseFloat(d) : parseInt(d);
              }
              return d;
            }

            x = cook(x);
            y = cook(y);

            // date comparison here
            if (this.columns[this.sortColumn].type === 'date') {
              if (!isValid(x)) {
                return -1 * (this.sortType === 'desc' ? -1 : 1);
              }
              if (!isValid(y)) {
                return (this.sortType === 'desc' ? -1 : 1);
              }
              return (compareAsc(x, y)) * (this.sortType === 'desc' ? -1 : 1);
            }

            // regular comparison here
            return (x < y ? -1 : (x > y ? 1 : 0)) * (this.sortType === 'desc' ? -1 : 1);
          })
        }

        // if the filtering is event based, we need to maintain filter
        // rows
        if (this.searchTrigger === 'enter') {
          this.filteredRows = computedRows;
        }

        return computedRows;
      },

      paginated() {
        var paginatedRows = this.processedRows;

        if (this.paginate) {
          var pageStart = (this.currentPage - 1) * this.currentPerPage;

          // in case of filtering we might be on a page that is
          // not relevant anymore
          // also, if setting to all, current page will not be valid
          if (pageStart >= this.processedRows.length
            || this.currentPerPage === -1) {
            this.currentPage = 1;
            pageStart = 0;
          }

          //calculate page end now
          var pageEnd = paginatedRows.length + 1;

          //if the setting is set to 'all'
          if (this.currentPerPage !== -1) {
            pageEnd = this.currentPage * this.currentPerPage;
          }

          paginatedRows = paginatedRows.slice(pageStart, pageEnd);
        }
        return paginatedRows;
      }
    },

    mounted() {
      this.filteredRows = JSON.parse(JSON.stringify(this.rows));

      // we need to preserve the original index of rows so lets do that
      for(const [index, row] of this.filteredRows.entries()) {
        row.originalIndex = index;
      }

      if (this.perPage) {
        this.currentPerPage = this.perPage;
      }

      //take care of default sort on mount
      if (this.defaultSortBy) {
        for (let [index, col] of this.columns.entries()) {
          if (col.field === this.defaultSortBy.field) {
            this.sortColumn = index;
            this.sortType = this.defaultSortBy.type || 'asc';
            this.sortChanged = true;
            break;
          }
        }
      }
    }
  }
</script>

<style lang="css" scoped>

/* Utility styles
************************************************/
.right-align{
  text-align: right;
}

.left-align{
  text-align: left;
}

.center-align{
  text-align: center;
}

.pull-left{
  float: left !important;
}

.pull-right{
  float: right !important;
}

.clearfix::after {
  display: block;
  content: "";
  clear: both;
}

/* Table specific styles
************************************************/

  table{
    border-collapse: collapse;
    background-color: transparent;
    margin-bottom: 0px;
  }
  .table{
    width: 100%;
    max-width: 100%;
    table-layout: auto;
  }

  .table-striped .table tbody tr:nth-of-type(odd) {
      background-color: rgba(35,41,53,.02);
  }

  .table-striped .table tbody tr:hover {
      background-color: rgba(35,41,53,.075);
  }

  .table-bordered .table td, .table-bordered th,
  .table-bordered .table-footer {
      border: 1px solid #ddd;
  }

  .table td, .table th:not(.line-numbers) {
    padding: .75rem 1.2rem .75rem .75rem;
    vertical-align: top;
    border-top: 1px solid #ddd;
  }

  .rtl .table td, .rtl .table th:not(.line-numbers) {
    padding: .75rem;
  }

  .table.condensed td, .table.condensed th {
    padding: .4rem .4rem .4rem .4rem;
  }

  .table thead th, .table.condensed thead th {
    cursor: pointer;
    vertical-align: bottom;
    padding-right: 1.5rem;
    background-color: rgba(35,41,53,0.05);
  }
  .rtl .table thead th, .rtl .table.condensed thead th {
    padding-left: 1rem;
    padding-right: .75rem;
  }

  tr.clickable {
    cursor: pointer;
  }

  .table input, .table select{
    box-sizing: border-box;
    display: block;
    width: 100%;
    height: 28px;
    padding: 5px 10px;
    font-size: 14px;
    line-height: 1.42857143;
    color: #555;
    background-color: #fff;
    background-image: none;
    border: 1px solid #ccc;
    box-shadow: inset 0 1px 1px rgba(35,41,53,.075);
  }

  table th.sorting-asc,
  table th.sorting-desc {
    color: rgba(0,0,0,.7);
    position: relative;
  }

  table th.sorting:after,
  table th.sorting-asc:after  {
    font-family: 'Material Icons';
    position: absolute;
    height: 0px;
    width: 0px;
    content: '';
    display: none;
    border: 4px solid transparent;
    border-bottom: 6px solid rgba(0,0,0,.6);
    margin-top: 2px;
    margin-left: 4px;
  }

  .rtl table th.sorting:after,
  .rtl table th.sorting-asc:after{
    margin-right: 4px;
    margin-left: 0;
  }

  table th.sorting-asc:after,
  table th.sorting-desc:after,
  table th.sorting:hover:after {
    display: inline-block;
  }

  table th.sorting-desc:after {
    border-bottom-color: transparent;
    border: 4px solid transparent;
    border-top: 6px solid rgba(0,0,0,.6);
    margin-top: 6px;
  }

.responsive {
  width: 100%;
}

/* Table header specific styles
************************************************/

.table-header{
  padding: .75rem;
}

.table-header .table-title{
  margin: 0px;
  font-size: 18px;
}

  /* Global Search
  **********************************************/
  .global-search{
    position: relative;
    margin-left: .45rem;
  }
  table .global-search-input{
    width: 100%;
  }

  /* Line numbers
  **********************************************/
  table th.line-numbers, .table.condensed th.line-numbers{
    background-color: rgba(35,41,53,0.05);
    padding-left: 3px;
    padding-right: 3px;
    word-wrap: break-word;
    width: 45px;
    text-align: center;
  }

  .good-table.rtl{
    direction: rtl;
  }

  .text-disabled{
    color: #aaa;
  }

</style>
