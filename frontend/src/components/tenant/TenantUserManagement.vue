<script setup lang="ts">
import { ref, computed } from 'vue'

import ButtonPrimary from '@/components/ui/ButtonPrimary.vue'
import ButtonSecondary from '@/components/ui/ButtonSecondary.vue'
import FloatingActionButton from '@/components/ui/FloatingActionButton.vue'
import UserSearch from '@/components/user/UserSearch.vue'
import DialogBox from '@/components/ui/DialogBox.vue'
import type { Role, Tenant, User } from '@/models'
import { ROLES } from '@/utils/constants'

const props = defineProps<{
  loadingSearch: boolean
  possibleRoles?: Role[]
  searchResults: User[]
  tenant?: Tenant
}>()

const emit = defineEmits<{
  (event: 'add', user: User): void
  (event: 'cancel'): void
  (event: 'remove-role', payload: { userId: string; roleId: string }): void
  (event: 'search', query: Record<string, string>): void
}>()

const showSearch = ref(false)
const selectedUser = ref<User | null>(null)
const selectedRoles = ref<Role[]>([])

const roles = computed(() => props.possibleRoles ?? [])

// Dialog state
const showInfoDialog = ref(false)
const showConfirmDialog = ref(false)
const pendingUser = ref<User | null>(null)
const pendingRole = ref<Role | null>(null)

// Info dialog state
const infoDialog = ref({
  title: 'Action Blocked',
  message: '',
  buttons: [{ text: 'OK', action: 'ok', type: 'primary' as const }],
})

// Confirmation dialog state
const confirmDialog = ref({
  title: 'Confirm Role Removal',
  message: 'Are you sure you want to remove this role from the user?',
  buttons: [
    { text: 'Cancel', action: 'cancel', type: 'secondary' as const },
    { text: 'Remove', action: 'remove', type: 'primary' as const },
  ],
})

function toggleSearch() {
  showSearch.value = !showSearch.value
  if (!showSearch.value) {
    selectedUser.value = null
    selectedRoles.value = []
  }
}

function onUserSelected(user: User) {
  selectedUser.value = user
}

function onSearch(query: Record<string, string>) {
  emit('search', query)
}

function toggleRole(role: Role, checked: boolean) {
  if (checked) {
    if (!selectedRoles.value.find((r) => r.id === role.id)) {
      selectedRoles.value.push(role)
    }
  } else {
    selectedRoles.value = selectedRoles.value.filter((r) => r.id !== role.id)
  }
}

function handleAddUser() {
  if (!selectedUser.value || selectedRoles.value.length === 0) {
    return
  }

  selectedUser.value.roles = [...selectedRoles.value]
  emit('add', selectedUser.value)
  toggleSearch()
}

function handleCancel() {
  emit('cancel')
  toggleSearch()
}

// Helper function to show info dialog
function showInfo(message: string) {
  infoDialog.value.message = message
  showInfoDialog.value = true
}

function onRemoveRole(user: User, role: Role) {
  if (!props.tenant) {
    return
  }

  const isOwnerRole = role.name === ROLES.TENANT_OWNER.value
  const ownerCount = props.tenant.getOwners().length

  if (isOwnerRole && ownerCount <= 1) {
    showInfo(
      'There must be at least one user with the Tenant Owner role. To remove ' +
        'this role from the current user, assign the role to another user first.',
    )
    return
  }

  if (user.roles.length <= 1) {
    showInfo(
      'The user cannot be removed from their only role. Each user must have ' +
        'at least one role to remain in a tenant. To proceed, assign a new ' +
        'role to this user before removing the current one.',
    )
    return
  }

  // Set up pending removal and open confirm dialog
  pendingUser.value = user
  pendingRole.value = role
  showConfirmDialog.value = true
}

function confirmRemoveRole() {
  if (pendingUser.value && pendingRole.value) {
    emit('remove-role', {
      userId: pendingUser.value.id,
      roleId: pendingRole.value.id,
    })
  }

  // Clear state
  pendingUser.value = null
  pendingRole.value = null
}

