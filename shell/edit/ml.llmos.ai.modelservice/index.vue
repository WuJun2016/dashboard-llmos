<script>
import CreateEditView from '@shell/mixins/create-edit-view';
import FormValidation from '@shell/mixins/form-validation';
import LLMOSWorkload from '@shell/mixins/llmos/ml-workload';
import { MANAGEMENT, PVC } from '@shell/config/types';
import { mergeEnvs } from '@shell/utils/merge';
import { REGISTRY as REGISTRY_ANNOTATIONS } from '@shell/config/labels-annotations';
import { SETTING } from '@shell/config/settings';
import { allHash } from '@shell/utils/promise';
import RemoteModelList from '@shell/edit/ml.llmos.ai.modelservice/RemoteModelList.vue';
import LocalModelList from '@shell/edit/ml.llmos.ai.modelservice/LocalModelList';
import { _CREATE } from '@shell/config/query-params';
import LabeledSelect from '@shell/components/form/LabeledSelect.vue';
import ArgumentVars from './components/ArgumentVars.vue';

export default {
  name:       'EditModelService',
  mixins:     [CreateEditView, FormValidation, LLMOSWorkload, LabeledSelect],
  components: {
    RemoteModelList, ArgumentVars, LocalModelList
  },
  props: {
    value: {
      type:     Object,
      required: true,
    },
    mode: {
      type:    String,
      default: 'create',
    },

    createOption: {
      default: (text) => {
        if (text) {
          return { metadata: { name: text } };
        }
      },
      type: Function,
    },
  },

  async fetch() {
    const inStore = this.$store.getters['currentProduct'].inStore;
    const hash = await allHash({
      modelServiceDefaultImage: this.$store.getters[`${ inStore }/byId`](
        MANAGEMENT.SETTING,
        SETTING.MODEL_SERVICE_DEFAULT_IMAGE
      ),
      settings:     this.$store.dispatch(`${ inStore }/findAll`, { type: MANAGEMENT.SETTING }),
      volumeClaims: this.$store.dispatch(`${ inStore }/findAll`, { type: PVC }),
    });

    await this.resourceManagerFetchSecondaryResources(this.secondaryResourceData);

    const huggingFaceProxy = hash.settings?.find(
      (item) => item.id === SETTING.HUGGINGfACE_ENDPOINT
    );

    const container = this.value.spec?.template?.spec?.containers[0];

    const hfToken = container.env.find(
      (env) => env.name === 'HUGGING_FACE_HUB_TOKEN'
    );

    if (!hfToken) {
      this.hfToken = {
        name:      'HUGGING_FACE_HUB_TOKEN',
        valueFrom: { secretKeyRef: { name: '', key: '' } },
      };
    }

    const hfEndpoint = container.env.find((env) => env.name === 'HF_ENDPOINT');

    if (!hfEndpoint && huggingFaceProxy) {
      this.hfEndpoint = { name: 'HF_ENDPOINT', value: huggingFaceProxy.value };
    }

    if (!this.container.image) {
      const systemImageRegistry = hash.settings?.find(
        (item) => item.id === SETTING.GLOBAL_SYSTEM_IMAGE_REGISTRY
      );
      const msDefaultImage =
        hash.modelServiceDefaultImage.value ||
        hash.modelServiceDefaultImage.default;
      const registry = systemImageRegistry?.value || 'ghcr.io';

      const image = this.processImageString(msDefaultImage, registry);

      this.container.image = image;
    }
  },

  data() {
    const excludeEnvs = [
      'HUGGING_FACE_HUB_TOKEN',
      'HF_ENDPOINT',
      'VLLM_USE_MODELSCOPE',
    ];

    const items = [{ title: 'Choose Model' }, { title: 'Set Config' }];

    return {
      current: 0,
      items,
      open:    false,
      hfToken: {
        name:      'HUGGING_FACE_HUB_TOKEN',
        valueFrom: { secretKeyRef: { name: '', key: '' } },
      },
      hfEndpoint:    { name: 'HF_ENDPOINT', value: '' },
      excludeEnvs,
      events:        [],
      sourceOptions: [
        { label: 'HuggingFace', value: 'huggingface' },
        { label: 'ModelScope', value: 'modelscope' },
        { label: 'Local', value: 'local' },
      ],
    };
  },

  created() {
    this.registerBeforeHook(this.willSave, 'willSave');
  },

  computed: {
    showRemoteList() {
      return (
        this.spec.modelRegistry === 'huggingface' ||
        this.spec.modelRegistry === 'modelscope'
      );
    },
    disableNext() {
      return !this.spec.modelRegistry;
    },

    showConfig() {
      return this.current === 1 || this.isEdit || this.isView;
    },

    validationPassed() {
      return (
        !!this.spec.modelRegistry &&
        !!this.value.metadata.name &&
        !!this.spec.model
      );
    },

    // page show logic
    disableModelInput() {
      if (
        (this.spec.modelRegistry &&
        this.spec.modelRegistry !== 'local' &&
        this.mode === _CREATE) || this.isEdit
      ) {
        return true;
      } else {
        return false;
      }
    },
    isHuggingFace() {
      return this.spec.modelRegistry === 'huggingface';
    },
  },

  methods: {
    willSave() {
      this.errors = [];

      if (Array.isArray(this.container.args)) {
        this.container.args = this.container.args.filter((arg) => !!arg && arg.trim() !== '');
      }

      this.update();

      if (this.container.image === '') {
        this.errors.push(this.t('validation.required', { key: 'Image' }, true));
      }

      if (
        this.container.resources.requests.cpu === 0 ||
        this.container.resources.requests.memory === 0
      ) {
        this.errors.push(
          this.t('validation.required', { key: 'CPU or Memory' }, true)
        );
      }

      if (this.spec.model === '') {
        this.errors.push(
          this.t('validation.required', { key: 'Model Name' }, true)
        );
      }

      if (this.errors.length > 0) {
        return Promise.reject(this.errors);
      }
    },

    update() {
      this.mergeEnvs();
    },

    mergeEnvs() {
      const huggingFaceEnv = [this.hfToken, this.hfEndpoint];

      this.container.env = mergeEnvs(this.container.env, huggingFaceEnv);
    },

    processImageString(imageString, customRegistry) {
      // Check if the image contains a registry
      const regex =
        /^([a-zA-Z0-9._-]+\/)?([a-zA-Z0-9_-]+)\/([a-zA-Z0-9_-]+):([a-zA-Z0-9._-]+)$/;

      const match = imageString.match(regex);

      if (match) {
        const registry = match[1]; // First group is the optional registry part

        // If a registry is present, return the string as is
        if (registry) {
          return imageString;
        } else {
          // Prepend custom registry if missing
          return `${ customRegistry }/${ imageString }`;
        }
      }
      throw new Error('Invalid image string format.');
    },
    next() {
      this.current++;
    },
    prev() {
      this.current--;
    },

    updateModelInfo(item) {
      this.spec.tags = item.tags || [];
      this.spec.pipelineTag = item.pipelineTag || '';
      this.spec.libraryName = item.libraryName || '';
      this.spec.model = item.id;
    },

    async updateLocalModelInfo(localModelVersion = {}) {
      const volumeClaims = localModelVersion.volumeClaims;

      const namespace = localModelVersion?.metadata?.namespace;
      const localModel = localModelVersion?.spec?.localModel;
      const volumeSnapshotName = localModelVersion?.status?.volumeSnapshot;

      this.spec.model = `${ namespace }/${ localModel }`;

      const volumeClaimTemplate = this.spec.volumeClaimTemplates[0];
      const containerTemplate = this.spec.template.spec.containers[0];

      Object.assign(containerTemplate, {
        volumeMounts: [{
          name:      volumeClaims?.spec?.volumeName,
          mountPath: '/root/.cache',
        }]
      });

      const pvc = await this.$store.dispatch('cluster/create', {
        metadata: {
          name:        volumeSnapshotName,
          namespace:   volumeClaims?.metadata?.namespace,
          annotations: { [REGISTRY_ANNOTATIONS.IS_LOCAL_MODEL]: 'true' },
        },
        spec: {
          storageClassName: volumeClaims?.spec?.storageClassName,
          resources:        { requests: { storage: volumeClaims?.spec?.resources?.requests?.storage } },
          accessModes:      ['ReadWriteOnce'],
        },
        type: PVC,
      });

      Object.assign(volumeClaimTemplate, pvc);
    },

    updateModelRegistry() {
      this.spec.model = '';
    }
  },

  watch: {
    'spec.modelRegistry'(newValue) {
      const query = { ...this.$route.query };

      if (newValue) {
        query.modelRegistry = newValue;
      } else {
        delete query.modelRegistry;
      }

      if (this.spec.model) {
        query.modelname = this.spec.model;
      }

      this.$router.replace({ query });
    },

    'spec.model'(newValue) {
      const query = { ...this.$route.query };

      if (newValue) {
        query.modelname = newValue;
      } else {
        delete query.modelname;
      }

      if (this.spec.modelRegistry) {
        query.modelRegistry = this.spec.modelRegistry;
      }

      this.$router.replace({ query });
    }
  },
};
</script>

