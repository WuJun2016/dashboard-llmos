<script lang="ts">
import { defineComponent } from 'vue';
import { _VIEW } from '@shell/config/query-params';
import AsyncButton, { AsyncButtonCallback } from '@shell/components/AsyncButton.vue';
import Banner from '@shell/components/Banner/Banner.vue';

export default defineComponent({
  components: { AsyncButton, Banner },

  props: {
    /**
     * Current mode of the page
     * passed to asyncButton to determine lables of the button
     */
    mode: {
      type:     String,
      required: true,
    },

    errors: {
      type:    Array,
      default: null,
    },

    disableSave: {
      type:    Boolean,
      default: false,
    },
  },

  computed: {
    isView(): boolean {
      return this.mode === _VIEW;
    },
  },

  methods: {
    save(buttonCb: AsyncButtonCallback) {
      this.$emit('save', buttonCb);
    },

    done() {
      this.$emit('done');
    },
  },
});
</script>
<template>
  <div v-if="!isView">
    <div class="spacer-small" />

    <div
      v-for="(err, idx) in errors"
      :key="idx"
    >
      <Banner
        color="error"
        :label="err"
      />
    </div>
    <div class="buttons">
      <div class="left">
        <slot name="left" />
      </div>
      <div class="right">
        <slot name="cancel">
          <a-button @click="done">
            <t k="generic.cancel" />
          </a-button>
        </slot>
        <slot name="middle" />
        <slot name="save">
          <AsyncButton
            v-if="!isView"
            :mode="mode"
            type="primary"
            :disabled="disableSave"
            @click="save"
          />
        </slot>
        <slot name="right" />
      </div>
    </div>
  </div>
</template>

<style lang="scss">
.buttons {
  display: grid;
  grid-template-areas: 'left right';
  grid-template-columns: 'min-content auto';

  .left {
    grid-area: left;
    text-align: left;

    .btn,
    button {
      margin: 0 $column-gutter 0 0;
    }
  }

  .right {
    grid-area: right;
    text-align: right;

    .btn,
    button {
      margin: 0 0 0 $column-gutter;
    }
  }
}
</style>
