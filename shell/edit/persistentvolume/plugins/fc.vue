<script>
import { LabeledInput } from '@shell/components/form/LabeledInput';
import { RadioGroup } from '@shell/components/form/Radio';
import ArrayList from '@shell/components/form/ArrayList';

export default {
  components: {
    ArrayList, LabeledInput, RadioGroup
  },
  props: {
    value: {
      type:    Object,
      default: () => ({})
    },
    mode: {
      type:     String,
      required: true,
    },
  },
  data() {
    const readOnlyOptions = [
      {
        label: this.t('generic.yes'),
        value: true
      },
      {
        label: this.t('generic.no'),
        value: false
      }
    ];

    this.value.spec['fc'] = this.value.spec.fc || {};
    this.value.spec.fc['readOnly'] = this.value.spec.fc.readOnly || false;
    this.value.spec.fc['secretRef'] = this.value.spec.fc.secretRef || {};

    return { readOnlyOptions };
  },
  computed: {
    lun: {
      get() {
        return this.value.spec.fc.lun;
      },
      set(value) {
        this.value.spec.fc['lun'] = Number.parseInt(value, 10);
      }
    }
  }
};
</script>

<template>
  <div>
    <div class="row mb-20">
      <div class="col span-6">
        <ArrayList
          v-model:value="value.spec.fc.targetWWNs"
          :add-label="t('persistentVolume.fc.targetWWNS.add')"
          :mode="mode"
        />
      </div>
      <div class="col span-6">
        <ArrayList
          v-model:value="value.spec.fc.wwids"
          :add-label="t('persistentVolume.fc.wwids.add')"
          :mode="mode"
        />
      </div>
    </div>
    <div class="row mb-20">
      <div class="col span-6">
        <LabeledInput
          v-model:value="lun"
          :mode="mode"
          :label="t('persistentVolume.fc.lun.label')"
          :placeholder="t('persistentVolume.fc.lun.placeholder')"
          type="number"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.spec.fc.fsType"
          :mode="mode"
          :label="t('persistentVolume.shared.filesystemType.label')"
          :placeholder="t('persistentVolume.shared.filesystemType.placeholder')"
        />
      </div>
    </div>
    <div class="row mb-20">
      <div class="col span-6">
        <RadioGroup
          v-model:value="value.spec.fc.readOnly"
          name="readOnly"
          :mode="mode"
          :label="t('persistentVolume.shared.readOnly.label')"
          :options="readOnlyOptions"
          :row="true"
        />
      </div>
    </div>
  </div>
</template>
