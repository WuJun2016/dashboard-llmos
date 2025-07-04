<script>
import Loading from '@shell/components/Loading';
import BadgeStateFormatter from '@shell/components/formatter/BadgeStateFormatter';
import LiveDate from '@shell/components/formatter/LiveDate.vue';
import ActionMenu from '@shell/components/ActionMenuShell.vue';

import ResourceFetch from '@shell/mixins/resource-fetch';

import { allHash } from '@shell/utils/promise';
import { LLMOS } from '@shell/config/types';

export default {
  name: 'ModelList',

  components: {
    Loading,
    BadgeStateFormatter,
    LiveDate,
    ActionMenu,
  },

  mixins: [
    ResourceFetch,
  ],

  props: {
    resource: {
      type:     String,
      required: true,
    },

    schema: {
      type:     Object,
      required: true,
    },

    useQueryParamsForSimpleFiltering: {
      type:    Boolean,
      default: false
    },
  },

  data() {
    return {
      currentPageNumber: 1,
      pageSize:          9,
      searchQuery:       '',
      searchContext:     '',
      selectedTag:       '',
      selectedType:      '',
      selectedProvider:  '',
    };
  },

  computed: {
    splitDataSource() {
      const rows = this.filteredRows || [];

      if (rows.length > (this.currentPageNumber - 1) * this.pageSize) {
        return rows.slice((this.currentPageNumber - 1) * this.pageSize, this.currentPageNumber * this.pageSize);
      } else {
        return rows;
      }
    },

    filteredRows() {
      const rows = this.filterTagRows || [];

      let filteredByNamespace = [];
      const isAll = this.$store.getters['isAllNamespaces'];

      if (
        !this.isNamespaced ||
        this.ignoreFilter ||
        this.externalPaginationEnabled ||
        (isAll && !this.currentProduct?.hideSystemResources) ||
        (this.inStore ? this.$store.getters[`${ this.inStore }/haveNamespace`](this.schema.id)?.length : false)
      ) {
        filteredByNamespace = rows;
      } else {
        const includedNamespaces = this.$store.getters['namespaces']();
        const haveAllNamespace = this.$store.getters['haveAllNamespace'];

        filteredByNamespace = rows.filter((row) => {
          if (this.currentProduct?.hideSystemResources && this.isNamespaced) {
            return !!includedNamespaces[row.metadata.namespace] && !row.isSystemResource;
          } else if (!this.isNamespaced) {
            return true;
          } else if (haveAllNamespace) {
            return true;
          } else {
            return !!includedNamespaces[row.metadata.namespace];
          }
        });
      }

      if (this.searchQuery) {
        return filteredByNamespace.filter((row) => {
          return row.metadata.name.toLowerCase().includes(this.searchQuery.toLowerCase());
        });
      }

      return filteredByNamespace;
    },

    filterTagRows() {
      const rows = this.rows || [];

      if (this.selectedTag) {
        return rows.filter((row) => {
          return (row.spec.modelCard.metadata.tags || []).includes(this.selectedTag);
        });
      }

      return rows;
    },

    contextOptions() {
      return [{
        label: '全部',
        value: '',
      }, {
        label: '128K',
        value: '128K',
      }, {
        label: '256K',
        value: '256K',
      }];
    },

    providerOptions() {
      return [{
        label: '全部',
        value: '',
      }, {
        label: '通义',
        value: '通义',
      }, {
        label: 'DeepSeek',
        value: 'DeepSeek',
      }, {
        label: '其他',
        value: '其他',
      }];
    },

    typeOptions() {
      return [{
        label: '全部',
        value: '',
      }, {
        label: '文本生成',
        value: '文本生成',
      }, {
        label: '全模态',
        value: '全模态',
      }, {
        label: '推理模型',
        value: '推理模型',
      }];
    },

    inStore() {
      return this.$store.getters['currentProduct'].inStore;
    },
  },

  async fetch() {
    await allHash({
      models:             this.$fetchType(this.resource),
      localModels:        this.$store.dispatch(`${ this.inStore }/findAll`, { type: LLMOS.LOCAL_MODEL }),
      localModelVersions: this.$store.dispatch(`${ this.inStore }/findAll`, { type: LLMOS.LOCAL_MODEL_VERSION }),
    });
  },

  methods: {
    onDeleteModel(row) {
      row.promptRemove();
    },
  },
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div class="model-list">
    <div>
      <a-flex
        class="mb-10"
        justify="flex-end"
      >
        <span />
        <a-input
          v-model:value="searchQuery"
          :aria-label="t('sortableTable.searchLabel')"
          :placeholder="t('sortableTable.search')"
          :style="{ width: '200px' }"
        />
      </a-flex>
    </div>

    <div class="grid">
      <template v-if="splitDataSource.length === 0">
        <a-empty />
      </template>
      <template v-else>
        <div
          v-for="(row, i) in splitDataSource"
          :key="i"
          class="item"
          :data-testid="`cluster-tools-app-${row.id}`"
        >
          <div class="logo">
            <img
              class="size-[20px] mr-2"
              :src="row.iconUrl"
            >
          </div>
          <div class="name-version">
            <div class="d-flex">
              <router-link :to="row.detailLocation">
                <h3 class="name">
                  {{ row.id }}
                </h3>
              </router-link>
              <div class="state">
                <BadgeStateFormatter :row="row" />
              </div>
            </div>
            <div class="text-muted">
              <LiveDate
                :value="row.creationTimestamp"
                :add-suffix="true"
                :show-tooltip="false"
              />
            </div>
          </div>
          <div class="action">
            <ActionMenu
              :resource="row"
            />
          </div>
          <div class="description mt-10">
            {{ row.spec.modelCard?.description }}
          </div>
        </div>
      </template>
    </div>
  </div>

  <a-pagination
    v-model:current="currentPageNumber"
    :defaultPageSize="pageSize"
    :total="splitDataSource.length"
    show-less-items
    hideOnSinglePage
  />
</template>

<style lang="scss" scoped>
$margin: 10px;
$logo: 50px;

.model-list {
  min-height: 80vh;
}

.ant-pagination {
  text-align: end;
}

.ant-radio-group {
  border: none;
}

.filter-label {
  font-weight: 500;
  font-size: 14px;
}

.tag-label {
  min-width: 60px;
  font-size: 14px;
  color: var(--input-label);
}

.grid {
  display: flex;
  justify-content: flex-start;
  flex-wrap: wrap;
  margin: 0 -1 * $margin;

  :deep(.ant-empty) {
    margin: auto;  // 让空状态组件居中显示
  }

  @media only screen and (min-width: map-get($breakpoints, '--viewport-4')) {
    .item {
      width: 100%;
    }
  }
  @media only screen and (min-width: map-get($breakpoints, '--viewport-7')) {
    .item {
      width: 100%;
    }
  }
  @media only screen and (min-width: map-get($breakpoints, '--viewport-9')) {
    .item {
      width: calc(50% - 2 * #{$margin});
    }
  }
  @media only screen and (min-width: map-get($breakpoints, '--viewport-12')) {
    .item {
      width: calc(33.33333% - 2 * #{$margin});
    }
  }

  .item {
    display: grid;
    grid-template-areas:
      'logo name-version action'
      'description description description';
    grid-template-columns: $logo auto min-content;
    grid-template-rows: 50px 115px;
    row-gap: 5px;
    column-gap: $margin;

    margin: $margin;
    padding: $margin;
    position: relative;
    border: 1px solid var(--border);
    border-radius: calc(1.5 * var(--border-radius));

    .logo {
      grid-area: logo;
      text-align: center;
      width: $logo;
      height: $logo;
      border-radius: calc(2 * var(--border-radius));
      overflow: hidden;
      background-color: white;

      img {
        width: $logo - 4px;
        height: $logo - 4px;
        object-fit: contain;
        position: relative;
        top: 2px;
      }

      > i {
        background-color: var(--box-bg);
        border-radius: 50%;
        font-size: 32px;
        line-height: 50px;
        width: 50px;
      }
    }

    .name-version {
      grid-area: name-version;
      padding: 5px 0 0 0;

      .d-flex {
        display: flex;
        align-items: center;
      }
    }

    .name {
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      margin: 0;
      padding: 0 10px 0 0;  // Increased right padding
    }

    .state {
      display: flex;
      align-items: center;
    }

    .description {
      padding-left: 10px;
      padding-right: 10px;
      grid-area: description;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 5;  // Reduced from 4 to 3 lines
      overflow: hidden;
      text-overflow: ellipsis;
      color: var(--text-muted);  // Changed to use design system color
      line-height: 1.5;  // Added line height for better readability
    }

    .description-content {
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 3;
      line-clamp: 3;
      overflow: hidden;
      text-overflow: ellipsis;
      color: var(--text-muted);
    }

    .action {
      grid-area: action;
      white-space: nowrap;

      button {
        height: 30px;
      }
    }
  }
}

.fixed-header-actions {
  padding: 0 0 5px 0;
  width: 100%;
  z-index: z-index('fixedTableHeader');
  background: transparent;
  display: grid;
  grid-template-columns: [bulk] auto [middle] min-content [search] minmax(
      min-content,
      200px
    );
  grid-column-gap: 10px;

  .search {
    grid-area: search;
    text-align: right;
    justify-content: flex-end;
  }
}

.custom-radio-group {
  :deep(.ant-radio-button-wrapper) {
    height: 32px;
    padding: 4px 16px;
    border: 1px solid var(--border);
    border-radius: 6px;
    margin-right: 8px;
    background: transparent;

    &::before {
      display: none;  // 移除按钮之间的分隔线
    }

    &:first-child {
      border-radius: 6px;
    }

    &:last-child {
      border-radius: 6px;
    }

    &:hover {
      color: var(--primary);
      border-color: var(--primary);
    }

    &.ant-radio-button-wrapper-checked {
      color: var(--primary);
      border-color: var(--primary);
      box-shadow: none;

      &::before {
        display: none;
      }
    }
  }
}

.tag-label {
  min-width: 60px;
  font-size: 14px;
  color: var(--input-label);
  line-height: 32px;
}
</style>
