<script>
import { CONFIG_MAP, SECRET, NAMESPACE } from '@shell/config/types';
import { get } from '@shell/utils/object';
import { _VIEW } from '@shell/config/query-params';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { LabeledInput } from '@shell/components/form/LabeledInput';

export default {
  components: {
    LabeledSelect,
    LabeledInput
  },

  props: {
    mode: {
      type:    String,
      default: 'create'
    },
    value: {
      type:    Object,
      default: () => {
        return { valueFrom: {} };
      }
    },
    allConfigMaps: {
      type:    Array,
      default: () => []
    },
    allSecrets: {
      type:    Array,
      default: () => []
    },
    // filter resource options by namespace(s) selected in top nav
    namespaced: {
      type:    Boolean,
      default: true
    },
    loading: {
      default: false,
      type:    Boolean
    },
    defaultType: {
      type:    String,
      default: ''
    },
  },

  data() {
    const typeOpts = [
      { value: 'simple', label: 'Key/Value Pair' },
      { value: 'resourceFieldRef', label: 'Resource' },
      { value: 'configMapKeyRef', label: 'ConfigMap Key' },
      { value: 'secretKeyRef', label: 'Secret key' },
      { value: 'fieldRef', label: 'Pod Field' },
      { value: 'secretRef', label: 'Secret' },
      { value: 'configMapRef', label: 'ConfigMap' },
    ];

    const resourceKeyOpts = ['limits.cpu', 'limits.ephemeral-storage', 'limits.memory', 'requests.cpu', 'requests.ephemeral-storage', 'requests.memory'];
    let type;
    let hasDefaultType = false;
    let rowClass = 'var-row';

    if (this.defaultType) {
      type = this.defaultType;
      hasDefaultType = true;
      rowClass = 'var-row default-row';
    } else if (this.value.secretRef) {
      type = 'secretRef';
    } else if (this.value.configMapRef) {
      type = 'configMapRef';
    } else if (this.value.value) {
      type = 'simple';
    } else if (this.value.valueFrom) {
      type = Object.keys((this.value.valueFrom))[0] || 'simple';
    }

    let refName;
    let name;
    let fieldPath;
    let referenced;
    let key;
    let valStr;
    const keys = [];

    switch (type) {
    case 'resourceFieldRef':
      name = this.value.name;
      refName = this.value.valueFrom[type].containerName;
      key = this.value.valueFrom[type].resource || '';
      break;
    case 'configMapKeyRef':
      name = this.value.name;
      key = this.value.valueFrom[type].key || '';
      refName = this.value.valueFrom[type].name;
      referenced = this.allConfigMaps.filter((resource) => {
        return resource.metadata.name === refName;
      })[0];
      if (referenced && referenced.data) {
        keys.push(...Object.keys(referenced.data));
      }
      break;
    case 'secretRef':
    case 'configMapRef':
      name = this.value.prefix;
      refName = this.value[type].name;
      break;
    case 'secretKeyRef':
      name = this.value.name;
      key = this.value.valueFrom[type].key || '';
      refName = this.value.valueFrom[type].name;
      referenced = this.allSecrets.filter((resource) => {
        return resource.metadata.name === refName;
      })[0];
      if (referenced && referenced.data) {
        keys.push(...Object.keys(referenced.data));
      }
      break;
    case 'fieldRef':
      fieldPath = get(this.value.valueFrom, `${ type }.fieldPath`) || '';
      name = this.value.name;
      break;
    default:
      name = this.value.name;
      valStr = this.value.value;
      break;
    }

    return {
      referenced: refName,
      secrets:    this.allSecrets,
      typeOpts,
      type,
      refName,
      keys,
      key,
      fieldPath,
      name,
      resourceKeyOpts,
      valStr,
      hasDefaultType,
      rowClass,
    };
  },
  computed: {
    isView() {
      return this.mode === _VIEW;
    },

    namespaces() {
      if (this.namespaced) {
        const map = this.$store.getters.namespaces();

        return Object.keys(map).filter((key) => map[key]);
      } else {
        const inStore = this.$store.getters['currentStore'](NAMESPACE);

        return this.$store.getters[`${ inStore }/all`](NAMESPACE);
      }
    },

    sourceOptions() {
      if (this.type === 'configMapKeyRef' || this.type === 'configMapRef') {
        return this.allConfigMaps.filter((map) => this.namespaces.includes(map?.metadata?.namespace));
      } else if (this.type === 'secretRef' || this.type === 'secretKeyRef') {
        return this.allSecrets.filter((secret) => this.namespaces.includes(secret?.metadata?.namespace));
      } else {
        return [];
      }
    },

    needsSource() {
      return this.type !== 'simple' && this.type !== 'resourceFieldRef' && this.type !== 'fieldRef' && !!this.type;
    },

    sourceLabel() {
      let out;
      const { type } = this;

      if (!type) {
        return;
      }

      switch (type) {
      case 'secretKeyRef':
      case 'secretRef':
        out = 'workload.container.command.fromResource.secret';
        break;
      case 'configMapKeyRef':
      case 'configMapRef':
        out = 'workload.container.command.fromResource.configMap';
        break;
      default:
        out = 'workload.container.command.fromResource.source.label';
      }

      return this.t(out);
    },

    nameLabel() {
      if (this.type === 'configMapRef' || this.type === 'secretRef') {
        return this.t('workload.container.command.fromResource.prefix');
      } else {
        return this.t('workload.container.command.fromResource.name.label');
      }
    },

    extraColumn() {
      return ['resourceFieldRef', 'configMapKeyRef', 'secretKeyRef'].includes(this.type);
    },
  },

  watch: {
    type() {
      this.referenced = null;
      this.key = '';
      this.refName = '';
      this.keys = [];
      this.key = '';
      this.valStr = '';
      this.fieldPath = '';
    },

    referenced(neu, old) {
      if (neu) {
        if ((neu.type === SECRET || neu.type === CONFIG_MAP) && neu.data) {
          this.keys = Object.keys(neu.data);
        }
        this.refName = neu?.metadata?.name;
      }
      this.updateRow();
    },
  },

  methods: {
    reset() {
      const out = { name: this.name || this.refName };
      const type = this.type || this.defaultType;

      switch (type) {
      case 'configMapKeyRef':
      case 'secretKeyRef':
        this.value.valueFrom[type].key = '';
        this.value.valueFrom[type].name = '';
        this.referenced = '';
        this.key = '';
        out.valueFrom = {
          [this.type]: {
            key: '', name: '', optional: false
          }
        };
        break;
      default:
        this.value = '';
        out.name = '';
        out.prefix = '';
        out[this.type] = { name: '', optional: false };
      }
      this.$emit('input', out);
    },
    updateRow() {
      if (!this.name?.length && !this.refName?.length) {
        if (this.type !== 'fieldRef') {
          this.$emit('input', null);

          return;
        }
      }
      let out = { name: this.name || this.refName };

      switch (this.type) {
      case 'configMapKeyRef':
      case 'secretKeyRef':
        out.valueFrom = {
          [this.type]: {
            key: this.key, name: this.refName, optional: false
          }
        };
        break;
      case 'resourceFieldRef':
        out.valueFrom = {
          [this.type]: {
            containerName: this.refName, divisor: 1, resource: this.key
          }
        };
        break;
      case 'fieldRef':
        if (!this.fieldPath || !this.fieldPath.length) {
          out = null; break;
        }
        out.valueFrom = { [this.type]: { apiVersion: 'v1', fieldPath: this.fieldPath } };
        break;
      case 'simple':
        out.value = this.valStr;
        break;
      default:
        delete out.name;
        out.prefix = this.name;
        out[this.type] = { name: this.refName, optional: false };
      }
      this.$emit('input', out);
    },
    get
  }
};
</script>