// Dialog event handlers
function handleInfoClose() {
  // Dialog auto-closes, no additional cleanup needed
}

function handleConfirmRemove() {
  confirmRemoveRole()
}

function handleConfirmCancel() {
  // Clear pending state
  pendingUser.value = null
  pendingRole.value = null
}
</script>

<template>
  <v-container class="px-0" fluid>
    <v-row>
      <v-col cols="12">
        <h2 class="mb-6 mt-12">Tenant Users</h2>
        <v-data-table
          :header-props="{
            class: 'text-body-1 font-weight-bold bg-surface-light',
          }"
          :headers="[
            { title: 'Name', key: 'displayName', align: 'start' },
            { title: 'TMS Roles', key: 'roles', align: 'start' },
            { title: 'Email', key: 'email', align: 'start' },
          ]"
          :items="tenant?.users || []"
          :sort-by="[{ key: 'displayName', order: 'asc' }]"
          item-value="id"
          fixed-header
          hover
        >
          <template #no-data>
            <v-alert type="info">You have no users in this tenant.</v-alert>
          </template>
          <template #[`item.roles`]="{ item }">
            <div class="d-flex flex-wrap" style="gap: 8px; margin-block: 4px">
              <v-chip
                v-for="role in item.roles"
                :key="role.id"
                class="d-inline-flex align-center"
                color="primary"
              >
                {{ role.description }}
                <v-icon
                  class="ml-1 cursor-pointer"
                  icon="mdi-close"
                  size="small"
                  @click.stop="onRemoveRole(item, role)"
                />
              </v-chip>
            </div>
          </template>
        </v-data-table>
      </v-col>
    </v-row>

    <v-row v-if="!showSearch" class="mt-4">
      <v-col class="d-flex justify-start" cols="12">
        <FloatingActionButton
          icon="mdi-plus-box"
          text="Add User to Tenant"
          @click="toggleSearch"
        />
      </v-col>
    </v-row>

    <v-expand-transition>
      <div v-if="showSearch">
        <v-divider class="my-12" />

        <h2 class="mb-4">Add a user to this Tenant</h2>

        <p class="text-subtitle-1 mb-2 mt-8">
          1. Search for a user based on the selection criteria below:
        </p>

        <template v-if="tenant?.id">
          <UserSearch
            :loading="loadingSearch"
            :results="searchResults"
            :tenant-id="tenant.id"
            @search="onSearch"
            @select="onUserSelected"
          />

          <v-row v-if="selectedUser" class="mt-4">
            <v-col cols="12">
              <p class="text-subtitle-1 mb-2">
                2. Assign role(s) to this user:
              </p>

              <v-checkbox
                v-for="role in roles"
                :key="role.id"
                :label="role.description"
                :model-value="selectedRoles.some((r) => r.id === role.id)"
                class="my-0 py-0"
                hide-details
                @update:model-value="(checked) => toggleRole(role, !!checked)"
              />
            </v-col>
          </v-row>

          <v-row class="mt-8">
            <v-col class="d-flex justify-start" cols="12">
              <ButtonSecondary
                class="me-4"
                text="Cancel"
                @click="handleCancel"
              />

              <ButtonPrimary
                v-if="selectedUser"
                :disabled="selectedRoles.length === 0"
                text="Add User"
                @click="handleAddUser"
              />
            </v-col>
          </v-row>
        </template>

        <v-alert
          v-else
          text="Cannot search users: No tenant selected"
          type="warning"
        />
      </div>
    </v-expand-transition>

    <!-- Info dialog for single-button notifications -->
    <DialogBox
      v-model="showInfoDialog"
      :buttons="infoDialog.buttons"
      :message="infoDialog.message"
      :title="infoDialog.title"
      @ok="handleInfoClose"
    />

    <!-- Confirmation dialog for yes/no decisions -->
    <DialogBox
      v-model="showConfirmDialog"
      :buttons="confirmDialog.buttons"
      :message="confirmDialog.message"
      :title="confirmDialog.title"
      @cancel="handleConfirmCancel"
      @remove="handleConfirmRemove"
    />
  </v-container>
</template>
