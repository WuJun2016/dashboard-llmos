<script setup>
import { computed } from 'vue';
import { useStore } from 'vuex';
import { useI18n } from '@shell/composables/useI18n';
import { useFileItem } from '@shell/detail/ml.llmos.ai.model/FileList/useFileItem';
import { FileTextTwoTone, FolderTwoTone } from '@ant-design/icons-vue';
import dayjs from 'dayjs';
import { Modal, message } from 'ant-design-vue';
import { formatSi } from '@shell/utils/units';
import { diffFrom } from '@shell/utils/time';

const props = defineProps({
  file: {
    type:    Object,
    default: () => ({}),
  },

  resource: {
    type:     Object,
    required: true,
  },
});

const emit = defineEmits(['fetchFiles']);

const store = useStore();
const { t } = useI18n(store);

const isFile = computed(() => {
  return props.file.Size !== 0;
});

const fileSize = computed(() => {
  return isFile.value ? formatSi(props.file.Size, {
    increment: 1024,
    addSuffix: true,
    suffix:    'B',
  }) : '';
});

const lastModified = computed(() => {
  const now = dayjs();
  const out = diffFrom(dayjs(props.file.LastModified), now, (key, args) => t(key, args));

  return isFile.value ? `${ out.string } ago` : '';
});

const currentPath = computed(() => {
  const prefix = `datasets/${ props.resource.namespace }/${ props.resource.spec.dataset }/${ props.resource.spec.version }/`;

  return props.file.Path ? props.file.Path.replace(prefix, '') : '';
});

const {
  currentFolder,
  onRowClick,
} = useFileItem({
  props: {
    ...props,
    isFile,
    currentPath,
  },
  emit,
});

const removeFile = async(file) => {
  Modal.confirm({
    title: t('fileItem.deleteConfirm'),
    async onOk() {
      await props.resource.doAction('remove', { targetFilePath: currentPath.value });

      message.success(t('fileItem.deleteSuccess'));

      emit('fetchFiles', currentFolder.value);
    },
  });
};
</script>

<template>
  <div class="file-item">
    <div class="file-icon">
      <FileTextTwoTone v-if="isFile" />
      <FolderTwoTone v-else />
    </div>

    <a-row class="file-info">
      <a-col
        class="file-name"
        :span="12"
      >
        <span
          class="hand"
          @click="onRowClick"
        >
          {{ file.Name }}
        </span>
      </a-col>
      <a-col
        class="file-size"
        :span="4"
      >
        {{ fileSize }}
      </a-col>
      <a-col
        class="file-date"
        :span="4"
      >
        {{ lastModified }}
      </a-col>
      <a-col
        :span="4"
        class="file-action"
      >
        <span
          class="hand text-error"
          @click="removeFile(file)"
        >
          {{ t('fileItem.remove') }}
        </span>
      </a-col>
    </a-row>
  </div>
</template>

<style lang="scss" scoped>
.file-item {
  display: flex;
  align-items: center;
  padding: 12px 16px;
  border-bottom: 1px solid var(--border);

  &:hover {
    background-color: var(--hover-bg);
  }
}

.file-icon {
  margin-right: 8px;

  .anticon {
    font-size: 20px;
  }
}

.file-info {
  flex: 1;
  min-width: 0;
}

.file-name {
  font-weight: 500;
  color: var(--link);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.file-meta {
  font-size: 12px;
  color: var(--text-muted);
  margin-top: 4px;

  .file-size {
    margin-right: 16px;
  }
}

.file-date {
  text-align: end;
}

.file-action {
  text-align: end;
}
</style>
