<script lang="ts" setup>
import { createAppKit, useAppKitAccount } from '@reown/appkit/vue'
import { arbitrum, type AppKitNetwork } from '@reown/appkit/networks'
import { WagmiAdapter } from '@reown/appkit-adapter-wagmi'
import { configs } from './configs'
import { cookieStorage, createStorage, readContract, writeContract } from '@wagmi/core'
import { nftreeAbi } from './abis/nftree'
import { computed, ref } from 'vue'
import { erc20Abi } from './abis/erc20'

const metadata = {
  name: 'Rifai Sicilia - Adopt Trees',
  description: 'Adopt Trees with Rifai Sicilia',
  url: 'https://adopt.rifasicilia.com',
  icons: ['https://github.com/rifaisicilidao/nftree-ui/blob/main/public/logo.png']
}

const networks: [AppKitNetwork, ...AppKitNetwork[]] = [arbitrum]
const wagmiAdapter = new WagmiAdapter({
  networks,
  projectId: configs.projectId,
  storage: createStorage({
    storage: cookieStorage
  }),
})

createAppKit({
  adapters: [wagmiAdapter],
  networks,
  projectId: configs.projectId,
  metadata,
  themeMode: 'light',
  features: {
    analytics: true
  }
})

const accountData = useAppKitAccount()
const isConnected = computed(() => accountData.value.isConnected)
const campaign = ref<any>(null)
const startDate = ref<Date | null>(null)
const endDate = ref<Date | null>(null)
const totalTrees = ref<number | null>(null)
const mintedTrees = ref<number | null>(null)
const pricePerTree = ref<number | null>(null)
const tokenAddress = ref<string | null>(null)
const beneficiary = ref<string | null>(null)
const tokenSymbol = ref<string | null>(null)
const tokenDecimals = ref<number | null>(null)
const rifaiDonationFee = ref<number | null>(null)
const buttonText = ref<string>('Adopt a tree')
const buttonDisabled = ref<boolean>(false)

async function fetchCampaign() {
  if (!isConnected.value) return
  const result: any = await readContract(wagmiAdapter.wagmiConfig, {
    address: configs.contractAddress as `0x${string}`,
    abi: nftreeAbi,
    functionName: 'plantingCampaigns',
    args: [1],
  })
  campaign.value = JSON.parse(result[0] as string)
  startDate.value = new Date(Number(result[1]) * 1000)
  endDate.value = new Date(Number(result[2]) * 1000)
  totalTrees.value = result[3] as number
  mintedTrees.value = result[4] as number
  pricePerTree.value = Number(result[7])
  tokenAddress.value = result[6] as string
  beneficiary.value = result[5] as string
  rifaiDonationFee.value = Number(result[8])
  clearInterval(fetchInterval)
  // Fetch token informations
  const tokenSymbolResult = await readContract(wagmiAdapter.wagmiConfig, {
    address: tokenAddress.value as `0x${string}`,
    abi: erc20Abi,
    functionName: 'symbol',
  })
  tokenSymbol.value = tokenSymbolResult as string
  const tokenDecimalsResult = await readContract(wagmiAdapter.wagmiConfig, {
    address: tokenAddress.value as `0x${string}`,
    abi: erc20Abi,
    functionName: 'decimals',
  })
  tokenDecimals.value = Number(tokenDecimalsResult)
}

async function adoptTree() {
  if (!isConnected.value) return
  buttonText.value = 'Checking allowance...'
  buttonDisabled.value = true
  const allowance = await readContract(wagmiAdapter.wagmiConfig, {
    address: tokenAddress.value as `0x${string}`,
    abi: erc20Abi,
    functionName: 'allowance',
    args: [accountData.value.address, configs.contractAddress],
  })
  console.log('Allowance', allowance)
  if (Number(allowance) < Number(pricePerTree.value)) {
    console.log('Not enough allowance, need to approve first')
    buttonText.value = 'Approving...'
    try {
      const result = await writeContract(wagmiAdapter.wagmiConfig, {
        address: tokenAddress.value as `0x${string}`,
        abi: erc20Abi,
        functionName: 'approve',
        args: [configs.contractAddress, pricePerTree.value],
      })
      console.log(result)
      setTimeout(() => {
        buttonText.value = 'Waiting for confirmation...'
        adoptTree()
      }, 1500)
    } catch (error) {
      buttonText.value = 'Adopt a tree'
      buttonDisabled.value = false
      console.error(error)
    }
    return
  }
  buttonText.value = 'Adopting tree..'
  try {
    const result = await writeContract(wagmiAdapter.wagmiConfig, {
      address: configs.contractAddress as `0x${string}`,
      abi: nftreeAbi,
      functionName: 'adoptTree',
      args: [1, accountData.value.address],
    })
    console.log(result)
    await fetchCampaign()
    buttonDisabled.value = false
  } catch (error) {
    buttonText.value = 'Adopt a tree'
    buttonDisabled.value = false
    console.error(error)
  }
}

const fetchInterval = setInterval(fetchCampaign, 1000)
</script>

<template>
  <div class="container">
    <h1>Adopt trees with Rifai</h1>
    <appkit-button />
    <div class="campaign" v-if="isConnected">
      <div v-if="campaign" class="campaign-card">
        <div class="campaign-header">
          <img :src="campaign.image" alt="campaign image" class="campaign-image" />
          <h2 class="campaign-title">{{ campaign.name }}</h2>
        </div>

        <p class="campaign-description">{{ campaign.description }}</p>

        <div class="campaign-details">
          <div class="detail-item">
            <span class="label">Start Date</span>
            <span class="value">{{ startDate?.toLocaleDateString() }}</span>
          </div>
          <div class="detail-item">
            <span class="label">End Date</span>
            <span class="value">{{ endDate?.toLocaleDateString() }}</span>
          </div>
          <div class="detail-item">
            <span class="label">Adopted Trees</span>
            <span class="value">{{ mintedTrees }}</span>
          </div>
          <div class="detail-item">
            <span class="label">Total Trees</span>
            <span class="value">{{ totalTrees }}</span>
          </div>
          <div class="detail-item" v-if="pricePerTree && tokenDecimals">
            <span class="label">Price per Tree</span>
            <span class="value">{{ (pricePerTree / 10 ** tokenDecimals).toFixed(2) }} {{ tokenSymbol }}</span>
          </div>
          <div class="detail-item" v-if="rifaiDonationFee && tokenDecimals">
            <span class="label">Rifai donation fee</span>
            <span class="value">{{ (rifaiDonationFee / 10 ** tokenDecimals).toFixed(2) }} {{ tokenSymbol }}</span>
          </div>
        </div>

        <div class="campaign-footer">
          <div class="address-item">
            <span class="label">Beneficiary</span>
            <span class="value truncate">{{ beneficiary }}</span>
          </div>
          <button class="adopt-button" :disabled="buttonDisabled" @click="adoptTree">{{
            buttonText }}</button>
        </div>
      </div>
      <div v-if="!campaign" class="loading">
        <p>loading...</p>
      </div>
    </div>
  </div>
</template>