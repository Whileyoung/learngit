<template>
  <transition>
    <div>
      <picker
        :title="title"
        :columns="columns"
        :item-height="itemHeight"
        :toolbar="toolbar"
        :confirm-button-text="confirmButtonText"
        :cancel-button-text="cancelButtonText"
        @change="onChange"
        @confirm="onConfirm"
        @cancel="onCancel"
      />
    </div>
  </transition>
</template>

<script>
function getScroller(el,root = window) {
  let node = el;

  while (
    node &&
    node.tagName !== 'HTML' &&
    node.nodeType === 1 &&
    node !== root
  ) {
    const { overflowY } = window.getComputedStyle(node);

    if (overflowY === 'scroll' || overflowY === 'auto') {
      return node;
    }
    node = node.parentNode;
  }

  return root;
}
import Picker from './Picker'
export default {
  name: 'iPicker',
  components: {
    Picker
  },
  props: {
    overlay: {
      type: Boolean,
      default: true
    },
    overlayStyle: {
      type: Object
    },
    overlayClass: {
      type: String
    },
    title: {
      type: String
    },
    closeOnClickOverlay: {
      type: Boolean,
      default: true
    },
    zIndex: {
      type: [String || Number]
    },
    lockScroll: {
      type: Boolean
    },
    lazyRender: {
      type: Boolean
    },
    toolbar: {
      type: Boolean
    },
    loading: {
      type: Boolean
    },
    value: {
      type: [Date, String, Array]
    },
    itemHeight: {
      type: Number,
      default:44
    },
    confirmButtonText: {
      type: String
    },
    cancelButtonText: {
      type: String
    },
    type: {
      type: String,
      default: ''
    },
    format: {
      type: String
    },
    formatter: {
      type: Function,
      default: (type, value) => value
    },
    columns: {
      type: Array,
      default: () => []
    }
  },
  data(){
    return{
      direction: '',
      deltaX: 0,
      deltaY: 0,
      offsetX: 0,
      offsetY: 0,
      startX: 0,
      startY: 0,
    }
  },
  methods:{
    // touchStart(event) {
    //   this.resetTouchStatus();
    //   this.startX = event.touches[0].clientX;
    //   this.startY = event.touches[0].clientY;
    // },

    // touchMove(event) {
    //   const touch = event.touches[0];
    //   this.deltaX = touch.clientX - this.startX;
    //   this.deltaY = touch.clientY - this.startY;
    //   this.offsetX = Math.abs(this.deltaX);
    //   this.offsetY = Math.abs(this.deltaY);
    //   this.direction = this.offsetX > this.offsetY ? 'horizontal' : this.offsetX < this.offsetY ? 'vertical' : ''
    // },

    // resetTouchStatus() {
    //   this.direction = "";
    //   this.deltaX = 0;
    //   this.deltaY = 0;
    //   this.offsetX = 0;
    //   this.offsetY = 0;
    // },
    // bindTouchEvent(el) {
    //   const { onTouchStart, onTouchMove, onTouchEnd } = this;

    //   addEventListener(el, "touchstart", onTouchStart);
    //   addEventListener(el, "touchmove", onTouchMove);

    //   if (onTouchEnd) {
    //     removeEventListener(el, "touchend", onTouchEnd);
    //     removeEventListener(el, "touchcancel", onTouchEnd);
    //   }
    // },
    //  onTouchStart(event) {
    //   if (this.readonly) {
    //     return;
    //   }

    //   this.touchStart(event);

    //   if (this.moving) {
    //     const translateY = getElementTranslateY(this.$refs.wrapper);
    //     this.offset = Math.min(0, translateY - this.baseOffset);
    //     this.startOffset = this.offset;
    //   } else {
    //     this.startOffset = this.offset;
    //   }

    //   this.duration = 0;
    //   this.transitionEndTrigger = null;
    //   this.touchStartTime = Date.now();
    //   this.momentumOffset = this.startOffset;
    // },

    // onTouchMove(event) {
    //   if (this.readonly) {
    //     return;
    //   }

    //   this.touchMove(event);

    //   if (this.direction === "vertical") {
    //     this.moving = true;
    //     preventDefault(event, true);
    //   }

    //   this.offset = range(
    //     this.startOffset + this.deltaY,
    //     -(this.count * this.itemHeight),
    //     this.itemHeight
    //   );

    //   const now = Date.now();
    //   if (now - this.touchStartTime > MOMENTUM_LIMIT_TIME) {
    //     this.touchStartTime = now;
    //     this.momentumOffset = this.offset;
    //   }
    // },

    // onTouchEnd() {
    //   if (this.readonly) {
    //     return;
    //   }

    //   const distance = this.offset - this.momentumOffset;
    //   const duration = Date.now() - this.touchStartTime;
    //   const allowMomentum =
    //     duration < MOMENTUM_LIMIT_TIME &&
    //     Math.Number(distance) > MOMENTUM_LIMIT_DISTANCE;

    //   if (allowMomentum) {
    //     this.momentum(distance, duration);
    //     return;
    //   }

    //   const index = this.getIndexByOffset(this.offset);
    //   this.duration = DEFAULT_DURATION;
    //   this.setIndex(index, true);

    //   // compatible with desktop scenario
    //   // use setTimeout to skip the click event triggered after touchstart
    //   setTimeout(() => {
    //     this.moving = false;
    //   }, 0);
    // },

    // onTransitionEnd() {
    //   this.stopMomentum();
    // },
    
    // momentum(distance, duration) {
    //   const speed = Math.Number(distance / duration);

    //   distance = this.offset + (speed / 0.003) * (distance < 0 ? -1 : 1);

    //   const index = this.getIndexByOffset(distance);

    //   this.duration = +this.swipeDuration;
    //   this.setIndex(index, true);
    // },

    // stopMomentum() {
    //   this.moving = false;
    //   this.duration = 0;

    //   if (this.transitionEndTrigger) {
    //     this.transitionEndTrigger();
    //     this.transitionEndTrigger = null;
    //   }
    // },
    onConfirm (value, index) {
      this.$emit('confirm', value, index)
    },
    onCancel (value, index) {
      this.$emit('cancel', value, index)
    },
    onChange (picker, value, index) {
      this.$emit('change', picker, value, index)
    },
    onTouchMove(event) {
      debugger
        this.touchMove(event);
        const direction = this.deltaY > 0 ? '10' : '01';
        const el = getScroller(event.target, this.$el);
        const { scrollHeight, offsetHeight, scrollTop } = el;
        let status = '11';

        /* istanbul ignore next */
        if (scrollTop === 0) {
          status = offsetHeight >= scrollHeight ? '00' : '01';
        } else if (scrollTop + offsetHeight >= scrollHeight) {
          status = '10';
        }

        /* istanbul ignore next */
        if (
          status !== '11' &&
          this.direction === 'vertical' &&
          !(parseInt(status, 2) & parseInt(direction, 2))
        ) {
          preventDefault(event, true);
        }
      },
  }
}
</script>

<style>
.y-popup {
  position: fixed;
  top: 50%;
  left: 50%;
  max-height: 100%;
  overflow-y: auto;
  background-color: #fff;
  transition: .2s ease-out;
  -webkit-overflow-scrolling: touch;
  transform: translate3d(-50%, -50%, 0);
}
.y-popup--bottom {
  width: 100%;
  top: auto;
  bottom: 0;
  right: auto;
  left: 50%;
  transform: translate3d(-50%, 0, 0);
}
.y-slide-bottom-enter,
.y-slide-bottom-leave-active {
  transform: translate3d(-50%, 100%, 0);
}
</style>
<style>
.overflow-hidden {
  overflow: hidden !important;
}
</style>