<template>
  <div :class="rowClass">
    <div
      v-if="!hasDefaultType"
      class="type"
    >
      <LabeledSelect
        v-model:value="type"
        :mode="mode"
        :multiple="false"
        :options="typeOpts"
        option-label="label"
        :searchable="false"
        :reduce="e=>e.value"
        :label="t('workload.container.command.fromResource.type')"
        @update:value="updateRow"
      />
    </div>

    <div
      v-if="!hasDefaultType"
      class="name"
    >
      <LabeledInput
        v-model:value="name"
        :label="nameLabel"
        :placeholder="t('workload.container.command.fromResource.name.placeholder')"
        :mode="mode"
        @update:value="updateRow"
      />
    </div>

    <div
      v-if="type==='simple'"
      class="single-value"
    >
      <LabeledInput
        v-model:value="valStr"
        :label="t('workload.container.command.fromResource.value.label')"
        :placeholder="t('workload.container.command.fromResource.value.placeholder')"
        :mode="mode"
        @update:value="updateRow"
      />
    </div>

    <template v-else-if="needsSource">
      <div :class="{'single-value': type === 'configMapRef' || type === 'secretRef'}">
        <LabeledSelect
          v-model:value="referenced"
          :options="sourceOptions"
          :multiple="false"
          :get-option-label="opt=>get(opt, 'metadata.name') || opt"
          :get-option-key="opt=>opt.id|| opt"
          :mode="mode"
          :label="sourceLabel"
          :loading="loading"
        />
      </div>
      <div v-if="type!=='secretRef' && type!== 'configMapRef'">
        <LabeledSelect
          v-model:value="key"
          :multiple="false"
          :options="keys"
          :mode="mode"
          option-label="label"
          :label="t('workload.container.command.fromResource.key.label')"
          :loading="loading"
          @update:value="updateRow"
        />
      </div>
    </template>

    <template v-else-if="type==='resourceFieldRef'">
      <div>
        <LabeledInput
          v-model:value="refName"
          :label="t('workload.container.command.fromResource.containerName')"
          :placeholder="t('workload.container.command.fromResource.source.placeholder')"
          :mode="mode"
          @update:value="updateRow"
        />
      </div>
      <div>
        <LabeledSelect
          v-model:value="key"
          :label="t('workload.container.command.fromResource.key.label')"
          :multiple="false"
          :options="resourceKeyOpts"
          :mode="mode"
          :searchable="false"
          :placeholder="t('workload.container.command.fromResource.key.placeholder', null, true)"
          @update:value="updateRow"
        />
      </div>
    </template>

    <template v-else>
      <div class="single-value">
        <LabeledInput
          v-model:value="fieldPath"
          :placeholder="t('workload.container.command.fromResource.key.placeholder', null, true)"
          :label="t('workload.container.command.fromResource.key.label')"
          :mode="mode"
          @update:value="updateRow"
        />
      </div>
    </template>
    <div class="remove">
      <div v-if="!hasDefaultType">
        <a-button
          v-if="!isView"
          type="link"
          @click.stop="$emit('remove')"
        >
          {{ t('generic.remove') }}
        </a-button>
      </div>
      <div v-else>
        <a-button
          v-if="!isView"
          type="link"
          @click="reset"
        >
          {{ t('generic.reset') }}
        </a-button>
      </div>
    </div>
  </div>
</template>

<style lang='scss' scoped>
.var-row{
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr 80px;
  grid-column-gap: 20px;
  margin-bottom: 10px;
  align-items: center;

  .single-value {
    grid-column: span 2;
  }

  .remove BUTTON {
    padding: 0px;
  }
}

.default-row{
  grid-template-columns: 1fr 1fr !important;
}

</style>
