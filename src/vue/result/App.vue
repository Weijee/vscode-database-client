<template>
  <div id="app">
    <div class="hint">
      <div style="width:95%;">
        <el-input type="textarea" :autosize="{ minRows:2, maxRows:5}" v-model="toolbar.sql" class="sql-pannel" />
      </div>
      <div class="toolbar">
        <el-button v-if="showFullBtn" @click="()=>sendToVscode('full')" type="primary" title="Full Result View" icon="el-icon-rank" size="mini" circle>
        </el-button>
        <el-input v-model="table.search" size="mini" placeholder="Input To Search Data" style="width:200px" :clearable="true" />
        <el-button type="primary" size="mini" icon="el-icon-milk-tea" title="Buy the author a cup of coffee" circle @click="()=>sendToVscode('openCoffee')"></el-button>
        <el-button @click="$refs.editor.openInsert()" :disabled="result.tableCount!=1" type="info" title="Insert new row" icon="el-icon-circle-plus-outline" size="mini" circle>
        </el-button>
        <el-button @click="deleteConfirm" title="delete" type="danger" size="mini" icon="el-icon-delete" circle :disabled="!toolbar.show">
        </el-button>
        <el-button @click="exportOption.visible = true" type="primary" size="mini" icon="el-icon-bottom" circle title="Export"></el-button>
        <el-button type="success" size="mini" icon="el-icon-caret-right" :disabled="!toolbar.sql" title="Execute Sql" circle @click='info.visible = false;execute(toolbar.sql);'></el-button>
        <div style="display:inline-block;font-size:14px;padding-left: 8px;" class="el-pagination__total">
          CostTime: {{result.costTime}}ms
        </div>
        <div style="display:inline-block">
          <el-pagination @size-change="changePageSize" @current-change="page=>changePage(page,true)" @next-click="()=>changePage(1)" @prev-click="()=>changePage(-1)" :current-page.sync="page.pageNum" :small="true" :page-size="page.pageSize" :page-sizes="[100,30,50,300,500]" :layout="page.total!=null?'sizes,prev,pager, next, total':'sizes,prev, next'" :total="page.total">
          </el-pagination>
        </div>
      </div>
      <div v-if="info.visible ">
        <div v-if="info.error" class="info-panel" style="color:red !important" v-html="info.message"></div>
        <div v-if="!info.error" class="info-panel" style="color: green !important;" v-html="info.message"></div>
      </div>
    </div>
    <!-- trigger when click -->
    <ux-grid ref="dataTable" v-loading='table.loading' size='small' :cell-style="{height: '35px'}" @sort-change="sort" :height="remainHeight" width="100vh" stripe @selection-change="selectionChange" :checkboxConfig="{ checkMethod: ({row})=>editable&&!row.isFilter,highlight: true}" :data="result.data.filter(data => !table.search || JSON.stringify(data).toLowerCase().includes(table.search.toLowerCase()))" :show-header-overflow="false" :show-overflow="false">
      <ux-table-column type="checkbox" width="40" fixed="left"> </ux-table-column>
      <ux-table-column type="index" width="40" :seq-method="({row,rowIndex})=>(rowIndex||!row.isFilter)?rowIndex:undefined">
        <Controller slot="header" :result="result" :toolbar="toolbar" />
      </ux-table-column>
      <ux-table-column v-for="(field,index) in (result.fields||[]).filter(field=>toolbar.showColumns.includes(field.name.toLowerCase()))" :key="index" :resizable="true" :field="field.name" :title="field.name" :sortable="true" :width="computeWidth(field,0)" edit-render>
        <Header slot="header" slot-scope="scope" :result="result" :scope="scope" :index="index" />
        <Row slot-scope="scope" :scope="scope" :result="result" :filterObj="toolbar.filter" :editList.sync="update.editList" @execute="execute" @sendToVscode="sendToVscode" @openEditor="openEditor" />
      </ux-table-column>
    </ux-grid>
    <EditDialog ref="editor" :dbType="result.dbType" :database="result.database" :table="result.table" :primaryKey="result.primaryKey" :primaryKeyList="result.primaryKeyList" :columnList="result.columnList" @execute="execute" />
    <ExportDialog :visible.sync="exportOption.visible" @exportHandle="confirmExport" />
  </div>
