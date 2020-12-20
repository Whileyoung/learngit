<template>
<div class="x-picker">
  <div class="x-picker_toolbar" v-if="toolbar">
      <slot>
        <div class="x-picker_cancel" @click="emit('cancel')">{{ cancelButtonText || '取消' }}</div>
        <div class="x-picker_title" v-if="title" v-text="title"/>
        <div class="x-picker_confirm" @click="emit('confirm')">{{ confirmButtonText || '确认' }}</div>
      </slot>
  </div>
  <div class="x-columns" :style="columnsStyle" @ontouchmove.prevent>
    <picker-column
    v-for="(item, columnIndex) in formattedColumns"
    :key="columnIndex"
    :valueKey="valueKey"
    :itemHeight="itemHeight"
    :className="item.className"
    :allowHtml="allowHtml"
    :defaultIndex="defaultIndex"
    :swipeDuration="swipeDuration"
    :visibleItemCount="visibleItemCount"
    :initialOptions="item.values"
    @change="onChange(columnIndex)"
    />
    <div class="x-columns_mask" :style="maskStyle"></div>
    <div class="x-columns_frame van-hairline-unset--top-bottom" :style="frameStyle"></div>
  </div>
</div>
</template>
<script>
import PickerColumn from './PickerColumn';
/**默认选项高度 */
const DEFAULT_ITEM_HEIGHT = 44;
export default {
  components: {
    PickerColumn
    },
  props:{
      /** */
    title: String,
    readonly: Boolean,
    itemHeight: [Number, String],
    toolbar: Boolean,
    cancelButtonText: String,
    confirmButtonText: String,
    allowHtml: {
      type: Boolean,
      default: true,
    },
    visibleItemCount: {
      type: [Number, String],
      default: 6,
    },
    swipeDuration: {
      type: [Number, String],                                                  
      default: 1000,
    },
    defaultIndex: {
      type: [Number, String],
      default: 0,
    },
    columns: {
      type: Array,
      default: () => [],
    },
    valueKey: {
      type: String,
      default: 'label',
    },
  },
    /** */
    data() {
    return {
      children: [],
      formattedColumns: [],
    }
    }, 
    computed: {
    itemPxHeight() {
      return this.itemHeight ? this.itemHeight : DEFAULT_ITEM_HEIGHT;
    },
    wrapHeight(){
      return this.itemPxHeight * this.visibleItemCount;
    },
    frameStyle(){
      return {
        height: `${this.itemPxHeight}px`
      }
    },
    columnsStyle(){
      return{
        height: `${this.wrapHeight}px`
      }
    },
    maskStyle(){
      return{
        backgroundSize: `100% ${(this.wrapHeight - this.itemPxHeight) / 2}px`
      }
    },
    dataType() {
      const { columns } = this;
      const firstColumn = columns[0] || {};

      if (firstColumn.children) {
        return 'cascade';
      }

      if (firstColumn.values) {
        return 'object';
      }

      return 'text';
    },
  },
  watch: {
    columns: {
      handler: 'format',
      immediate: true,
    },
  },
  methods:{
      format() {
      const { columns, dataType } = this;
      if (dataType === 'text') {
        this.formattedColumns = [{ values: columns }];
      } else if (dataType === 'cascade') {
        this.formatCascade();
      } else {
        this.formattedColumns = columns;
      }
    },

    formatCascade() {
      const formatted = [];

      let cursor = { children: this.columns };

      while (cursor && cursor.children) {
        const { children } = cursor;
        let defaultIndex = cursor.defaultIndex || +this.defaultIndex;

        while (children[defaultIndex] && children[defaultIndex].disabled) {
          if (defaultIndex < children.length - 1) {
            defaultIndex++;
          } else {
            defaultIndex = 0;
            break;
          }
        }

        formatted.push({
          values: cursor.children,
          className: cursor.className,
          defaultIndex,
        });

        cursor = children[defaultIndex];
      }

      this.formattedColumns = formatted;
    },

    emit(event) {
      if (this.dataType === 'text') {
        this.$emit(event, this.getColumnValue(0), this.getColumnIndex(0));
      } else {
        let values = this.getValues();

        // compatible with old version of wrong parameters
        // should be removed in next major version
        // see: https://github.com/youzan/vant/issues/5905
        if (this.dataType === 'cascade') {
          values = values.map((item) => item[this.valueKey]);
        }

        this.$emit(event, values, this.getIndexes());
      }
    },

    onCascadeChange(columnIndex) {
      let cursor = { children: this.columns };
      const indexes = this.getIndexes();

      for (let i = 0; i <= columnIndex; i++) {
        cursor = cursor.children[indexes[i]];
      }

      while (cursor && cursor.children) {
        columnIndex++;
        this.setColumnValues(columnIndex, cursor.children);
        cursor = cursor.children[cursor.defaultIndex || 0];
      }
    },

    onChange(columnIndex) {
      if (this.dataType === 'cascade') {
        this.onCascadeChange(columnIndex);
      }

      if (this.dataType === 'text') {
        this.$emit(
          'change',
          this,
          this.getColumnValue(0),
          this.getColumnIndex(0)
        );
      } else {
        let values = this.getValues();
        if (this.dataType === 'cascade') {
          values = values.map((item) => item[this.valueKey]);
        }

        this.$emit('change', this, values, columnIndex);
      }
    },

    // get column instance by index
    getColumn(index) {
      return this.children[index];
    },

    // @exposed-api
    // get column value by index
    getColumnValue(index) {
      const column = this.getColumn(index);
      return column && column.getValue();
    },

    // @exposed-api
    // set column value by index
    setColumnValue(index, value) {
      const column = this.getColumn(index);

      if (column) {
        column.setValue(value);

        if (this.dataType === 'cascade') {
          this.onCascadeChange(index);
        }
      }
    },

    // @exposed-api
    // get column option index by column index
    getColumnIndex(columnIndex) {
      return (this.getColumn(columnIndex) || {}).currentIndex;
    },

    // @exposed-api
    // set column option index by column index
    setColumnIndex(columnIndex, optionIndex) {
      const column = this.getColumn(columnIndex);

      if (column) {
        column.setIndex(optionIndex);

        if (this.dataType === 'cascade') {
          this.onCascadeChange(columnIndex);
        }
      }
    },

    // @exposed-api
    // get options of column by index
    getColumnValues(index) {
      return (this.children[index] || {}).options;
    },

    // @exposed-api
    // set options of column by index
    setColumnValues(index, options) {
      const column = this.children[index];

      if (column) {
        column.setOptions(options);
      }
    },

    // @exposed-api
    // get values of all columns
    getValues() {
      return this.children.map((child) => child.getValue());
    },

    // @exposed-api
    // set values of all columns
    setValues(values) {
      values.forEach((value, index) => {
        this.setColumnValue(index, value);
      });
    },

    // @exposed-api
    // get indexes of all columns
    getIndexes() {
      return this.children.map((child) => child.currentIndex);
    },

    // @exposed-api
    // set indexes of all columns
    setIndexes(indexes) {
      indexes.forEach((optionIndex, columnIndex) => {
        this.setColumnIndex(columnIndex, optionIndex);
      });
    },

    // @exposed-api
    confirm() {
      this.children.forEach((child) => child.stopMomentum());
      this.emit('confirm');
    },

    cancel() {
      this.emit('cancel');
    },
  }
  }
