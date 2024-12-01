<script lang="ts" setup>
import { createAppKit, useAppKitAccount, useAppKitProvider } from '@reown/appkit/vue'
import { arbitrum, type AppKitNetwork } from '@reown/appkit/networks'
import { configs } from './configs'
import { EthersAdapter } from '@reown/appkit-adapter-ethers'
import { nftreeAbi } from './abis/nftree'
import { computed, ref, onMounted } from 'vue'
import { erc20Abi } from './abis/erc20'
import { BrowserProvider, Contract, Eip1193Provider } from 'ethers'
import axios from 'axios'

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
const nfts = ref<any[]>([])
const campaignId = ref<number | null>(1)

async function fetchCampaign() {
  if (!accountData.value.isConnected) return
  const { walletProvider } = useAppKitProvider('eip155')
  const ethersProvider = new BrowserProvider(walletProvider as Eip1193Provider)
  const signer = await ethersProvider.getSigner()
  // Reading the campaign data
  const NFTContract = new Contract(configs.contractAddress, nftreeAbi, signer)
  const result: any = await NFTContract.plantingCampaigns(campaignId.value)
  // Parsing the result
  campaign.value = JSON.parse(result[0] as string)
  campaign.value.id = campaignId.value
  startDate.value = new Date(Number(result[1]) * 1000)
  endDate.value = new Date(Number(result[2]) * 1000)
  totalTrees.value = result[3] as number
  const treeIds = await NFTContract.getCampaignTreeIds(campaignId.value)
  mintedTrees.value = treeIds.length
  pricePerTree.value = Number(result[7])
  tokenAddress.value = result[6] as string
  beneficiary.value = result[5] as string
  rifaiDonationFee.value = Number(result[8])
  clearInterval(fetchInterval)
  // Fetch token informations
  const tokenContract = new Contract(tokenAddress.value as `0x${string}`, erc20Abi, signer)
  const tokenSymbolResult = await tokenContract.symbol()
  tokenSymbol.value = tokenSymbolResult as string
  const tokenDecimalsResult = await tokenContract.decimals()
  tokenDecimals.value = Number(tokenDecimalsResult)
  getNFTsByOwner()
}

async function getNFTsByOwner() {
  if (!isConnected.value) return
  const { walletProvider } = useAppKitProvider('eip155')
  const ethersProvider = new BrowserProvider(walletProvider as Eip1193Provider)
  const signer = await ethersProvider.getSigner()
  const NFTContract = new Contract(configs.contractAddress, nftreeAbi, signer)
  // Fetching the NFTs
  const endpoint = `https://arb-mainnet.g.alchemy.com/nft/v3/${configs.alchemyApiKey}/getNFTsForOwner?owner=${accountData.value.address}&contractAddresses[]=${configs.contractAddress}`
  const result = await axios.get(endpoint)
  nfts.value = []
  const ids = []
  const parsedNfts = []
  for (const nft of result.data.ownedNfts) {
    ids.push(nft.tokenId)
    parsedNfts.push({
      tokenId: nft.tokenId,
      campaignId: await NFTContract.treeIdToCampaignId(nft.tokenId)
    })
  }
  const extendedMetadata = await NFTContract.getTreeExtendedMetadataBatch(ids)
  for (const nft of parsedNfts) {
    nfts.value.push({
      ...nft,
      extendedMetadata: extendedMetadata[nft.tokenId]
    })
  }
}

async function adoptTree() {
  if (!isConnected.value) return
  buttonText.value = 'Checking allowance...'
  buttonDisabled.value = true
  const { walletProvider } = useAppKitProvider('eip155')
  const ethersProvider = new BrowserProvider(walletProvider as Eip1193Provider)
  const signer = await ethersProvider.getSigner()
  // Checking the allowance
  const tokenContract = new Contract(tokenAddress.value as `0x${string}`, erc20Abi, signer)
  const allowance = await tokenContract.allowance(accountData.value.address, configs.contractAddress)
  console.log('Allowance', allowance)
  if (Number(allowance) < Number(pricePerTree.value)) {
    console.log('Not enough allowance, need to approve first')
    buttonText.value = 'Approving...'
    try {
      const result = await tokenContract.approve(configs.contractAddress, pricePerTree.value)
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
    const NFTContract = new Contract(configs.contractAddress, nftreeAbi, signer)
    const result = await NFTContract.adoptTree(campaignId.value, accountData.value.address)
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
onMounted(() => {
  const networks: [AppKitNetwork, ...AppKitNetwork[]] = [arbitrum]

  createAppKit({
    adapters: [new EthersAdapter()],
    networks,
    projectId: configs.projectId,
    metadata: configs.metadata,
    themeMode: 'light',
    features: {
      analytics: true
    }
  })
})
</script>

<template>
  <div class="container">
    <img src="/logo_header.svg" alt="Rifai logo" class="logo" />
    <h1>Adopt trees with Rifai Sicilia</h1>
    <p class="description">Adopt trees with Rifai Sicilia, you will receive a tree NFT for each tree you adopt.
      Our public planting campaign is live on Arbitrum, you can adopt trees from the campaign below or propose your own
      campaign by <a href="https://x.com/RifaiSicilia" target="_blank">contacting us</a>.</p>
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

        <div class="campaign-nfts">
          <h2>My adopted Trees</h2>
          <div class="nfts" v-if="nfts.length > 0">
            <div v-for="nft in nfts" :key="nft.tokenId">
              <div class="nft" v-if="nft.campaignId.toString() === campaign.id.toString()">
                <!-- <img :src="nft.extendedMetadata.image" alt="nft image" class="nft-image" /> -->
                <div class="tree">🌳</div>
                #{{ nft.tokenId }}
              </div>
            </div>
          </div>
          <div v-if="nfts.length === 0">
            <p>You have not adopted any trees yet 😭.</p>
          </div>
        </div>
      </div>
      <div v-if="!campaign" class="loading">
        <p>loading...</p>
      </div>
    </div>
  </div>
</template>