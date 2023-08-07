<template>
  <div class="combobox">
    <input
      ref="input"
      :id="name"
      class="combobox-inp"
      :name="name"
      type="text"
      :comboxWidth="width"
      :comboxHeight="height"
      :url="url"
      :paramSize="pageSize"
      :paramName="paramName"
      :class="{ noborder: !border, 'combobox-inp-disabled': disabled }"
      @mousedown="ComboBox_ShowDownList"
      @keydown="ComboBox_OnKeyDown"
      @focus="ComboBox_OnFocus"
      :isFocus="isFocus"
      @blur="ComboBox_Hidden"
      autocomplete="off"
      :disabled="disabled"
      :value="modelValue"
    />
    <div
      ref="container"
      class="container"
      v-show="isOpen"
      :id="containerid"
      :style="{ width: width + 'px', height: height + 'px' }"
    >
      <div class="Header">
        <table>
          <colgroup>
            <col
              v-for="(item, index) in this.datas.columns"
              :style="{ width: item }"
              :key="'head' + index"
            />
          </colgroup>
          <tr class="tablehead">
            <td></td>
            <td v-for="item in this.datas.headers" :key="item.code">
              {{ item.name }}
            </td>
          </tr>
        </table>
      </div>
      <div class="Content" :style="{ height: height - 64 + 'px' }">
        <table ref="datatable">
          <col
            v-for="(item, index) in this.datas.columns"
            :style="{ width: item }"
            :key="'col' + index"
          />
          <tr
            class="TableRow"
            @mousedown="ComboBox_ItemSelected(data.olddata)"
            :data="JSON.stringify(data.olddata)"
            v-for="(data, index) in filterdata"
            :key="index"
          >
            <td>{{ index + 1 }}</td>
            <td v-for="(val, i) in data.data" :key="'td-' + i">
              <div>{{ val }}</div>
            </td>
          </tr>
        </table>
      </div>
      <div class="Footer">
        <div>
          共{{ this.datas.rowCount }}行|当前{{ start }}-{{ end }}行
          键盘“↓”可加载下一页
        </div>
      </div>
    </div>
  </div>
</template>
<style>
.combobox {
  display: inline-block;
  background-color: #fff !important;
  width: 100%;
}
.container {
  position: absolute;
  z-index: 999;
  background-color: #fff !important;
  box-shadow: rgba(50, 50, 93, 0.25) 0px 50px 100px -20px,
    rgba(0, 0, 0, 0.3) 0px 30px 60px -30px,
    rgba(10, 37, 64, 0.35) 0px -2px 6px 0px inset;
  box-sizing: content-box;
  padding: 10px;
}
.Content {
  position: absolute;
  top: 42px;
  overflow: hidden;
}
table {
  border-collapse: collapse;
  border-spacing: 0px;
  table-layout: fixed;
  word-break: break-all;
}
table,
td {
  border: 1px solid #dcdee2;
}
.Header .tablehead td,
.Content .TableRow td,
.Content .TableRowOver td {
  height: 32px;
  overflow: hidden;
}
.tablehead {
  background-color: #f8f8f9;
}
.tablehead td {
  text-align: center;
  background-color: #fff;
}
tr .tablehead td {
  text-align: center;
}
.TableRow {
  cursor: pointer;
  background: #f8f8f9;
}
tr .tablehead td,
tr .TableRow td {
  background: #f8f8f9 !important;
}

.TableRowOver td:first-child,
.TableRow td:first-child {
  text-align: center;
}
.TableRow:hover,
.TableRowOver {
  background-color: #2db7f5;
  color: #2d2d2d;
}
.Footer {
  position: absolute;
  bottom: 1px;
  right: 1px;
  left: 1px;
  border: 1px solid #dcdee2;
  background-color: #f8f8f9;
}
.combobox-inp {
  outline: none;
  display: inline-block;
  width: 100%;
  height: 32px;
  line-height: 1.5;
  padding: 4px 7px;
  font-size: 14px;
  border: 1px solid #dcdee2;
  border-radius: 4px;
  color: #515a6e;
  background-color: #fff;
  background-image: none;
  position: relative;
  cursor: text;
  transition: border 0.2s ease-in-out, background 0.2s ease-in-out,
    box-shadow 0.2s ease-in-out;
}
.noborder {
  border: none;
}
.combobox-inp:hover {
  border-color: #57a3f3;
}
.combobox-inp::focus {
  border-color: #57a3f3;
  outline: 0;
  box-shadow: 0 0 0 2px rgba(45, 140, 240, 0.2);
}
.combobox-inp-disabled {
  background-color: #f3f3f3;
  opacity: 1;
  cursor: not-allowed;
  color: #ccc;
}
</style>
<script>
import axios from "@/api/axios-request";

var nextPageTimer = null;
var stopBubble = function (e) {
  if (e && e.stopPropagation) {
    e.stopPropagation();
  } else {
    e.cancelBubble = true;
  }
};

/**
 * 阻止浏览器默认动作
 */
