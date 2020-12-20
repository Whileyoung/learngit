<template>
  <div class="x-column">
    <!-- <section class="x-columns_mask"></section> -->
    <ul ref="wrapper" :style="wrapperStyle" class="x-columns_wrapper">
      <li
        :style="optionStyle"
        class="x-column_item"
        :class="{ 'x-column-selected': index === currentIndex }"
        v-for="(option, index) in options"
        :key="getOptionText(option)"
        v-html="getOptionText(option)"
        @click="onClickItem(index)"
        onTransitionend="{this.onTransitionEnd}"
      />
    </ul>
  </div>
</template>

<script>
const { hasOwnProperty } = Object.prototype;
/**默认持续时间 */
const DEFAULT_DURATION = 200;
const MOMENTUM_LIMIT_TIME = 300;
const MOMENTUM_LIMIT_DISTANCE = 15;
/** */
function getElementTranslateY(element) {
  const style = window.getComputedStyle(element);
  const transform = style.transform || style.webkitTransform;
  const translateY = transform.slice(7, transform.length - 1).split(", ")[5];
  return Number(translateY);
}
/** */
function isDef(val) {
  return val !== undefined && val !== null;
}
/** */
function isObject(val) {
  return val !== null && typeof val === "object";
}
/** */
function isDisabled(option) {
  return isObject(option) && option.disabled;
}
function deepAssign(to, from) {
  Object.keys(from).forEach((key) => {
    assignKey(to, from, key);
  });
  return to;
}
/** */
function assignKey(to, from, key) {
  const val = from[key];

  if (!isDef(val)) {
    return;
  }

  if (!hasOwnProperty.call(to, key) || !isObject(val)) {
    to[key] = val;
  } else {
    // eslint-disable-next-line @typescript-eslint/no-use-before-define
    to[key] = deepAssign(Object(to[key]), from[key]);
  }
}
/** */
function cloneDeep(obj) {
  if (Array.isArray(obj)) {
    return obj.map((item) => cloneDeep(item));
  }

  if (typeof obj === "object") {
    return deepAssign({}, obj);
  }

  return obj;
}
/** */
const MIN_DISTANCE = 10;
function getDirection(x, y) {
  if (x > y && x > MIN_DISTANCE) {
    return "horizontal";
  }

  if (y > x && y > MIN_DISTANCE) {
    return "vertical";
  }

  return "";
}
/** */
function range(num, min, max) {
  return Math.min(Math.max(num, min), max);
}
function stopPropagation(event) {
  event.stopPropagation();
}

