<template>
  <div>
    <Scoreboard />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import Scoreboard from './components/Scoreboard.vue';

export default defineComponent({
  components: { Scoreboard },
});
</script>
<style>
:root {
  background-color: #1f0c19;
  color: #ffffff;
}
</style>