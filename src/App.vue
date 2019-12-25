<template>
  <Default>
    <transition name="fade" appear>
      <router-view />
    </transition>
  </Default>
</template>

<script>
import Default from '~/layouts/Default.vue'
import '~/assets/styles/reset.scss'
import '~/assets/styles/global.scss'


export default {
  components: {
    Default
  }
}
</script>
