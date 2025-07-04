<script>
import { Checkbox } from '@shell/components/form/Checkbox';
import CreateEditView from '@shell/mixins/create-edit-view';
import CruResource from '@shell/components/CruResource';
import NameNsDescription from '@shell/components/form/NameNsDescription';
import Tab from '@shell/components/Tabbed/Tab';
import { RadioGroup } from '@shell/components/form/Radio';
import LabeledInput from '@shell/components/form/LabeledInput/LabeledInput.vue';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import UnitInput from '@shell/components/form/UnitInput';
import uniq from 'lodash/uniq';
import { _CREATE, _EDIT, FOCUS, _VIEW } from '@shell/config/query-params';
import { STORAGE_CLASS, PV } from '@shell/config/types';
import StatusTable from '@shell/components/StatusTable';
import ResourceTabs from '@shell/components/form/ResourceTabs';
import Labels from '@shell/components/form/Labels';
import { Banner } from '@shell/components/Banner';
import ResourceManager from '@shell/mixins/resource-manager';
import { allHash } from '@shell/utils/promise';
import FormValidation from '@shell/mixins/form-validation';

const DEFAULT_STORAGE = '10Gi';

export default {
  name: 'LLMOSVolume',

  components: {
    Banner,
    Checkbox,
    CruResource,
    LabeledInput,
    LabeledSelect,
    Labels,
    NameNsDescription,
    RadioGroup,
    ResourceTabs,
    StatusTable,
    Tab,
    UnitInput,
  },

  mixins: [CreateEditView, FormValidation, ResourceManager],
  async fetch() {
    const inStore = this.$store.getters['currentProduct'].inStore;

    const hash = await allHash({ storageClass: this.$store.dispatch(`${ inStore }/findAll`, { type: STORAGE_CLASS }) });

    // don't block UI for these resources
    if (this.$store.getters['cluster/canList'](PV)) {
      this.resourceManagerFetchSecondaryResources(this.secondaryResourceData);
    }

    this.storageClassOptions = hash.storageClass.map((s) => s.name).sort();
    this.storageClassOptions.unshift(this.t('persistentVolumeClaim.useDefault'));

    this.value.spec['storageClassName'] = this.value.spec.storageClassName || this.storageClassOptions[0];
  },
  data() {
    const canListPersistentVolumes = this.$store.getters['cluster/canList'](PV);
    const canListStorageClasses = this.$store.getters['cluster/canList'](STORAGE_CLASS);
    const sourceOptions = [
      {
        label: this.t('persistentVolumeClaim.source.options.new'),
        value: 'new'
      },
      {
        label: this.t('persistentVolumeClaim.source.options.existing'),
        value: 'existing'
      }
    ];

    const defaultAccessModes = ['ReadWriteOnce'];

    this.value['spec'] = this.value.spec || {};
    this.value.spec['resources'] = this.value.spec.resources || {};
    this.value.spec.resources['requests'] = this.value.spec.resources.requests || {};
    this.value.spec.resources.requests['storage'] = this.value.spec.resources.requests.storage || DEFAULT_STORAGE;
    if (this.realMode === _CREATE) {
      this.value.spec['accessModes'] = defaultAccessModes;
    }

    const defaultTab = this.$route.query[FOCUS] || null;

    return {
      secondaryResourceData:   this.secondaryResourceDataConfig(),
      sourceOptions,
      source:                  this.value.spec.volumeName ? sourceOptions[1].value : sourceOptions[0].value,
      immutableMode:           this.realMode === _CREATE ? _CREATE : _VIEW,
      persistentVolumeOptions: [],
      persistentVolumes:       [],
      storageClassOptions:     [],
      defaultTab,
      canListPersistentVolumes,
      canListStorageClasses,
    };
  },
  computed: {
    readWriteOnce: {
      get() {
        return this.value.spec.accessModes.includes('ReadWriteOnce');
      },
      set(value) {
        this.checkboxSetter('ReadWriteOnce', value);
      }
    },
    readOnlyMany: {
      get() {
        return this.value.spec.accessModes.includes('ReadOnlyMany');
      },
      set(value) {
        this.checkboxSetter('ReadOnlyMany', value);
      }
    },
    readWriteMany: {
      get() {
        return this.value.spec.accessModes.includes('ReadWriteMany');
      },
      set(value) {
        this.checkboxSetter('ReadWriteMany', value);
      }
    },
    persistentVolume: {
      get() {
        return this.value.spec.volumeName;
      },
      set(value) {
        const persistentVolume = this.persistentVolumes.find((pv) => pv.metadata.name === value);

        this.value.spec['storageClassName'] = '';

        if (persistentVolume) {
          this.value.spec.resources.requests['storage'] = persistentVolume.spec.capacity?.storage;
          if (persistentVolume.spec?.storageClassName) {
            this.value.spec['storageClassName'] = persistentVolume.spec?.storageClassName ;
          }
        }

        this.value.spec['volumeName'] = value;
      }
    },
    storageAmountMode() {
      if (this.isCreate) {
        return _CREATE;
      } else if (this.isEdit && this.value.expandable && this.value.bound) {
        return _EDIT;
      }

      return _VIEW;
    }
  },
  created() {
    this.registerBeforeHook(this.willSave, 'willSave');
  },
  mounted() {
    const focus = this.$refs.volumeSize?.focus;

    if (this.defaultTab === 'volumeclaim' && focus) {
      setTimeout(() => focus());
    }
  },
  methods: {
    secondaryResourceDataConfig() {
      return {
        namespace: this.value?.metadata?.namespace || null,
        data:      {
          [PV]: {
            applyTo: [
              { var: 'persistentVolumes' },
              {
                var:         'persistentVolumeOptions',
                parsingFunc: (data) => {
                  return data
                    .map((s) => {
                      const status = s.status.phase === 'Available' ? '' : ` (${ s.status.phase })`;

                      return {
                        label: `${ s.metadata.name }${ status }`,
                        value: s.metadata.name
                      };
                    })
                    .sort((l, r) => l.label.localeCompare(r.label));
                }
              }
            ]
          },
        }
      };
    },
    checkboxSetter(key, value) {
      if (value) {
        this.value.spec.accessModes.push(key);
        this.value['accessModes'] = uniq(this.value.spec.accessModes);
      } else {
        const indexOf = this.value.spec.accessModes.indexOf(key);

        if (indexOf >= 0) {
          this.value.spec.accessModes.splice(indexOf, 1);
        }
      }
    },
    isPersistentVolumeSelectable(option) {
      const persistentVolume = this.persistentVolumes.find((pv) => pv.metadata.name === option.value);

      return persistentVolume.status.phase === 'Available';
    },
    updateDefaults(source) {
      if (source === 'new') {
        this.value.spec.resources.requests['storage'] = DEFAULT_STORAGE;
      }

      this['persistentVolume'] = null;
    },
    willSave() {
      if (this.value.spec.storageClassName === this.t('persistentVolumeClaim.useDefault')) {
        delete this.value.spec['storageClassName'];
      }
    }
  }
};
</script>

