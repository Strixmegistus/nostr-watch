<template>
  <span 
    v-if="this.store.jobs.getActiveSlug === slug"
    class="text-inherit">
    <span class="text-inherit">connecting to history relay</span>
  </span>
</template>

<style scoped>

</style>

<script>
import { defineComponent, toRefs } from 'vue'

import { setupStore } from '@/store'

import RelayMethods from '@/shared/relays-lib.js'
import SharedComputed from '@/shared/computed.js'

import { RelayPool } from 'nostr'

const localMethods = {
  invalidate(force){
    if( (!this.isExpired(this.slug, 1000) && !force) ) 
      return
    this.queueJob(
      this.slug, 
      () => {
        //console.log('checking history node')
        const $pool = new RelayPool(['wss://history.nostr.watch'])
        const begin = Date.now()
        const timeout = setTimeout( () => {
          this.store.status.historyNode = 0
          const currentMode = this.store.prefs.clientSideProcessing
          this.store.prefs.clientSideProcessingUpgrade = this.store.prefs.clientSideProcessing
          this.store.prefs.clientSideProcessing = true
          if(!currentMode)
            location.reload()
          this.closePool($pool)
          setTimeout(()=>this.store.jobs.completeJob(this.slug), 500)
        }, 5000)
        $pool
          .on('open', () => {
            clearTimeout(timeout)
            const delta = Date.now()-begin 
            if(delta < 1000)
              this.store.status.historyNode = 1
            else 
              this.store.status.historyNode = 2           
            this.store.jobs.completeJob(this.slug)
          })
      },
      true
    )
  },
  timeUntilRefresh(){
    return this.timeSince(Date.now()-(this.store.jobs.getLastUpdate(this.slug)+this.store.prefs.duration-Date.now())) 
  },
  timeSinceRefresh(){
    return this.timeSince(this.store.jobs.getLastUpdate(this.slug)) || Date.now()
  },
}

export default defineComponent({
  name: 'StatusCheckHistoryNode',
  components: {},
  data() {
    return {
      slug: 'status/history' //REMEMBER TO CHANGE!!!
    }
  },
  setup(props){
    const {resultsProp: results} = toRefs(props)
    return { 
      store : setupStore(),
      results: results
    }
  },
  created(){
    clearInterval(this.interval)
  },
  unmounted(){
    clearInterval(this.interval)
  },
  beforeMount(){
    this.lastUpdate = this.store.jobs.getLastUpdate(this.slug)
  },
  mounted(){
    //console.log('is processing', this.store.jobs.isJobActive(this.slug))

    if(this.store.jobs.isJobActive(this.slug))
      this.invalidate(true)
    else
      this.invalidate()

    // this.setRefreshInterval()
  },
  updated(){},
  computed: Object.assign(SharedComputed, {}),
  methods: Object.assign(localMethods, RelayMethods),
  props: {
    resultsProp: {
      type: Object,
      default(){
        return {}
      }
    },
  },
})
</script>

<style scoped>
 #refresh { font-size: 12pt; color:#666; margin-bottom:15px }
 #refresh button { cursor: pointer; border-radius: 3px; border: 1px solid #a0a0a0; color:#333 }
 #refresh button:hover {color:#000;}
 #refresh button[disabled] {color:#999 !important; border-color:#e0e0e0}
</style>