function preventDefault(event, isStopPropagation) {
  /* istanbul ignore else */
  if (typeof event.cancelable !== "boolean" || event.cancelable) {
    event.preventDefault();
  }

  if (isStopPropagation) {
    stopPropagation(event);
  }
}
export default {
  props: {
    valueKey: String,
    readonly: Boolean,
    allowHtml: Boolean,
    className: String,
    itemHeight: Number,
    defaultIndex: Number,
    swipeDuration: [Number, String],
    visibleItemCount: [Number, String],
    initialOptions: {
      type: Array,
      default: () => [],
    },
  },
  data() {
    return {
      /**方向 */
      direction: "",
      /**偏离量 */
      offset: 0,
      /**持续时间 */
      duration: 0,
      options: cloneDeep(this.initialOptions),
      currentIndex: this.defaultIndex,
    };
  },
  /** */
  created() {
    if (this.$parent.children) {
      this.$parent.children.push(this);
    }

    this.setIndex(this.currentIndex);
  },
  mounted() {
    this.bindTouchEvent(this.$el);
  },
  destroyed() {
    const { children } = this.$parent;
    if (children) {
      children.splice(children.indexOf(this), 1);
    }
  },
  watch: {
    initialOptions: "setOptions",
    defaultIndex(val) {
      this.setIndex(val);
    },
  },
  computed: {
    wrapperStyle() {
      return {
        transform: `translate3d(0, ${this.offset + this.baseOffset}px, 0)`,
        transitionDuration: `${this.duration}ms`,
        transitionProperty: this.duration ? "all" : "none",
      };
    },
    optionStyle() {
      return {
        height: `${this.itemHeight}px`,
      };
    },
    count() {
      return this.options.length;
    },

    baseOffset() {
      return (this.itemHeight * (this.visibleItemCount - 1)) / 2;
    },
  },
  methods: {
    touchStart(event) {
      this.resetTouchStatus();
      this.startX = event.touches[0].clientX;
      this.startY = event.touches[0].clientY;
    },

    touchMove(event) {
      const touch = event.touches[0];
      this.deltaX = touch.clientX - this.startX;
      this.deltaY = touch.clientY - this.startY;
      this.offsetX = Math.abs(this.deltaX);
      this.offsetY = Math.abs(this.deltaY);
      this.direction =
        this.offsetX > this.offsetY && this.offsetX > 10
          ? "horizontal"
          : this.offsetX < this.offsetY && this.offsetY > 10
          ? "vertical"
          : "";
    },

    resetTouchStatus() {
      this.direction = "";
      this.deltaX = 0;
      this.deltaY = 0;
      this.offsetX = 0;
      this.offsetY = 0;
    },
    bindTouchEvent(el) {
      let passiveSupported = false
      const { onTouchStart, onTouchMove, onTouchEnd } = this;
      const willPreventDefault = passiveSupported ? { passive: false } : false;
      const willNotPreventDefault = passiveSupported
        ? { passive: true }
        : false;
      el.addEventListener("touchstart", onTouchStart, willNotPreventDefault);
      el.addEventListener("touchmove", onTouchMove, willPreventDefault);

      if (onTouchEnd) {
        el.addEventListener("touchend", onTouchEnd, willNotPreventDefault);
        el.addEventListener("touchcancel", onTouchEnd, willNotPreventDefault);
      }
    },
    setOptions(options) {
      if (JSON.stringify(options) !== JSON.stringify(this.options)) {
        this.options = cloneDeep(options);
        this.setIndex(this.defaultIndex);
      }
    },

    onTouchStart(event) {
      if (this.readonly) {
        return;
      }

      this.touchStart(event);

      if (this.moving) {
        const translateY = getElementTranslateY(this.$refs.wrapper);
        this.offset = Math.min(0, translateY - this.baseOffset);
        this.startOffset = this.offset;
        console.log(translateY);
      } else {
        this.startOffset = this.offset;
      }

      this.duration = 0;
      this.transitionEndTrigger = null;
      this.touchStartTime = Date.now();
      this.momentumOffset = this.startOffset;
    },

    onTouchMove(event) {
      if (this.readonly) {
        return;
      }

      this.touchMove(event);

      if (this.direction === "vertical") {
        this.moving = true;
        preventDefault(event, true);
      }

      this.offset = range(
        this.startOffset + this.deltaY,
        -(this.count * this.itemHeight),
        this.itemHeight
      );

      const now = Date.now();
      if (now - this.touchStartTime > MOMENTUM_LIMIT_TIME) {
        this.touchStartTime = now;
        this.momentumOffset = this.offset;
      }
    },

    onTouchEnd() {
      if (this.readonly) {
        return;
      }

      const distance = this.offset - this.momentumOffset;
      const duration = Date.now() - this.touchStartTime;
      const allowMomentum =
        duration < MOMENTUM_LIMIT_TIME &&
        Math.abs(distance) > MOMENTUM_LIMIT_DISTANCE;

      if (allowMomentum) {
        this.momentum(distance, duration);
        return;
      }

      const index = this.getIndexByOffset(this.offset);
      this.duration = DEFAULT_DURATION;
      this.setIndex(index, true);

      // compatible with desktop scenario
      // use setTimeout to skip the click event triggered after touchstart
      setTimeout(() => {
        this.moving = false;
      }, 0);
    },

    onTransitionEnd() {
      this.stopMomentum();
    },

    onClickItem(index) {
      if (this.moving || this.readonly) {
        return;
      }

      this.transitionEndTrigger = null;
      this.duration = DEFAULT_DURATION;
      this.setIndex(index, true);
    },

    adjustIndex(index) {
      index = range(index, 0, this.count);

      for (let i = index; i < this.count; i++) {
        if (!isDisabled(this.options[i])) return i;
      }

      for (let i = index - 1; i >= 0; i--) {
        if (!isDisabled(this.options[i])) return i;
      }
    },

    getOptionText(option) {
      if (isObject(option) && this.valueKey in option) {
        return option[this.valueKey];
      }
      return option;
    },

    setIndex(index, emitChange) {
      index = this.adjustIndex(index) || 0;

      const offset = -index * this.itemHeight;

      const trigger = () => {
        if (index !== this.currentIndex) {
          this.currentIndex = index;

          if (emitChange) {
            this.$emit("change", index);
          }
        }
      };

      // trigger the change event after transitionend when moving
      if (this.moving && offset !== this.offset) {
        this.transitionEndTrigger = trigger;
      } else {
        trigger();
      }
      this.offset = offset;
    },

    setValue(value) {
      const { options } = this;
      for (let i = 0; i < options.length; i++) {
        if (this.getOptionText(options[i]) === value) {
          return this.setIndex(i);
        }
      }
    },
    getValue() {
      return this.options[this.currentIndex];
    },
    getIndexByOffset(offset) {
      return range(Math.round(-offset / this.itemHeight), 0, this.count - 1);
    },
    momentum(distance, duration) {
      const speed = Math.abs(distance / duration);
      distance = this.offset + (speed / 0.003) * (distance < 0 ? -1 : 1);
      const index = this.getIndexByOffset(distance);
      this.duration = +this.swipeDuration;
      this.setIndex(index, true);
    },
    stopMomentum() {
      this.moving = false;
      this.duration = 0;
      if (this.transitionEndTrigger) {
        this.transitionEndTrigger();
        this.transitionEndTrigger = null;
      }
    },
  },
};
</script>
<style>
.x-picker {
  position: relative;
  background-color: white;
  user-select: none;
}
.x-picker_toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 44px;
}

.x-picker_cancel,
.x-picker_confirm {
  height: 100%;
  padding: 0 16px;
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
.x-column-selected {
  color: black;
}
.x-column_item {
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