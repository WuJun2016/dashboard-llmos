<script>
import { LabeledInput } from '@shell/components/form/LabeledInput';
import { RadioGroup } from '@shell/components/form/Radio';

export default {
  components: { LabeledInput, RadioGroup },
  props:      {
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

    this.value.spec['quobyte'] = this.value.spec.quobyte || {};
    this.value.spec.quobyte['readOnly'] = this.value.spec.quobyte.readOnly || false;

    return { readOnlyOptions };
  },
};
</script>

<template>
  <div>
    <div class="row mb-20">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.spec.quobyte.volume"
          :mode="mode"
          :label="t('persistentVolume.quobyte.volume.label')"
          :placeholder="t('persistentVolume.quobyte.volume.placeholder')"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.spec.quobyte.user"
          :mode="mode"
          :label="t('persistentVolume.quobyte.user.label')"
          :placeholder="t('persistentVolume.quobyte.user.placeholder')"
        />
      </div>
    </div>
    <div class="row mb-20">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.spec.quobyte.group"
          :mode="mode"
          :label="t('persistentVolume.quobyte.group.label')"
          :placeholder="t('persistentVolume.quobyte.group.placeholder')"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.spec.quobyte.registry"
          :mode="mode"
          :label="t('persistentVolume.quobyte.registry.label')"
          :placeholder="t('persistentVolume.quobyte.registry.placeholder')"
        />
      </div>
    </div>
    <div class="row mb-20">
      <div class="col span-6">
        <RadioGroup
          v-model:value="value.spec.quobyte.readOnly"
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
