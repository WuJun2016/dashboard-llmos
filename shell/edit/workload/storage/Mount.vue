<script>
import { LabeledInput } from '@shell/components/form/LabeledInput';
import { Checkbox } from '@shell/components/form/Checkbox';
import { mapGetters } from 'vuex';
import { removeObject } from '@shell/utils/array';
export default {
  name:       'Mount',
  components: { LabeledInput, Checkbox },
  props:      {
    mode: {
      type:    String,
      default: 'create',
    },

    // volume name
    name: {
      type:    String,
      default: '',
    },

    container: {
      type:     Object,
      required: true,
    },
  },

  data() {
    const volumeMounts = (this.container.volumeMounts || []).filter(
      (mount) => mount.name === this.name
    );

    return { volumeMounts };
  },

  computed: { ...mapGetters({ t: 'i18n/t' }) },

  watch: {
    volumeMounts(neu) {
      this.container.volumeMounts = (this.container.volumeMounts || []).filter(
        (mount) => mount.name && mount.name !== this.name
      );
      this.container.volumeMounts.push(...neu);
    },

    name(neu) {
      this.updateMountNames(neu);
    },
  },

  created() {
    if (!this.volumeMounts.length) {
      this.volumeMounts.push({ name: this.name });
    }
  },

  methods: {
    add() {
      this.volumeMounts.push({ name: this.name });
    },

    remove(volumeMount) {
      removeObject(this.volumeMounts, volumeMount);
    },

    updateMountNames(name) {
      this.volumeMounts.forEach((mount) => {
        mount.name = name;
      });
    },
  },
};
</script>

<template>
  <div>
    <div
      v-if="volumeMounts.length"
      class="mount-headers"
    >
      <span>
        {{ t('workload.storage.mountPoint') }}
        <span class="text-error">*</span>
      </span>
      <span>
        {{ t('workload.storage.subPath') }}
      </span>
      <span class="read-only">
        {{ t('workload.storage.readOnly') }}
      </span>
      <span />
    </div>
    <div
      v-for="(volumeMount, i) in volumeMounts"
      :key="i"
      class="mount-rows"
    >
      <div>
        <LabeledInput
          :id="`mount-path-${i}`"
          v-model:value="volumeMount.mountPath"
          :mode="mode"
        />
      </div>
      <div>
        <LabeledInput
          v-model:value="volumeMount.subPath"
          :mode="mode"
        />
      </div>
      <div class="read-only">
        <Checkbox
          v-model:value="volumeMount.readOnly"
          :mode="mode"
        />
      </div>
      <div class="remove">
        <a-button
          v-if="mode !== 'view'"
          id="remove-mount"
          type="link"
          @click="remove(volumeMount)"
        >
          {{ t('generic.remove') }}
        </a-button>
      </div>
    </div>
    <div class="row">
      <a-button
        v-if="mode !== 'view'"
        id="add-mount"
        @click="add()"
      >
        {{ t('workload.storage.addMount') }}
      </a-button>
    </div>
  </div>
</template>

<style lang="scss">
.mount-headers,
.mount-rows {
  display: grid;
  grid-template-columns: 42% 42% 5% auto;
  grid-gap: $column-gutter;
  margin-bottom: 10px;
  align-items: center;

  .remove BUTTON {
    padding: 0px;
  }
}

.mount-headers {
  color: var(--input-label);
}
</style>
