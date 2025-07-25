<script setup lang="ts">
import { nextTick, ref, watch } from 'vue'
import { VForm } from 'vuetify/components'

import ButtonPrimary from '@/components/ui/ButtonPrimary.vue'
import ButtonSecondary from '@/components/ui/ButtonSecondary.vue'
import type { GroupDetailFields } from '@/models'

const props = defineProps<{
  isDuplicateName: boolean
}>()

const emit = defineEmits<{
  (event: 'clear-duplicate-error'): void
  (event: 'submit', group: GroupDetailFields, addUser: boolean): void
}>()

// Auto-bound v-model from parent
const dialogVisible = defineModel<boolean>()

const closeDialog = () => (dialogVisible.value = false)

// Form state
const form = ref<InstanceType<typeof VForm>>()
const addUser = ref(false)
const formData = ref<GroupDetailFields>({
  description: '',
  name: '',
})
const isFormValid = ref(false)

// Clear the state when the dialog is opened. This is for the case that the
// user opens the dialog, enters data, cancels, and opens it again - the form
// should be empty.
watch(
  () => dialogVisible.value,
  async (newVal) => {
    if (newVal) {
      addUser.value = false
      formData.value = {
        description: '',
        name: '',
      }
      isFormValid.value = false

      // Trigger validation when dialog is shown, so that the user knows which
      // fields are required.
      await nextTick()
      form.value?.validate()
    }
  },
)

// When parent sets the duplicated name flag, force re-validation so that the
// message is displayed.
watch(
  () => props.isDuplicateName,
  async () => {
    await nextTick()
    await form.value?.validate()
  },
)

watch(
  () => [formData.value.name],
  () => {
    emit('clear-duplicate-error')
  },
)

// Validation
const rules = {
  maxLength: (max: number) => (value: string) =>
    !value || value.length <= max || `Must be ${max} characters or less`,
  notDuplicated: () => !props.isDuplicateName || 'Group name must be unique',
  required: (value: string) => !!value || 'Required',
}

const handleSubmit = () => {
  if (isFormValid.value) {
    emit('submit', formData.value, addUser.value)
    // Let parent decide when to close the dialog
  }
}
</script>

<template>
  <v-dialog v-model="dialogVisible" max-width="600px">
    <v-card class="pa-6">
      <v-card-title>Create a Group</v-card-title>
      <v-card-subtitle class="my-6">
        <a href="#">Learn more about Groups</a>
      </v-card-subtitle>
      <v-card-text>
        <v-form ref="form" v-model="isFormValid">
          <v-row>
            <v-col cols="12">
              <v-text-field
                v-model="formData.name"
                :maxlength="30"
                :rules="[
                  rules.required,
                  rules.maxLength(30),
                  rules.notDuplicated,
                ]"
                label="Group Name"
                required
              />
            </v-col>
          </v-row>
          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="formData.description"
                :rules="[rules.required, rules.maxLength(500)]"
                counter="500"
                label="Group Description"
                rows="1"
                auto-grow
                required
              ></v-textarea>
            </v-col>
          </v-row>
          <v-row>
            <v-col cols="12">
              <v-checkbox
                v-model="addUser"
                label="Add me as a user to this group"
              ></v-checkbox>
            </v-col>
          </v-row>
        </v-form>
      </v-card-text>
      <v-card-actions class="d-flex justify-end">
        <ButtonSecondary class="me-4" text="Cancel" @click="closeDialog" />
        <ButtonPrimary
          :disabled="!isFormValid"
          text="Submit"
          @click="handleSubmit"
        />
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>