<template>
  <CruResource
    :done-route="doneRoute"
    :resource="value"
    :mode="mode"
    :bannerErrors="errors"
    :apply-hooks="applyHooks"
    class="flex flex-col h-full"
    :validation-passed="validationPassed"
    @finish="save"
    @error="e=>errors = e"
  >
    <a-steps
      v-if="isCreate"
      :current="current"
      :items="items"
      class="mb-10"
      size="small"
    />

    <div v-if="showConfig">
      <NameNsDescription
        :value="value"
        :namespaced="true"
        :mode="mode"
      />
      <ResourceTabs
        :value="value"
        class="mt-15"
        :need-conditions="false"
        :need-related="false"
        :side-tabs="true"
        :mode="mode"
      >
        <Tab
          name="general"
          label="General"
          class="bordered-table"
          :weight="tabWeightMap.general"
        >
          <a-row
            :gutter="[16]"
            class="mb-[16px]"
          >
            <a-col :span="12">
              <LabeledSelect
                v-model:value="spec.modelRegistry"
                label="Source"
                :options="sourceOptions"
                :mode="mode"
                required
                :disabled="disableModelInput"
              />
            </a-col>

            <a-col :span="12">
              <LabeledInput
                v-model:value="spec.model"
                :required="true"
                :localized-label="true"
                :mode="mode"
                :disabled="disableModelInput"
                :label="t('modelService.model')"
                :tooltip="t('modelService.modelTooltip')"
                :placeholder="t('modelService.modelPlaceholder')"
              />
            </a-col>
          </a-row>

          <a-row
            :gutter="[16, 24]"
            class="mb-[16px]"
          >
            <a-col :span="12">
              <LabeledInput
                v-model:value="spec.servedModelName"
                :localized-label="true"
                :mode="mode"
                :label="t('modelService.modelName')"
                :tooltip="t('modelService.modelNamePlaceholder')"
              />
            </a-col>
          </a-row>

          <a-row
            :gutter="[16]"
            class="mb-[16px]"
          >
            <a-col :span="12">
              <h3>{{ t('workload.container.command.args') }}</h3>
              <ArgumentVars
                v-model="container.args"
                class="w-full"
                :disabled="isView"
              />
            </a-col>
          </a-row>

          <template v-if="isHuggingFace">
            <h3>{{ t("modelService.huggingFaceToken") }}</h3>
            <a-row
              :gutter="[16]"
              class="mb-[16px]"
            >
              <a-col :span="12">
                <ValueFromResource
                  v-model:value="hfToken"
                  :value="hfToken"
                  default-type="secretKeyRef"
                  :all-secrets="namespacedSecrets"
                  :mode="mode"
                  :loading="isLoadingSecondaryResources"
                  @update:value="update"
                />
              </a-col>

              <a-col :span="12">
                <LabeledInput
                  v-model:value="hfEndpoint.value"
                  :localized-label="true"
                  :mode="mode"
                  class="mb-20"
                  :label="t('modelService.hf.endpoint')"
                />
              </a-col>
            </a-row>
          </template>

          <div class="row">
            <div class="col span-12">
              <h3>{{ t("workload.container.titles.env") }}</h3>
              <EnvVars
                :mode="mode"
                :config-maps="namespacedConfigMaps"
                :secrets="namespacedSecrets"
                :value="container"
                :excludes="excludeEnvs"
                :loading="isLoadingSecondaryResources"
              />
            </div>
          </div>

          <AdvancedSection
            class="col span-12 advanced"
            :mode="mode"
          >
            <div class="row">
              <div class="col span-6">
                <LabeledInput
                  v-model:value="container.image"
                  :required="true"
                  :mode="mode"
                  :label="t('modelService.image')"
                />
              </div>

              <div class="col span-6">
                <LabeledSelect
                  v-model:value="spec.serviceType"
                  :mode="mode"
                  :options="svcOptions"
                  :label="t('workload.networking.networkMode.label')"
                  :placeholder="
                    t('workload.networking.networkMode.placeholder')
                  "
                  @update:value="update"
                />
              </div>
            </div>
          </AdvancedSection>
        </Tab>

        <Tab
          :label="t('workload.container.titles.resources')"
          name="resources"
          :weight="tabWeightMap['resources']"
        >
          <!-- Resources and Limitations -->
          <ContainerResourceLimit
            v-model:value="flatResources"
            :mode="mode"
            :runtime-classes="runtimeClasses"
            :pod-spec="podTemplateSpec"
            :handle-gpu-limit="true"
            :handle-v-gpu="true"
            :show-tip="false"
          />
        </Tab>

        <Tab
          :label="t('generic.volume.title')"
          name="volumes"
          :weight="tabWeightMap['volumes']"
        >
          <Volume
            v-model:value="spec"
            :namespace="value.metadata?.namespace"
            :register-before-hook="registerBeforeHook"
            :mode="mode"
            :save-pvc-hook-name="savePvcHookName"
            :loading="isLoadingSecondaryResources"
            :namespaced-pvcs="pvcs"
            @removePvcForm="clearPvcFormState"
          />
        </Tab>

        <Tab
          :label="t('workload.container.titles.nodeScheduling')"
          name="nodeScheduling"
          :weight="tabWeightMap['nodeScheduling']"
        >
          <NodeScheduling
            :mode="mode"
            :value="podTemplateSpec"
            :nodes="allNodes"
            :loading="isLoadingSecondaryResources"
          />
        </Tab>
      </ResourceTabs>
    </div>

    <div
      v-if="isCreate"
      class="grow-1 h-[0] flex flex-col"
    >
      <LabeledSelect
        v-if="current === 0"
        v-model:value="spec.modelRegistry"
        label="Source"
        :options="sourceOptions"
        :mode="mode"
        required
        class="mb-10"
        @update:value="updateModelRegistry"
      />

      <div class="grow-1">
        <RemoteModelList
          v-if="current === 0 && showRemoteList"
          :key="spec.modelRegistry"
          :default-search="spec.model"
          class="h-full overflow-hidden"
          :source="spec.modelRegistry"
          @update:item="updateModelInfo"
        />
        <LocalModelList
          v-else-if="current === 0 && spec.modelRegistry === 'local'"
          :default-search="spec.model"
          class="h-full overflow-hidden"
          :source="spec.modelRegistry"
          @update:item="updateLocalModelInfo"
        />
      </div>
    </div>

    <template #createBtnPrefix>
      <div
        v-if="isCreate"
        class="inline-block"
      >
        <a-button
          v-if="current === 0"
          type="primary"
          :disabled="disableNext"
          @click="next"
        >
          {{ t('generic.next') }}
        </a-button>

        <a-button
          v-if="current === 1"
          @click="prev"
        >
          {{ t('generic.previous') }}
        </a-button>
      </div>
    </template>
  </CruResource>
</template>