</template>

<script>
import { getVscodeEvent } from "../util/vscode";
import Row from "./component/Row";
import Controller from "./component/Row/Controller.vue";
import Header from "./component/Row/Header.vue";
import ExportDialog from "./component/ExportDialog.vue";
import EditDialog from "./component/EditDialog";
import { util } from "./mixin/util";
let vscodeEvent;

export default {
  mixins: [util],
  components: {
    ExportDialog,
    EditDialog,
    Controller,
    Row,
    Header,
  },
  data() {
    return {
      showFullBtn: false,
      remainHeight: 0,
      connection: {},
      result: {
        data: [],
        dbType: "",
        costTime: 0,
        sql: "",
        primaryKey: null,
        columnList: null,
        primaryKeyList: null,
        database: null,
        table: null,
        tableCount: null,
        pageSize: null,
      },
      page: {
        pageNum: 1,
        pageSize: -1,
        total: null,
      },
      table: {
        search: "",
        loading: true,
        widthItem: {},
      },
      toolbar: {
        sql: null,
        show: false,
        // using to clear filter input value
        filter: {},
        showColumns: [],
      },
      exportOption: {
        visible: false,
      },
      info: {
        sql: null,
        message: null,
        visible: false,
        error: false,
        needRefresh: true,
      },
      update: {
        editList: {},
        lock: false,
      },
    };
  },
  mounted() {
    this.remainHeight = window.innerHeight - 120;
    this.showFullBtn = window.outerWidth / window.innerWidth >= 2;
    window.addEventListener("resize", () => {
      this.remainHeight = window.innerHeight - 120;
      this.showFullBtn = window.outerWidth / window.innerWidth >= 2;
    });
    const handlerData = (data, sameTable) => {
      this.result = data;
      this.toolbar.sql = data.sql;

      if (sameTable) {
        this.clear();
      } else {
        this.reset();
      }
      // only es have.
      if (data.total != null) {
        this.page.total = parseInt(data.total);
      } else if (
        this.result.tableCount == 1 &&
        this.page.pageSize < this.result.data.length + 1
      ) {
        this.count();
      } else {
        this.page.total = this.result.data.length - 1;
      }
      this.update.editList = [];
      this.update.lock = false;
    };
    const handlerCommon = (res) => {
      if (this.$refs.editor) {
        this.$refs.editor.close();
      }
      this.info.visible = true;
      this.info.message = res.message;
    };
    vscodeEvent = getVscodeEvent();

    vscodeEvent.on("updateSuccess", () => {
      for (const index in this.update.editList) {
        const element = this.update.editList[index];
        this.result.data[index] = element;
      }
      this.update.editList = [];
      this.update.lock = false;
      this.$message({
        showClose: true,
        duration: 500,
        message: "Update Success",
        type: "success",
      });
    });
    window.onkeypress = (e) => {
      if (
        (e.code == "Enter" && e.target.classList.contains("edit-column")) ||
        (e.ctrlKey && e.code == "KeyS")
      ) {
        this.save();
        e.stopPropagation();
        e.preventDefault();
      }
    };
    window.addEventListener("message", ({ data }) => {
      if (!data) return;
      const response = data.content;
      console.log(data);
      this.table.loading = false;
      switch (data.type) {
        case "EXPORT_DONE":
          this.exportOption.visible = false;
          break;
        case "RUN":
          this.toolbar.sql = response.sql;
          this.table.loading = response.transId != this.result.transId;
          break;
        case "DATA":
          handlerData(response);
          break;
        case "NEXT_PAGE":
          this.result.data = response.data;
          this.toolbar.sql = response.sql;
          this.result.data.unshift({ isFilter: true, content: "" });
          break;
        case "COUNT":
          this.page.total = parseInt(response.data);
          break;
        case "DML":
        case "DDL":
        case "MESSAGE_BLOCK":
          handlerCommon(response);
          this.info.error = false;
          this.info.needRefresh = false;
          if (
            response.message.indexOf("AffectedRows") != -1 ||
            response.isInsert
          ) {
            this.refresh();
          }
          break;
        case "ERROR":
          handlerCommon(response);
          this.info.error = true;
          break;
        case "MESSAGE":
          if (response.message) {
            this.$message({
              showClose: true,
              duration: 1000,
              message: response.message,
              type: response.success ? "success" : "error",
            });
          }
          this.refresh();
          break;
      }
    });
    vscodeEvent.emit("init");
    window.addEventListener("keyup", (event) => {
      if (event.key == "c" && event.ctrlKey) {
        document.execCommand("copy");
      }
    });
  },
  methods: {
    save() {
      if (Object.keys(this.update.editList).length == 0 && this.update.lock) {
        return;
      }
      this.update.lock = true;
      let sql = "";
      for (const index in this.update.editList) {
        const element = this.update.editList[index];
        sql += this.$refs.editor.buildUpdateSql(
          element,
          this.result.data[index]
        );
      }
      if (sql) {
        vscodeEvent.emit("saveModify", sql);
      }
    },
    sendToVscode(event, param) {
      vscodeEvent.emit(event, param);
    },
    openEditor(row, isCopy) {
      if (isCopy) {
        this.$refs.editor.openCopy(row);
      } else {
        this.$refs.editor.openEdit(row);
      }
    },
    confirmExport(exportOption) {
      vscodeEvent.emit("export", {
        option: {
          ...exportOption,
          sql: this.result.sql,
          table: this.result.table,
        },
      });
    },
    changePageSize(size) {
      this.page.pageSize = size;
      vscodeEvent.emit("changePageSize", size);
      this.changePage(0);
    },
    sort(row) {
      if (this.result.dbType == "ElasticSearch") {
        vscodeEvent.emit("esSort", [{ [row.prop]: { order: row.order } }]);
        return;
      }
      let sortSql = this.result.sql
        .replace(/\n/, " ")
        .replace(";", "")
        .replace(/order by .+? (desc|asc)?/gi, "")
        .replace(/\s?(limit.+)?$/i, ` ORDER BY ${row.prop} ${row.order} \$1 `);
      this.execute(sortSql + ";");
    },
    getTypeByColumn(key) {
      if (!this.result.columnList) return;
      for (const column of this.result.columnList) {
        if (column.name === key) {
          return column.simpleType || column.type;
        }
      }
    },
    deleteConfirm() {
      this.$confirm("Are you sure you want to delete this data?", "Warning", {
        confirmButtonText: "OK",
        cancelButtonText: "Cancel",
        type: "warning",
      })
        .then(() => {
          let checkboxRecords = this.$refs.dataTable
            .getCheckboxRecords()
            .filter(
              (checkboxRecord) => checkboxRecord[this.result.primaryKey] != null
            )
            .map((checkboxRecord) =>
              this.wrapQuote(
                this.getTypeByColumn(this.result.primaryKey),
                checkboxRecord[this.result.primaryKey]
              )
            );
          let deleteSql = null;
          if (this.result.dbType == "ElasticSearch") {
            deleteSql =
              checkboxRecords.length > 1
                ? `POST /_bulk\n${checkboxRecords
                    .map(
                      (c) =>
                        `{ "delete" : { "_index" : "${this.result.table}", "_id" : "${c}" } }`
                    )
                    .join("\n")}`
                : `DELETE /${this.result.table}/_doc/${checkboxRecords[0]}`;
          } else if (this.result.dbType == "MongoDB") {
            deleteSql = `db('${this.result.database}').collection("${
              this.result.table
            }")
              .deleteMany({_id:{$in:[${checkboxRecords.join(",")}]}})`;
          } else {
            deleteSql =
              checkboxRecords.length > 1
                ? `DELETE FROM ${this.result.table} WHERE ${
                    this.result.primaryKey
                  } in (${checkboxRecords.join(",")})`
                : `DELETE FROM ${this.result.table} WHERE ${this.result.primaryKey}=${checkboxRecords[0]}`;
          }
          this.execute(deleteSql);
        })
        .catch((e) => {
          if (e) {
            this.$message.error(e);
          } else {
            this.$message({ type: "warning", message: "Delete canceled" });
          }
        });
    },
    /**
     * compute column row width, get maxium of fieldName or value or fieldType by top 10 row.
     */
    computeWidth(field, index) {
      // only compute once.
      let key = field.name;
      if (this.table.widthItem[key]) return this.table.widthItem[key];
      if (!index) index = 0;
      if (!this.result.data[index] || index > 10) return 70;
      const value = this.result.data[index][key];
      var dynamic = Math.max(
        (key + "").length * 10,
        (value + "").length * 10,
        (field.type + "").length * 10
      );
      if (dynamic > 150) dynamic = 150;
      if (dynamic < 70) dynamic = 70;
      var nextDynamic = this.computeWidth(field, index + 1);
      if (dynamic < nextDynamic) dynamic = nextDynamic;
      // cache column width
      this.table.widthItem[key] = dynamic;
      return dynamic;
    },
    refresh() {
      if (this.result.sql) {
        this.execute(this.result.sql);
      }
    },
    count() {
      if (!this.result.table) return;
      this.info.visible = false;
      vscodeEvent.emit("count", { sql: this.result.sql });
    },
    execute(sql) {
      if (!sql) return;
      vscodeEvent.emit("execute", {
        sql,
      });
      this.table.loading = true;
    },
    changePage(pageNum, jump) {
      vscodeEvent.emit("next", {
        sql: this.result.sql,
        pageNum: jump ? pageNum : this.page.pageNum + pageNum,
        pageSize: this.page.pageSize,
      });
      this.table.loading = true;
    },

    initShowColumn() {
      const fields = this.result.fields;
      if (!fields) return;
      this.toolbar.showColumns = [];
      for (let i = 0; i < fields.length; i++) {
        if (!fields[i].name) continue;
        this.toolbar.showColumns.push(fields[i].name.toLowerCase());
      }
    },
    // show call when load same table data
    clear() {
      // reset page
      this.page.pageNum = 1;
      this.page.pageSize = this.result.pageSize;
      this.page.total = null;
      // info
      if (this.info.needRefresh) {
        this.info.visible = false;
      } else {
        this.info.needRefresh = true;
      }
      // loading
      this.table.loading = false;
      // toolbar
      this.toolbar.show = false;
    },
    selectionChange(selection) {
      this.toolbar.show =
        this.result.primaryKey &&
        selection.length > 0 &&
        this.result.tableCount == 1;
    },
    // show call when change table
    reset() {
      this.clear();
      // table
      this.table.widthItem = {};
      this.initShowColumn();
      // add filter row
      if (this.result.columnList) {
        this.result.data.unshift({ isFilter: true, content: "" });
      }
      // toolbar
      if (!this.result.sql.match(/\bwhere\b/gi)) {
        this.toolbar.filter = {};
        this.$refs.dataTable.clearSort();
      }
    },
  },
  computed: {
    editable() {
      return this.result.primaryKey && this.result.tableCount == 1;
    },
    columnCount() {
      if (this.result.data == undefined || this.result.data[0] == undefined)
        return 0;
      return Object.keys(this.result.data[0]).length;
    },
  },
};
</script>