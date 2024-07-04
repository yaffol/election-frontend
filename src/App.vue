<template>
  <v-app>
    <v-main>
      <CandidateGrid />
    </v-main>
  </v-app>
</template>

<script>
import CandidateGrid from './components/CandidateGrid.vue';

export default {
  name: 'App',
  components: {
    CandidateGrid
  }
};
</script>
