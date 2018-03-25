<template>
  <div class="home">
    <ExploreVenue/>
  </div>
</template>

<script>
// @ is an alias to /src
import ExploreVenue from '@/components/ExploreVenue.vue'

export default {
  name: 'home',
  components: {
    ExploreVenue
  }
}
</script>

<style lang="scss">
body {
  margin: 0;
}
.home {
  padding: 30px 10px;
}
</style>