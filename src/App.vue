<template>
  <v-app>
    <v-main>
      <mainPage/>
    </v-main>
  </v-app>
</template>

<script>
import mainPage from './components/mainPage.vue'

export default {
  name: 'App',

  components: {
    mainPage,
  },

  data: () => ({
    //
  }),
}
</script>
