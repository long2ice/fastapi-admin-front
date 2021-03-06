<template>
  <div class="custom-table">
    <div class="row">
      <div class="col col-md-12">
        <b-btn
          class="mr-1" size="sm"
          :to="uri + '/create'"
          variant="secondary"
          v-if="_.get(actions,'toolbar.create') !== false"
        >
          <i class="icon-plus"></i>
          {{ $t('actions.create') }}
        </b-btn>
        <b-btn
          class="mr-1" size="sm"
          @click="fetch"
          variant="success"
          v-if="_.get(actions, 'toolbar.reload') !== false"
        >
          <i class="icon-reload"></i>
          {{ $t('actions.reload') }}
        </b-btn>
        <b-btn
          class="mr-1" size="sm"
          v-for="button in _.get(actions, 'toolbar.extra', [])"
          :key="button.label"
          v-bind="button"
        >{{ button.label }}
        </b-btn>
        <b-dropdown no-caret size="sm" class="mr-1" variant="primary">
          <template v-slot:button-content>
            <i class="icon-settings">
              {{ $t('actions.columns') }}
            </i>
          </template>
          <b-dd-form>
            <b-form-checkbox-group stacked v-model="columns_select" :options="column_options"
                                   @change="columns_select_change">
            </b-form-checkbox-group>
          </b-dd-form>
        </b-dropdown>
      </div>
    </div>
    <div class>
      <div class="my-2">
        <b-form-builder
          :onSubmit="doSearch"
          back-text
          inline
          v-if="_.keys(table.searchFields).length > 0"
          :submit-text="$t('actions.search')"
          :fields="searchFields"
          v-model="table.searchModel"
        >

          <div slot="extra-buttons" class="ml-1">
            <b-button
              type="button"
              @click="searchAndExport"
              variant="success"
              v-if="this.export"
            >{{ $t('actions.search_and_export') }}
            </b-button>
          </div>
          <div slot="extra-buttons" class="ml-1">
            <b-form-file
              accept=".csv, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel"
              @input="select_file" v-show="false" ref="file">
            </b-form-file>
            <b-button
              type="button"
              variant="secondary"
              v-if="this.import"
              @click="$refs.file.$el.childNodes[0].click()"
            >{{ $t('actions.import') }}
            </b-button>
          </div>
        </b-form-builder>
      </div>
      <div class="row align-items-center">
        <div class="col-md-8">
          <b-pagination
            :limit="pageLimit"
            v-model="currentPage"
            :total-rows="total"
            :per-page="perPage"
          ></b-pagination>
        </div>
        <div class="col-md-4 form-inline justify-content-end">
          <b-select v-model="perPage" :options="pages" class="mx-2">
          </b-select>
          <span>{{ $t('messages.paginate', {total: total}) }}</span>
        </div>
      </div>
      <b-table
        class="data-table"
        v-if="table.fields"
        ref="table"
        :items="fetchItems"
        :fields="columns"
        selectable
        :per-page="perPage"
        @row-selected="onRowSelected"
        :select-mode="selectModel"
        :current-page="currentPage"
        :sort-by.sync="sortBy"
        :sort-desc.sync="sortDesc"
        :sort-direction="sortDirection"
      >
        <template v-slot:cell(selected)="{ rowSelected }">
          <template v-if="rowSelected">
            <span aria-hidden="true">&check;</span>
            <span class="sr-only">Selected</span>
          </template>
          <template v-else>
            <span aria-hidden="true">&nbsp;</span>
            <span class="sr-only">Not selected</span>
          </template>
        </template>
        <template v-slot:cell()="data">
          <template v-if="['datetime'].includes(data.field.type)">
            {{ $d(new Date(data.value), 'long') }}
          </template>
          <template v-else-if="['date'].includes(data.field.type)">
            {{ $d(new Date(data.value), 'short') }}
          </template>
          <template v-else-if="['image'].includes(data.field.type)">
            <b-img width="50px" class="type-image" :src="data.value" fluid/>
          </template>
          <template v-else-if="['link'].includes(data.field.type)">
            <a :class="data.field.classes" :href="data.value" :target="data.field.target">
              <i :class="data.field.icon" v-if="data.field.icon"></i>
              {{ data.value }}
            </a>
          </template>
          <template v-else-if="['switch', 'boolean', 'checkbox'].includes(data.field.type)">
            <b-badge :variant="data.value ? 'success' : 'danger'">
              {{ data.value ? 'Yes' : 'No' }}
            </b-badge>
          </template>
          <template v-else-if="['html'].includes(data.field.type)">
            <div v-html="data.value" class=" data-value-html"></div>
          </template>
          <template v-else-if="['select', 'select2', 'radiolist', 'checkboxlist'].includes(data.field.type)">
            {{ _.find(data.field.options, {value: data.value}).text }}
          </template>
          <template v-else>
            <span class="d-inline-block text-truncate" style="max-width: 200px;">
            {{ data.value }}
            </span>
          </template>
        </template>
        <template v-slot:cell(_actions)="row">
          <b-btn
            v-if="actions.view !== false"
            variant="success"
            size="sm"
            :to="`${uri}/${row.item[pk]}`"
            class="mr-1"
          >{{ $t('actions.view') }}
          </b-btn>
          <b-btn
            v-if="actions.edit !== false"
            variant="primary"
            size="sm"
            :to="`${uri}/${row.item[pk]}/edit`"
            class="mr-1"
          >{{ $t('actions.edit') }}
          </b-btn>
          <b-btn
            v-if="actions.delete !== false"
            size="sm"
            @click.stop="remove(row.item[pk])"
          >{{ $t('actions.delete') }}
          </b-btn>
        </template>
      </b-table>
      <div class="form-inline my-2">
        <b-button class="mr-1" size="sm" @click="selectAllRows">{{ $t("actions.select_all") }}</b-button>
        <b-button class="mr-1" size="sm" @click="clearSelected">{{ $t("actions.clear_selected") }}</b-button>
        <b-form-select class="mr-1" size="sm" v-model="selectBulkAction" :options="bulkActions"></b-form-select>
        <b-button @click="submitBulk" size="sm" variant="primary">{{ $t("actions.submit") }}</b-button>
      </div>
      <div class="row align-items-center">
        <div class="col-md-10">
          <b-pagination
            :limit="pageLimit"
            v-model="currentPage"
            :total-rows="total"
            :per-page="perPage"
          ></b-pagination>
        </div>
        <div class="col-md-2 text-right">{{ $t('messages.paginate', {total: total}) }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import {mapGetters, mapState} from "vuex";
import _ from "lodash";
import {saveAs} from 'file-saver';
import XLSX from "xlsx";

export default {
  components: {},
  props: {},
  data() {
    return {
      init: false,
      loaded: false,
      table: {},
      modes: ['multi', 'single', 'range'],
      selectModel: 'multi',
      total: 0,
      pageLimit: 10,
      currentPage: 1,
      sortBy: this.pk,
      sortDesc: true,
      sortDirection: null,
      perPage: 10,
      where: {},
      selected_pk_list: [],
      bulkActions: {},
      selectBulkAction: null,
      pages: [10, 20, 50, 100],
      columns_select: [],
      column_options: [],
      searchFields: {}
    };
  },
  watch: {
    "$route.query"(val) {
      this.applyRouteQuery();
    },
    "$route.params"(val) {
      this.applyRouteQuery();
      this.fetch();
    }
  },
  computed: {
    ...mapState(["site", "i18n", "auth"]),
    ...mapGetters(["currentLanguage"]),
    columns() {
      let entries = Object.entries(this.table.fields)
      let filter_columns = entries.filter(items => {
        return this.columns_select.indexOf(items[0]) > -1 || items[0] === "_actions";
      })
      return filter_columns.map(([name, field]) => {
        return {
          key: name,
          ...field,
        }
      });
    },
    pk() {
      return _.get(this.table, "pk");
    },
    export() {
      return _.get(this.table, "export");
    },
    import() {
      return _.get(this.table, "import_");
    },
    populate() {
      return _(this.table.fields || {})
        .map("ref")
        .filter()
        .map(v => v.split(".").shift())
        .uniq()
        .toJSON();
    },
    actions() {
      return _.get(this.table, "fields._actions", {});
    },
    resource() {
      return this.$route.params.resource;
    },
    uri() {
      return this.site.resource_prefix + '/' + this.resource.replace(/\./g, "/");
    }
  },
  methods: {
    columns_select_change(values) {
      localStorage.setItem(this.resource, JSON.stringify(values))
    },
    get_select_change() {
      let columns_select = localStorage.getItem(this.resource)
      if (!columns_select) {
        return []
      } else {
        return JSON.parse(columns_select)
      }
    },
    submitBulk() {
      if (window.confirm(this.$t("messages.confirm_bulk_action"))) {
        this.$http.post(this.uri + '/bulk/' + this.selectBulkAction, {
          pk_list: this.selected_pk_list
        }).then(() => {
          this.$snotify.success(this.$t("messages.bulk_success"));
          this.fetch();
        });
      }
    },
    selectAllRows() {
      this.$refs.table.selectAllRows()
    },
    onRowSelected(items) {
      this.selected_pk_list = _.map(items, item => {
        return item[this.pk];
      })
    },
    clearSelected() {
      this.$refs.table.clearSelected()
    },
    doSearch(params) {
      this.where = _.omitBy(params, v => v === null);
      this.$refs.table.refresh();
    },
    searchAndExport() {
      const query = JSON.stringify({
        where: _.clone(this.table.searchModel),
        with: _.clone(this.populate)
      });
      this.$http.get(this.uri + "/export", {
        responseType: 'arraybuffer',
        params: {
          query: query
        }
      }).then(res => {
        const blob = new Blob([res.data]);
        saveAs(blob, `${this.resource}.xlsx`)
      })
    },
    select_file(file) {
      let reader = new FileReader();
      let that = this;
      reader.onload = function (e) {
        let data = new Uint8Array(e.target.result);
        let workbook = XLSX.read(data, {type: 'array', cellDates: true});
        let sheet_name = workbook.SheetNames[0];
        let sheet = workbook.Sheets[sheet_name];
        const opts = {
          /** Default value for null/undefined values */
          defval: ''//给defval赋值为空的字符串
        }
        let json_data = XLSX.utils.sheet_to_json(sheet,opts);
        that.$http.post(that.uri + '/import', json_data).then(res => {
          that.$refs.table.refresh();
          that.$snotify.success(this.$t("messages.import", res.data));
        })
      };
      reader.readAsArrayBuffer(file);
    },
    applyRouteQuery() {
      const {sort = {}, page = 1, where = {}} = JSON.parse(
        this.$route.query.query || "{}"
      );
      const [sortBy, sortDesc] = Object.entries(sort).pop() || [];
      sortBy && (this.sortBy = sortBy);

      if (sortDesc) {
        this.sortDesc = sortDesc === -1;
      }
      this.total = page * this.perPage;
      this.currentPage = page;
      this.where = where;
      this.init = true;
    },
    remove(id) {
      if (window.confirm("是否删除?")) {
        this.$http.delete(`${this.uri}/${id}`).then(res => {
          this.$snotify.success(this.$t("messages.deleted"));
          this.$refs.table.refresh();
        });
      }
    },
    fetchItems(ctx) {
      const query = _.merge({}, _.get(this.table, "query"), {
        page: ctx.currentPage,
        size: ctx.perPage,
        sort: {[ctx.sortBy]: this.sortDesc ? -1 : 1},
        where: this.where,
        with: this.populate
      });
      // console.log(query)

      if (!this.init) {
        // this.$router.replace({
        //   query: { query: JSON.stringify(query) }
        // });
        return [];
      }
      // this.$router.push({
      //   query: { query: JSON.stringify(query) }
      // });
      return this.$http
        .get(this.uri, {
          params: {
            query: JSON.stringify(query)
          }
        })
        .then(res => {
          const {total, data} = res.data;
          this.total = total;
          return data;
        })
        .catch(e => {
          return [];
        });
    },
    fetch() {
      this.init = false;
      this.columns_select = this.get_select_change();
      let columns_selected = this.columns_select.length !== 0;
      this.column_options = [];
      this.$http.get(this.uri + "/grid").then(res => {
        _.mapValues(res.data.bulk_actions, action => {
          action.text = this.$t(`actions.${action.text}`)
        });

        this.bulkActions = res.data.bulk_actions;

        _.mapValues(res.data.fields, field => {
          field.thClass = "bg-light";
        });
        _.forEach(res.data.fields, (value, key) => {
          this.column_options.push({
            text: value.label, value: key
          });
          if (!columns_selected) {
            this.columns_select.push(key);
          }
        })
        this.table = res.data;
        this.searchFields = this.table.searchFields;

        if (_.get(this.table, "fields._actions") !== false) {
          _.set(
            this.table,
            "fields._actions.label",
            this.$t("actions.actions")
          );
          _.set(
            this.table,
            "fields._actions.thClass",
            'bg-light'
          );
        }
        this.init = true;
        if (this.$refs.table) {
          this.$refs.table.refresh();
        }
      }).catch(error => {
        this.table = {}
      });
    }
  },
  mounted() {
  },
  created() {
    this.applyRouteQuery();

    this.fetch();
    // this.fetchTable();
  }
};
</script>
