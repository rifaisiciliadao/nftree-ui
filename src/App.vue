<script lang="ts" setup>
import { createAppKit } from '@reown/appkit/vue'
import { arbitrum, base, type AppKitNetwork } from '@reown/appkit/networks'
import { WagmiAdapter } from '@reown/appkit-adapter-wagmi'

const projectId = '33f21597377d115f723f71b1de23b4f2'

const metadata = {
  name: 'Rifai Sicilia - Adopt Trees',
  description: 'Adopt Trees with Rifai Sicilia',
  url: 'https://adopt.rifasicilia.com',
  icons: ['https://github.com/rifaisicilidao/nftree-ui/blob/main/public/logo.png']
}

const networks: [AppKitNetwork, ...AppKitNetwork[]] = [arbitrum, base]
const wagmiAdapter = new WagmiAdapter({
  networks,
  projectId
})

createAppKit({
  adapters: [wagmiAdapter],
  networks,
  projectId,
  metadata,
  features: {
    analytics: true
  }
})
</script>

<template>
  <appkit-button />
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}

.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}

.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
