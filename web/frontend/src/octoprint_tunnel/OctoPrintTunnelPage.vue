<template>
  <div>
    <pull-to-reveal>
      <navbar view-name="app.views.tunnel_views.tunnel"></navbar>
    </pull-to-reveal>

    <div v-if="!isPro" class="floating-panel text-center pb-2">
      <div class="text-muted">Month-To-Date Usage/Free Limit(<a href="https://help.thespaghettidetective.com/kb/guide/en/octoprint-tunneling-Osas3m3O3L/Steps/300026,361069,366057,300029">?</a>)</div>
      <div :class="usageClass">{{ usageMTD }}/{{ humanizedUsageCap }}</div>
      <div v-if="overage">Your month-to-date tunneling usage is over the Free plan limit. <a type="button" class="btn btn-sm btn-primary" href="/ent/pricing/">Get Unlimited Tunneling</a></div>
    </div>
    <div>
      <iframe v-if="printerId" :src="iframeUrl()" class="tunnel-iframe"></iframe>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import split from 'lodash/split'
import filesize from 'filesize'
import urls from '@lib/server_urls'
import PullToReveal from '@common/PullToReveal.vue'
import Navbar from '@common/Navbar.vue'

export default {
  name: 'OctoPrintTunnelPage',

  components: {
    PullToReveal,
    Navbar,
  },
  data: function() {
    return {
      bytesMTD: null,
      usageCap: null,
      isPro: true,
      printerId: null,
    }
  },

  computed: {
    usageClass() {
      return {
        'text-success': this.bytesMTD < this.usageCap * 0.8,
        'text-warning': this.bytesMTD >= this.usageCap * 0.8 && this.bytesMTD < this.usageCap,
        'text-danger': this.bytesMTD >= this.usageCap
      }
    },
    usageMTD() {
      return filesize(this.bytesMTD)
    },
    humanizedUsageCap() {
      return filesize(this.usageCap)
    },
    overage() {
      return this.bytesMTD >= this.usageCap
    },
  },

  created() {
    this.isPro = JSON.parse(document.querySelector('#user-json').text).is_pro
    this.printerId = split(window.location.pathname, '/').slice(-2, -1).pop()
    this.usageCap = JSON.parse(document.querySelector('#settings-json').text).OCTOPRINT_TUNNEL_CAP
  },

  mounted() {
    this.$swal.DismissableToast({
        html: '<h4 class="text-center p-2"><img style="height: 1.1em;margin-right: 0.75em;" src="/static/img/octoprint-tunnel.png" />OctoPrint Secure Tunnel</h4><div class="p-1">It may take long time for OctoPrint page to load as it is securely tunneled via The Spaghetti Detective server.</div><div class="p-1"><a target="_blank" href="https://help.thespaghettidetective.com/kb/guide/en/octoprint-tunneling-Osas3m3O3L/Steps/300026,361069,300029">Learn more about OctoPrint Tunneling\'s security and page load speed. <i class="fas fa-external-link-alt"></i></a></div>',
        customClass: {
          container: 'dark-backdrop',
        },
      },
      'octoprint-tunnel.warning')

    const self = this
    const fetchUsage = () => {
      axios
        .get(urls.tunnelUsage())
        .then((resp) => {
          self.bytesMTD = resp.data.total
        })
    }
    setInterval(fetchUsage, 15*1000)
    setTimeout(fetchUsage, 4000)
  },

  methods: {
    iframeUrl() {
      return `/octoprint/${this.printerId}/`
    },
  }
}
</script>

<style lang="sass" scoped>
@use "~main/theme"

.tunnel-iframe
    width: 100%
    height: 100vh
    background: rgb(var(--color-white))
    position: absolute
    top: 0
    left: 0
    padding-top: 52px
    padding-bottom: 68px

.floating-panel
  position: fixed
  bottom: 15px
  right: 15px
  box-shadow: 2px 2px 10px rgba(0,0,0,.3)
  background-color: rgb(var(--color-body-bg))
  padding: 10px
  max-width: 300px
  z-index: 10
</style>
