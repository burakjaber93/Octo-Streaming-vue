<template>
  <div>
    <VAppBar
      density="compact"
      flat>
      <span class="text-h6 hidden-sm-and-down">
        {{ library.Name }}
      </span>
      <VChip
        size="small"
        class="hidden-sm-and-down ma-2">
        <template v-if="loading && items.length === lazyLoadLimit && initialId === route.params.itemId">
          {{ t('lazyLoading', { value: items.length }) }}
        </template>
        <JProgressCircular
          v-else-if="loading"
          indeterminate
          class="uno-h-full" />
        <template v-else>
          {{ items.length ?? 0 }}
        </template>
      </VChip>
      <VDivider
        vertical
        inset
        class="hidden-sm-and-down mx-2" />
      <TypeButton
        v-if="hasViewTypes"
        v-model="viewType"
        :type="library.CollectionType" />
      <VDivider
        v-if="isSortable && hasViewTypes"
        inset
        vertical
        class="mx-2" />
      <SortButton
        v-if="isSortable"
        :ascending="sortAscending"
        @change="onChangeSort" />
      <FilterButton
        v-if="library && viewType && isSortable"
        :item="library"
        @change="onChangeFilter" />
      <VSpacer />
      <PlayButton
        v-if="library"
        :item="library"
        shuffle />
      <PlayButton
        v-if="library"
        :item="library" />
    </VAppBar>
    <VContainer>
      <ItemGrid
        :items="items">
        <h1
          v-if="!loading && items.length === 0"
          class="text-h5">
          {{ hasFilters ? t('libraryEmptyFilters') : t('libraryEmpty') }}
        </h1>
      </ItemGrid>
    </VContainer>
    <ScrollToTopButton />
  </div>
</template>

<script setup lang="ts">
import {
  BaseItemKind, SortOrder
} from '@jellyfin/sdk/lib/generated-client';
import { getArtistsApi } from '@jellyfin/sdk/lib/utils/api/artists-api';
import { getGenresApi } from '@jellyfin/sdk/lib/utils/api/genres-api';
import { getItemsApi } from '@jellyfin/sdk/lib/utils/api/items-api';
import { getMusicGenresApi } from '@jellyfin/sdk/lib/utils/api/music-genres-api';
import { getPersonsApi } from '@jellyfin/sdk/lib/utils/api/persons-api';
import { getStudiosApi } from '@jellyfin/sdk/lib/utils/api/studios-api';
import { computed, onBeforeMount, ref, shallowRef } from 'vue';
import { useTranslation } from 'i18next-vue';
import { useRoute } from 'vue-router';
import { useBaseItem } from '#/composables/apis';
import type { Filters } from '#/components/Buttons/FilterButton.vue';
import { useItemPageTitle } from '#/composables/page-title';

const { t } = useTranslation();
const route = useRoute('/library/[itemId]');

const lazyLoadLimit = 50;
const initialId = route.params.itemId;

const COLLECTION_TYPES_MAPPINGS: Record<string, BaseItemKind> = {
  tvshows: BaseItemKind.Series,
  movies: BaseItemKind.Movie,
  books: BaseItemKind.Book,
  music: BaseItemKind.MusicAlbum,
  boxsets: BaseItemKind.BoxSet,
  livetv: BaseItemKind.LiveTvChannel
};

const innerItemKind = shallowRef<BaseItemKind>();
const sortBy = shallowRef<string>();
const sortAscending = shallowRef(true);
const queryLimit = shallowRef<number | undefined>(lazyLoadLimit);
const filters = ref<Filters>({
  status: [],
  features: [],
  genres: [],
  ratings: [],
  types: [],
  years: []
});

function onChangeSort(sort: string, ascending: boolean): void {
  sortBy.value = sort;
  sortAscending.value = ascending;
}

function onChangeFilter(changedFilters: Filters): void {
  filters.value = changedFilters;
}

const { data: libraryQuery } = await useBaseItem(getItemsApi, 'getItems')(() => ({
  ids: [route.params.itemId]
}));
const library = computed(() => libraryQuery.value[0]!);

const viewType = computed({
  get() {
    if (innerItemKind.value) {
      return innerItemKind.value;
    } else {
      return library.value.CollectionType ? COLLECTION_TYPES_MAPPINGS[library.value.CollectionType] : undefined;
    }
  },
  set(newVal) {
    innerItemKind.value = newVal;
  }
});

const hasFilters = computed(() =>
  Object.values(filters.value).some(({ length }) => length > 0)
);

const hasViewTypes = computed(
  () =>
    ['movies', 'music', 'tvshows', 'livetv'].includes(library.value.CollectionType ?? '')
);

const isSortable = computed(
  () =>
    viewType.value && 
    !['MusicArtist', 'Person', 'Genre', 'MusicGenre', 'Studio'].includes(viewType.value)
);

const recursive = computed(() =>
  // Force recursive for Live TV to find items inside folders created by Xtream
  library.value.CollectionType === 'livetv' ? true :
  (library.value.CollectionType === 'homevideos' || library.value.Type === 'Folder' ? undefined : true)
);

const parentId = computed(() => library.value.Id);
const methods = computed(() => {
  switch (viewType.value) {
    case 'MusicArtist': return [getArtistsApi, 'getArtists'] as const;
    case 'Person': return [getPersonsApi, 'getPersons'] as const;
    case 'Genre': return [getGenresApi, 'getGenres'] as const;
    case 'MusicGenre': return [getMusicGenresApi, 'getMusicGenres'] as const;
    case 'Studio': return [getStudiosApi, 'getStudios'] as const;
    default: return [getItemsApi, 'getItems'] as const;
  }
});

const api = computed(() => methods.value[0]);
const method = computed(() => methods.value[1]);

const { loading, data: items } = await useBaseItem(api, method)(() => ({
  parentId: parentId.value,
  personTypes: viewType.value === 'Person' ? ['Actor'] : undefined,
  // DANGER/FIX: If this is Live TV, do NOT filter by ItemType. Show everything.
  includeItemTypes: library.value.CollectionType === 'livetv' ? undefined : (viewType.value ? [viewType.value] : undefined),
  sortOrder: [sortAscending.value ? SortOrder.Ascending : SortOrder.Descending],
  sortBy: [sortBy.value ?? 'SortName'],
  recursive: recursive.value,
  limit: queryLimit.value
}));

useItemPageTitle(library);

onBeforeMount(() => {
  queryLimit.value = undefined;
});
</script>