<template>
  <CruResource
    :done-route="doneRoute"
    :mode="mode"
    :resource="value"
    :subtypes="[]"
    :validation-passed="true"
    :bannerErrors="errors"
    @error="e=>errors = e"
    @finish="save"
    @cancel="done"
  >
    <NameNsDescription
      :value="value"
      :mode="mode"
      :register-before-hook="registerBeforeHook"
      :namespaced="true"
    />

    <ResourceTabs
      v-model:value="value"
      :mode="mode"
      :side-tabs="true"
      :default-tab="defaultTab"
    >
      <Tab
        name="volumeclaim"
        :label="t('persistentVolumeClaim.volumeClaim.label')"
        :weight="4"
      >
        <div class="row">
          <div class="col span-6">
            <RadioGroup
              v-model:value="source"
              name="source"
              :mode="immutableMode"
              :label="t('persistentVolumeClaim.source.label')"
              :options="sourceOptions"
              @update:value="updateDefaults"
            />
          </div>
          <div class="col span-6">
            <div class="row">
              <div
                v-if="source === 'new'"
                class="col span-12"
              >
                <LabeledSelect
                  v-if="canListStorageClasses"
                  v-model:value="value.spec.storageClassName"
                  :options="storageClassOptions"
                  :label="t('persistentVolumeClaim.volumeClaim.storageClass')"
                  :mode="immutableMode"
                />
                <LabeledInput
                  v-else
                  v-model:value="value.spec.storageClassName"
                  :label="t('persistentVolumeClaim.volumeClaim.storageClass')"
                  :mode="immutableMode"
                  :tooltip="t('persistentVolumeClaim.volumeClaim.tooltips.noStorageClass')"
                />
              </div>
              <div
                v-else
                class="col span-12"
              >
                <LabeledSelect
                  v-if="canListPersistentVolumes"
                  v-model:value="persistentVolume"
                  :options="persistentVolumeOptions"
                  :label="t('persistentVolumeClaim.volumeClaim.persistentVolume')"
                  :selectable="isPersistentVolumeSelectable"
                  :mode="immutableMode"
                  :loading="isLoadingSecondaryResources"
                />
                <LabeledInput
                  v-else
                  v-model:value="persistentVolume"
                  :label="t('persistentVolumeClaim.volumeClaim.persistentVolume')"
                  :mode="immutableMode"
                  :tooltip="t('persistentVolumeClaim.volumeClaim.tooltips.noPersistentVolume')"
                />
              </div>
            </div>
            <div class="row">
              <div class="col span-12 mt-10">
                <UnitInput
                  ref="volumeSize"
                  v-model:value="value.spec.resources.requests.storage"
                  :label="t('persistentVolumeClaim.volumeClaim.requestStorage')"
                  :mode="storageAmountMode"
                  :input-exponent="3"
                  :output-modifier="true"
                  :increment="1024"
                  :min="1"
                  :required="true"
                />
                <Banner
                  v-if="isEdit && !value.expandable"
                  color="info"
                  class="mt-10"
                >
                  {{ t('persistentVolumeClaim.expand.notSupported') }}
                </Banner>
                <Banner
                  v-else-if="isEdit && !value.bound"
                  color="info"
                  class="mt-10"
                >
                  {{ t('persistentVolumeClaim.expand.notBound') }}
                </Banner>
              </div>
            </div>
          </div>
        </div>
      </Tab>
      <Tab
        name="customize"
        :label="t('persistentVolumeClaim.customize.label')"
        :weight="3"
      >
        <div class="access">
          <h3>{{ t('persistentVolumeClaim.accessModes') }}</h3>
          <span class="text-error">*</span>
        </div>
        <div>
          <Checkbox
            v-model:value="readWriteOnce"
            :label="t('persistentVolumeClaim.customize.accessModes.readWriteOnce')"
            :mode="immutableMode"
          />
        </div>
        <div>
          <Checkbox
            v-model:value="readOnlyMany"
            :label="t('persistentVolumeClaim.customize.accessModes.readOnlyMany')"
            :mode="immutableMode"
          />
        </div>
        <div>
          <Checkbox
            v-model:value="readWriteMany"
            :label="t('persistentVolumeClaim.customize.accessModes.readWriteMany')"
            :mode="immutableMode"
          />
        </div>
      </Tab>
      <Tab
        v-if="isView"
        name="status"
        :label="t('persistentVolumeClaim.status.label')"
        :weight="2"
      >
        <StatusTable :resource="value" />
      </Tab>
      <Tab
        name="labels-and-annotations"
        label-key="generic.labelsAndAnnotations"
        :weight="-1"
      >
        <Labels
          default-container-class="labels-and-annotations-container"
          :value="value"
          :mode="mode"
          :display-side-by-side="false"
        />
      </Tab>
    </ResourceTabs>
  </CruResource>
</template>

<style lang='scss' scoped>
.access {
  display: flex;
  flex-direction: row;
}
</style>
