<script setup lang="ts">
import AppHeader from '@/components/layout/AppHeader.vue'
import AppNavigation from '@/components/layout/AppNavigation.vue'
import AppNotifications from '@/components/layout/AppNotifications.vue'
</script>

<template>
  <v-app>
    <!-- Custom Components common to all application views -->
    <AppNotifications />
    <AppHeader />

    <!-- Router view for dynamic component rendering -->
    <v-main>
      <AppNavigation />
      <v-container class="mt-10 px-12" fluid>
        <router-view />
      </v-container>
    </v-main>
  </v-app>
</template>
