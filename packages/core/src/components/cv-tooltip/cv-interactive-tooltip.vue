<template>
  <div class="cv-interactive-tooltip">
    <div :id="`${uid}-label`" :class="`${carbonPrefix}--tooltip__label`">
      <slot name="label"></slot>

      <button
        :aria-expanded="dataVisible ? 'true' : 'false'"
        :aria-labelledby="`${uid}-label`"
        :class="`${carbonPrefix}--tooltip__trigger`"
        :aria-controls="`${uid}`"
        aria-haspopup="true"
        ref="trigger"
        type="button"
        @click="toggle"
        @keydown.tab="onTriggerTab"
        @focusout="checkFocusOut"
      >
        <slot name="trigger">
          <Information16 />
        </slot>
      </button>
    </div>

    <div
      :id="uid"
      aria-hidden="true"
      :data-floating-menu-direction="direction"
      :class="[
        `${carbonPrefix}--tooltip`,
        {
          [`${carbonPrefix}--tooltip--shown`]: dataVisible,
          [`${carbonPrefix}--tooltip--${direction}`]: direction,
          [`${carbonPrefix}--tooltip--align-${alignment}`]: alignment,
        },
      ]"
      ref="popup"
      role="dialog"
      :aria-describedby="`${uid}-body`"
      :aria-labelledby="`${uid}-label`"
      @focusout="checkFocusOut"
      :style="{ left: left + 'px', top: top + 'px' }"
      tabindex="-1"
      @mousedown.prevent="preventFocusOut"
    >
      <div
        class="cv-interactive-tooltip__before-content"
        ref="beforeContent"
        tabindex="0"
        style="position: absolute; left: -9999px; width: 1px; height: 1px"
        @focus="focusBeforeContent"
      />
      <span :class="`${carbonPrefix}--tooltip__caret`"></span>
      <div :class="`${carbonPrefix}--tooltip__content`">
        <slot name="content"></slot>
      </div>
      <div
        class="cv-interactive-tooltip__after-content"
        ref="afterContent"
        tabindex="0"
        style="position: absolute; left: -9999px; width: 1px; height: 1px"
        @focus="focusAfterContent"
      />
    </div>
  </div>
</template>

<script>
import { uidMixin, carbonPrefixMixin } from '../../mixins';
import Information16 from '@carbon/icons-vue/es/information/16';
import { alignments, directions } from './consts';

export default {
  name: 'CvInteractiveTooltip',
  mixins: [uidMixin, carbonPrefixMixin],
  components: { Information16 },
  props: {
    alignment: { type: String, default: 'center', validator: val => alignments.includes(val) },
    direction: {
      type: String,
      default: 'top',
      validator: val => {
        const valid = directions.includes(val);
        if (!valid) {
          console.warn(`CVInteractiveTooltip.direction must be one of the following: ${directions}`);
        }
        return valid;
      },
    },
    visible: { type: Boolean, default: false },
  },
  data() {
    return {
      dataVisible: false,
      left: -9999, // offscreen
      top: 0,
    };
  },
  computed: {
    contentAfter() {
      return this.direction === 'right' || this.direction === 'bottom';
    },
  },
  watch: {
    visible() {
      if (this.visible) {
        this.show();
      } else {
        this.hide();
      }
    },
    direction() {
      if (this.visible) {
        this.position();
      }
    },
    dataVisible() {
      if (this.dataVisible) {
        this.$emit('tooltip-shown');
      } else {
        this.$emit('tooltip-hidden');
      }
    },
  },
  methods: {
    positionListen(on) {
      if (on) {
        window.addEventListener('scroll', this.position);
        window.addEventListener('resize', this.position);
      } else {
        window.removeEventListener('scroll', this.position);
        window.removeEventListener('resize', this.position);
      }
    },
    position() {
      const menuPosition = this.$refs.trigger.getBoundingClientRect();
      const pixelsScrolledX = window.scrollX || window.pageXOffset;
      const pixelsScrolledY = window.scrollY || window.pageYOffset;

      if (this.direction === 'top' || this.direction === 'bottom') {
        this.left =
          menuPosition.left +
          0.5 +
          (this.$refs.trigger.offsetWidth - this.$refs.popup.offsetWidth) / 2 +
          pixelsScrolledX;

        if (this.direction === 'bottom') {
          this.top = menuPosition.bottom + 10 + pixelsScrolledY;
        } else {
          this.top = menuPosition.top - 15 - this.$refs.popup.offsetHeight + pixelsScrolledY;
        }
      } else {
        this.top =
          menuPosition.top +
          (this.$refs.trigger.offsetHeight - 0.5 - this.$refs.popup.offsetHeight) / 2 +
          pixelsScrolledY;
        if (this.direction === 'left') {
          this.left = menuPosition.left - 10 - this.$refs.popup.offsetWidth + pixelsScrolledX;
        } else {
          this.left = menuPosition.right + 15 + pixelsScrolledX;
        }
      }
    },
    show() {
      this.dataVisible = true;
      this.positionListen(true);

      this.$nextTick(() => {
        this.position();
        this.$refs.trigger.focus();
      });
    },
    hide() {
      this.dataVisible = false;
      this.positionListen(true);
    },
    toggle() {
      if (this.dataVisible) {
        this.hide();
      } else {
        this.show();
      }
    },
    onTriggerTab(ev) {
      if (!ev.shiftKey) {
        if (this.contentAfter) {
          // move focus before content before tab press
          this.$refs.beforeContent.focus();
        }
      } else {
        if (!this.contentAfter) {
          // move focus after content before tab press
          this.$refs.afterContent.focus();
        }
      }
    },
    checkFocusOut(ev) {
      this.dataVisible = ev.relatedTarget === this.$refs.trigger || this.$refs.popup.contains(ev.relatedTarget);
    },
    focusBeforeContent(ev) {
      if (this.contentAfter) {
        if (this.$refs.popup.contains(ev.relatedTarget)) {
          this.$refs.trigger.focus();
        }
      } else {
        this.$refs.trigger.focus();
        this.dataVisible = this.contentAfter;
      }
    },
    focusAfterContent(ev) {
      if (!this.contentAfter) {
        if (this.$refs.popup.contains(ev.relatedTarget)) {
          this.$refs.trigger.focus();
        }
      } else {
        this.$refs.trigger.focus();
        this.dataVisible = !this.contentAfter;
      }
    },
    preventFocusOut() {
      // This is here to prevent focus being lost if the user clicks on the contents of the interactive tool tip
    },
  },
  mounted() {
    // move popup out to body to ensure it appears above other elements
    this.popupEl = document.body.appendChild(this.$refs.popup);

    if (this.visible) {
      this.show();
    } else {
      this.hide();
    }
  },
  beforeDestroy() {
    this.positionListen(false);
    if (this.popupEl) {
      // move back to where it came from
      this.$el.appendChild(this.popupEl);
    }
  },
};
</script>