var stopDefault = function (e) {
  if (e && e.preventDefault) {
    e.preventDefault();
  } else {
    e.returnValue = false;
  }
  return false;
};

var _checkHandle = null;

export default {
  name: "Combobox",
  data() {
    return {
      isFocus: false,
      isOpen: false,
      value: this.currentValue,
      datas: {
        headers: [],
        rows: [],
        columns: [],
        codes: [],
        pageCount: 1,
        pageIndex: 1,
        rowCount: 0,
      },
    };
  },
  props: {
    name: {
      type: String,
    },
    comboxWidth: {
      type: [Number, String],
      default: 400,
    },
    comboxHeight: { type: [Number, String], default: 400 },
    url: String,
    pageSize: {
      type: [Number, String],
      default: 10,
    },
    paramName: {
      type: String,
      default: "keyword",
    },
    border: {
      type: Boolean,
      default: true,
    },
    currentValue: [Number, String],
    disabled: {
      type: Boolean,
      default: false,
    },
    field: {
      type: String,
    },
    modelValue: {
      type: [Number, String],
    },
    codeValue: [Number, String],
    nameValue: [Number, String],
    codeField: String,
    nameField: String,
  },

  computed: {
    width() {
      return this.comboxWidth;
    },
    height() {
      return this.comboxHeight;
    },
    containerid() {
      return this.name + "Container";
    },

    start() {
      return (this.datas.pageIndex - 1) * this.pageSize + 1;
    },
    end() {
      return this.datas.pageIndex * this.pageSize;
    },
    filterdata() {
      var rows = this.datas.rows;
      return rows;
    },
  },
  components: {},
  mounted() {
    //console.info('挂载',this.currentValue);
    this.$refs.input.value = this.currentValue || "";
    this.$nextTick(function () {
      this.$refs.input.value = this.currentValue || "";
    });
  },
  watch: {
    currentValue: function () {
      // console.info('监控',val);
      this.$refs.input.value = this.currentValue || "";
    },
    code(val) {
      this.ComboBox_StartWebRequest();
    },
  },
  methods: {
    check(key) {
      return this.datas.codes.includes(key);
    },
    computeW: function () {
      return (this.comboxWidth - 25) / this.datas.headers.length + "px";
    },
    ComboBox_Init() {
      this.isOpen = true;
    },
    ComboBox_OnFocus() {
      this.$refs.input.select();
      this.isFocus = true;
    },
    ComboBox_ShowDownList() {
      if (!this.isOpen) {
        this.ComboBox_Init();
        this.ComboBox_StartWebRequest();
      }
    },
    ComboBox_OnBlur() {
      this.$refs.input.lastValue = null;
      this.ComboBox_Hidden();
    },
    ComboBox_StartWebRequest() {
      let url = this.url;
      if(!url){
        return;
      }
      let params = {};
      params.pageSize = this.pageSize;
      params.pageNum = this.datas.pageIndex;
      params.keyword = this.$refs.input.value;
      
      axios
        .request({
          url: url,
          method: "get",
          params: params,
        })
        .then((res) => {
          if (res.status == "1") {
            var codes = [];
            var headers = res.data.headers;
            var width = this.comboxWidth;
            if (!headers) {
              return;
            }
            var l = headers.length;
            var columns = this.datas.columns;
            columns.length = 0;
            columns.push("25px");
            var totalWidth = width - 25;
            var widthHead = headers.filter((elem) => elem.width);
            widthHead.forEach((elem) => {
              totalWidth = totalWidth - elem.width;
            });
            var avgWidth = totalWidth / (l - widthHead.length);

            //console.info(widthHead);
            headers.forEach(function (head) {
              codes.push(head.code);
              if (head.width) {
                columns.push(head.width + "px");
              } else {
                columns.push(avgWidth + "px");
              }
            });

            //处理数据
            var rows = [];
            var tempRows = res.data.rows;
            tempRows.forEach((element) => {
              var obj = {};
              obj.olddata = element;
              obj.data = [];

              codes.forEach((code) => {
                obj.data.push(element[code]);
              });
              rows.push(obj);
              this.datas.rows.push(obj);
            });
            //this.datas.rows.concat(rows);

            this.datas.codes = codes;
            this.datas.headers = res.data.headers;
            this.datas.rowCount = res.data.rowCount;
            this.datas.pageCount = Math.ceil(res.data.rowCount / this.pageSize);
            this.dict.usersList = res.data.usersList;
          }
        })
        .catch((err) => {})
        .finally(() => {});
    },
    ComboBox_OnKeyDown(e) {
      var sender = this.$refs.input;
      var container = this.$refs.container;
      var attRequired = this.$refs.input.getAttribute("required");
      var pageCount = this.datas.pageCount;
      if (!container && e.keyCode == 13) {
        if (attRequired != "true") {
          return true;
        } else {
          if (sender.value != sender.getAttribute("oldValue")) {
            e.cancelBubble = true;
          }
          return false;
        }
      }

      this.ComboBox_Init();

      if (e.keyCode == 8 || e.keyCode == 46) {
        this.$emit("clear");
      }
      var table = this.$refs.datatable;
      if (table) {
        var rowIndex = 0;
        if (table.getAttribute("rowIndex")) {
          rowIndex = parseInt(table.getAttribute("rowIndex"));
        }

        switch (e.keyCode) {
          case 38:
            e.cancelBubble = false;
            if (table.rows[rowIndex - 1]) {
              if (table.rows[rowIndex]) {
                table.rows[rowIndex].className = "TableRow";
              }

              table.rows[rowIndex - 1].className = "TableRowOver";
              table.setAttribute("rowIndex", rowIndex - 1);
              if (
                table.rows[rowIndex - 1].offsetTop <
                container.childNodes[1].scrollTop
              ) {
                container.childNodes[1].scrollTop =
                  table.rows[rowIndex - 1].offsetTop;
              }
            }
            return;
          case 40:
            //按键
            e.cancelBubble = false;
            var pageIndex = sender.getAttribute("pageIndex")
              ? parseInt(sender.getAttribute("pageIndex"))
              : 1;
            if (table.rows[rowIndex + 1]) {
              table.rows[rowIndex].className = "TableRow";
              table.rows[rowIndex + 1].className = "TableRowOver";
              table.setAttribute("rowIndex", rowIndex + 1);
              if (
                table.rows[rowIndex + 1].offsetTop +
                  table.rows[rowIndex + 1].offsetHeight >
                container.childNodes[1].scrollTop +
                  container.childNodes[1].clientHeight
              ) {
                container.childNodes[1].scrollTop =
                  table.rows[rowIndex + 1].offsetTop;
              }
            } else if (pageIndex < pageCount) {
              if (nextPageTimer) {
                window.clearTimeout(nextPageTimer);
              }
              nextPageTimer = window.setTimeout(() => {
                pageIndex++;
                this.datas.pageIndex = pageIndex;
                //console.log("pageIndex", this.datas.pageIndex);
                sender.setAttribute("pageIndex", pageIndex);
                this.ComboBox_StartWebRequest();
              }, 1000);
            } else {
              console.warn("参数错误，请检查");
            }
            return;
          case 13:
            e.cancelBubble = false;
            var isEnter = true;
            var enterCallback = sender.getAttribute("entercallback");
            if (enterCallback) {
              isEnter = eval(enterCallback)(sender);
              if (!isEnter) {
                stopBubble(e);
                stopDefault(e);
                return false;
              }
            }
            if (table.rows[rowIndex]) {
              var data = table.rows[rowIndex].getAttribute("data");
              data = JSON.parse(data);
              this.ComboBox_ItemSelected(data);
              sender.setAttribute("oldValue", sender.value);
              this.checkHidden(sender, e);
            } else {
              if (
                sender.value != sender.getAttribute("oldValue") &&
                sender.getAttribute("required") == "true"
              ) {
                e.cancelBubble = true;
              }
              this.checkHidden(sender, e);
            }
            this.isOpen = false;
            return;
        }
      }
      if (e.keyCode == 13) {
        container.style.display = "none";
        this.checkHidden(sender, e);

        if (attRequired != "true") {
          return true;
        } else {
          if (sender.value != sender.getAttribute("oldValue")) {
            sender.value = "";
            e.cancelBubble = true;
          }
          return false;
        }
      }
      this.getData(sender);
    },
    checkHidden(sender, e) {},
    checkData() {
      var sender = this.$refs.input;
      // console.info('对比值',sender.value,sender.lastValue);
      if (sender.value != sender.lastValue) {
        this.LoadData();
        sender.lastValue = sender.value;
      }
    },
    LoadData() {
      this.pageIndex = 1;
      this.datas.rows = [];
      this.ComboBox_StartWebRequest();
    },
    getData() {
      if (_checkHandle) {
        window.clearTimeout(_checkHandle);
      }
      var vm = this;
      _checkHandle = window.setTimeout(function () {
        vm.checkData();
      }, 1);
    },
    ComboBox_ItemSelected(data) {
      if (this.field) {
        this.$refs.input.value = data[this.field];
        this.$emit("update:modelValue", data[this.field]);
      }
      
      if (this.codeField) {
        // console.info(this.codeField,data[this.codeField]);
        this.$emit("update:codeValue", data[this.codeField]);
      }
      if (this.nameField) {
        this.$emit("update:nameValue", data[this.nameField]);
      }
      var selection = this.$refs.input;
      this.$emit("command", selection, data);
      this.ComboBox_Hidden();
    },
    ComboBox_Hidden() {
      var sender = this.$refs.input;
      sender.setAttribute("pageIndex", "1");
      this.datas.pageIndex = 1;
      var container = this.$refs.container;
      if (!container) {
        return;
      }
      this.datas.rows = [];
      this.datas.headers = [];
      this.isOpen = false;

      this.$refs.datatable.setAttribute("rowindex", "0");
    },
    setVal(val) {
      this.$refs.input.value = val;
    },
    getVal() {
      return this.$refs.input.value;
    },
  },
};
</script>
