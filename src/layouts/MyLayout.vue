<template>
  <q-layout>
    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script>
export default {
  name: 'MyLayout',
};
</script>

<style lang="scss">
  .q-page-container {
    padding-top: 0!important;
  }
</style>
