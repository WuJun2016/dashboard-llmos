<script>
import { mapGetters } from 'vuex';
import { LabeledInput } from '@shell/components/form/LabeledInput';
import { Checkbox } from '@shell/components/form/Checkbox';

export default {
  components: { LabeledInput, Checkbox },

  props: {
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },
    mode: {
      type:    String,
      default: 'create'
    }
  },

  computed: { ...mapGetters({ t: 'i18n/t' }) },

};
</script>

<template>
  <div>
    <div class="row mb-10">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.name"
          :required="true"
          :mode="mode"
          :label="t('workload.storage.volumeName')"
        />
      </div>
      <div class="col span-6">
        <Checkbox
          v-model:value="value.gcePersistentDisk.readOnly"
          :mode="mode"
          :label="t('workload.storage.readOnly')"
        />
      </div>
    </div>
    <div class="row mb-10">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.gcePersistentDisk.pdName"
          :mode="mode"
          :label="t('workload.storage.csi.pdName')"
          :required="true"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model.number="value.gcePersistentDisk.partition"
          :mode="mode"
          :label="t('workload.storage.csi.partition')"
        />
      </div>
    </div>
    <div class="row">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.gcePersistentDisk.fsType"
          :mode="mode"
          :label="t('workload.storage.csi.fsType')"
        />
      </div>
    </div>
  </div>
</template>
