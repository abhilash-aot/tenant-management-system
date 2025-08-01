<script setup lang="ts">
import { computed, ref, watch } from 'vue'

import ButtonPrimary from '@/components/ui/ButtonPrimary.vue'
import type { User } from '@/models'
import { type IdirSearchType, IDIR_SEARCH_TYPE } from '@/utils/constants'

// --- Component Interface -----------------------------------------------------

defineProps<{
  loading?: boolean
  searchResults: User[] | null
}>()

const emit = defineEmits<{
  (event: 'clear-search'): void
  (event: 'search', searchType: IdirSearchType, searchText: string): void
  (event: 'select', user: User): void
}>()

// --- Component State ---------------------------------------------------------

// Redefine the list of search types so that they're in the order wanted by the
// component.
const SEARCH_TYPES = [
  {
    title: IDIR_SEARCH_TYPE.FIRST_NAME.title,
    value: IDIR_SEARCH_TYPE.FIRST_NAME.value,
  },
  {
    title: IDIR_SEARCH_TYPE.LAST_NAME.title,
    value: IDIR_SEARCH_TYPE.LAST_NAME.value,
  },
  { title: IDIR_SEARCH_TYPE.EMAIL.title, value: IDIR_SEARCH_TYPE.EMAIL.value },
]

const searchText = ref('')
const searchType = ref<IdirSearchType>(IDIR_SEARCH_TYPE.FIRST_NAME.value)
const selectedUser = ref<User[]>([])

// --- Watchers and Effects ----------------------------------------------------

watch([searchText, searchType], () => {
  emit('clear-search')
})

// Emit when a user is selected
watch(selectedUser, (selection) => {
  if (selection?.length) {
    emit('select', selection[0])
  }
})

// --- Computed Values ---------------------------------------------------------

// Sort the results by the search type, so that it is updated whenever the user
// changes the search type.
const defaultSort = computed(() => [{ key: `ssoUser.${searchType.value}` }])

// The SSO API will return a 400 if the search text is less than 2 characters.
const isSearchEnabled = computed(() => {
  return searchText.value && searchText.value.length >= 2
})

// --- Component Methods -------------------------------------------------------

function handleSearch() {
  emit('search', searchType.value, searchText.value)
}
</script>

<template>
  <v-row>
    <v-col cols="2">
      <v-select
        v-model="searchType"
        :items="SEARCH_TYPES"
        label="Search by"
        hide-details
      />
    </v-col>
    <v-col cols="4">
      <v-text-field
        v-model="searchText"
        label="Search text"
        hide-details
        @keyup.enter="handleSearch"
      />
    </v-col>
    <v-col class="d-flex align-center" cols="2">
      <ButtonPrimary
        :disabled="!isSearchEnabled"
        text="Search"
        @click="handleSearch"
      />
    </v-col>
  </v-row>

  <v-row v-if="searchResults !== null || loading">
    <v-col cols="12">
      <h4 class="my-6">Search Results</h4>

      <v-data-table
        v-model="selectedUser"
        :header-props="{
          class: 'text-body-1 font-weight-bold bg-surface-light',
        }"
        :headers="[
          { title: 'First Name', key: 'ssoUser.firstName', align: 'start' },
          { title: 'Last Name', key: 'ssoUser.lastName', align: 'start' },
          { title: 'Email', key: 'ssoUser.email', align: 'start' },
        ]"
        :items="searchResults || []"
        :loading="loading"
        :sort-by="defaultSort"
        select-strategy="single"
        striped="even"
        return-object
        show-select
      >
        <template #no-data>
          <v-alert type="info">No matching users found</v-alert>
        </template>
      </v-data-table>
    </v-col>
  </v-row>
</template>