</script>
<style>
.x-picker {
  position: relative;
  background-color: white;
  user-select: none;
}
  .x-picker_toolbar{
    display: flex;
    align-items: center;
    justify-content: space-between;
    height: 44px;
  }

  .x-picker_cancel,
  .x-picker_confirm
   {
    height: 100%;
    padding: 0  16px;
    font-size: 14px;
    background-color: transparent;
    border: none;
    cursor: pointer;
    color: #969799;
  }

  .x-picker_title {
    max-width: 50%;
    font-weight: 500;
    font-size: 16px;
    line-height: 20px;
    text-align: center;
  }

  .x-columns {
    position: relative;
    display: flex;
    cursor: grab;
  }


  .x-columns_frame {
    position: absolute;
    top: 50%;
    right: 16px;
    left: 16px;
    z-index: 2;
    transform: translateY(-50%);
    pointer-events: none;
        border-bottom: 1px solid #969799;
    border-top: 1px solid #969799;
  }
  .x-columns_mask {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1;
    width: 100%;
    height: 100%;
    background-image: linear-gradient(
        180deg,
        hsla(0, 0%, 100%, 0.9),
        hsla(0, 0%, 100%, 0.4)
      ),
      linear-gradient(0deg, hsla(0, 0%, 100%, 0.9), hsla(0, 0%, 100%, 0.4));
    background-repeat: no-repeat;
    background-position: top, bottom;
    transform: translateZ(0);
    pointer-events: none;
  }

  .x-column {
    flex: 1;
    overflow: hidden;
    font-size: 16px;
  }
  .x-column ul {
  margin: 0;
  padding: 0;
  list-style: none;
}
    .x-columns_wrapper {
      transition-timing-function: cubic-bezier(0.23, 1, 0.68, 1);
    }
.x-column-selected{
  color: black;
}
    .x-column_item{
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 0 4px;
      color: black;
    }
      .x--disabled {
        cursor: not-allowed;
        opacity: 0.5;
      }

</style>