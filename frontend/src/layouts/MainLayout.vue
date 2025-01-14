<template>
	<q-layout view="hHh lpR fff">
		<q-header elevated>
			<q-toolbar class="bg-toolbar text-white">
				<q-btn v-if="drawerEnabled" dense flat round icon="menu" @click="drawerOpened = !drawerOpened" />
				<a @click="home" class="gt-xs">
					<img :src="require(`@/assets/wban-logo-${currentBlockchain.network}.svg`)" class="currency-logo" />
				</a>
				<q-toolbar-title @click="home">{{ appTitle }}</q-toolbar-title>
				<blockchain-chooser />
				<q-avatar v-if="banAddress" class="gt-xs">
					<img @click="openBan(banAddress)" :src="banAddressPicture" :alt="banAddress" />
					<q-tooltip>{{ banAddress }}</q-tooltip>
				</q-avatar>
				<q-btn flat round dense @click="openNftPage()">
					<q-icon name="img:nft.svg" />
					<q-tooltip>wBAN NFTs</q-tooltip>
				</q-btn>
				<q-btn flat round dense icon="redeem" class="text-primary">
					<q-menu id="donations">
						<div class="row no-wrap q-pa-md">
							<div class="column">
								<div class="text-h6 q-mb-md">Donations</div>
								<p>Working on this app has been quite a journey, and still going on!</p>
								<p>
									Tips are always appreciated and if you would like to support my efforts, you can always help me by
									sending Banano tips at: <span class="banano-address">{{ banWalletForTips }}</span>
								</p>
							</div>
							<q-separator vertical inset class="q-mx-lg" />
							<div class="column items-center" v-if="$q.platform.is.desktop">
								<div class="text-subtitle1 q-mt-md q-mb-xs">Tip me at:</div>
								<q-icon :name="banWalletForTipsQRCode" size="200px" />
							</div>
							<q-btn v-if="$q.platform.is.mobile" color="primary" text-color="secondary" label="OK" v-close-popup />
						</div>
					</q-menu>
				</q-btn>
				<q-btn flat round dense icon="settings">
					<settings-menu />
				</q-btn>
			</q-toolbar>
		</q-header>
		<q-drawer
			v-if="drawerEnabled"
			v-model="drawerOpened"
			behavior="desktop"
			side="left"
			overlay
			elevated
			:width="230"
			:breakpoint="500"
		>
			<q-list>
				<q-item clickable v-ripple @click="depositBAN">
					<q-item-section avatar>
						<q-icon name="img:ban-deposit.svg" size="3em" />
					</q-item-section>
					<q-separator vertical inset />
					<q-item-section>Deposit BAN</q-item-section>
				</q-item>
				<q-item clickable v-ripple @click="withdrawBAN">
					<q-item-section avatar>
						<q-icon name="img:ban-withdraw.svg" size="3em" />
					</q-item-section>
					<q-separator vertical inset />
					<q-item-section>Withdraw BAN</q-item-section>
				</q-item>
				<q-item clickable v-ripple @click="swap">
					<q-item-section avatar>
						<q-icon name="img:wban-swap.svg" size="3em" />
					</q-item-section>
					<q-separator vertical inset />
					<q-item-section>Swap</q-item-section>
				</q-item>
				<q-item clickable v-ripple to="/farms">
					<q-item-section avatar>
						<q-icon name="img:wban-farming.svg" size="3em" />
					</q-item-section>
					<q-separator vertical inset />
					<q-item-section>Stake &amp; Farm</q-item-section>
				</q-item>
			</q-list>
		</q-drawer>
		<q-page-container>
			<q-banner v-if="!backendOnline || inError" inline-actions class="text-secondary text-center bg-primary">
				{{ errorMessage }}
				<br />
				<a v-if="errorLink !== ''" :href="errorLink">{{ errorLink }}</a>
			</q-banner>
			<router-view />
		</q-page-container>
		<q-footer class="bg-footer">
			<span @click="openGithub(appVersion)">
				<q-icon name="fab fa-github" size="20px" style="margin-top: -3px" />
				wBAN v{{ appVersion }}
			</span>
			-
			<span class="social">
				<q-icon name="live_help" color="white" size="24px" />
				<a
					v-if="$q.screen.gt.xs"
					href="https://wrap-that-potassium.gitbook.io/wban/introduction/quick-tour"
					target="_blank"
					>Documentation
				</a>

				<q-icon name="fab fa-discord" color="white" size="20px" />
				<a v-if="$q.screen.gt.xs" href="https://chat.banano.cc" target="_blank">Discord</a>

				<q-icon name="fab fa-telegram" color="white" size="20px" />
				<a v-if="$q.screen.gt.xs" href="https://t.me/banano_official" target="_blank">Telegram</a>
			</span>
		</q-footer>
	</q-layout>
</template>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator'
import { namespace } from 'vuex-class'
import { Screen } from 'quasar'
import router from '@/router'
import accounts from '@/store/modules/accounts'
import ban from '@/store/modules/ban'
import backend from '@/store/modules/backend'
import prices from '@/store/modules/prices'
import SettingsMenu from '@/components/SettingsMenu.vue'
import BlockchainChooser from '@/components/BlockchainChooser.vue'
import { blockchainAddressFilter } from '@/utils/filters'
import QRCode from 'qrcode'
import { openURL } from 'quasar'
import { Network, POLYGON_MAINNET } from '@/utils/Networks'

const accountsStore = namespace('accounts')
const banStore = namespace('ban')
const backendStore = namespace('backend')
const contractsStore = namespace('contracts')

@Component({
	components: {
		SettingsMenu,
		BlockchainChooser,
	},
	filters: {
		blockchainAddressFilter,
	},
})
export default class MainLayout extends Vue {
	@accountsStore.State('network')
	currentBlockchain!: Network

	@accountsStore.Getter('isUserConnected')
	isUserConnected!: boolean

	@accountsStore.State('activeAccount')
	activeAccount!: string

	@banStore.Getter('banAddress')
	banAddress!: string

	@banStore.Getter('banAddressPicture')
	banAddressPicture!: string

	@backendStore.Getter('setupDone')
	setupDone!: boolean

	@backendStore.Getter('online')
	backendOnline!: boolean

	@backendStore.Getter('inError')
	inError!: boolean

	@backendStore.Getter('errorMessage')
	errorMessage!: string

	@backendStore.Getter('errorLink')
	errorLink!: string

	@contractsStore.Getter('wbanAddress')
	wbanAddress!: string

	appTitle: string = process.env.VUE_APP_TITLE || 'wBAN -- Broken Release!!!'
	appVersion: string = process.env.VUE_APP_VERSION || '0'

	banWalletForTips = 'ban_1wban1mwe1ywc7dtknaqdbog5g3ah333acmq8qxo5anibjqe4fqz9x3xz6ky'
	banWalletForTipsQRCode = ''

	drawerOpened = false

	get drawerEnabled() {
		return Screen.lt.sm && this.isUserConnected
	}

	home() {
		if (this.$route.path != '/') {
			router.push('/')
		}
	}

	depositBAN() {
		document.dispatchEvent(new CustomEvent('deposit-ban'))
		this.drawerOpened = false
	}

	withdrawBAN() {
		document.dispatchEvent(new CustomEvent('withdraw-ban'))
		this.drawerOpened = false
	}

	swap() {
		document.dispatchEvent(new CustomEvent('swap'))
		this.drawerOpened = false
	}

	openBan(address: string) {
		openURL(`https://creeper.banano.cc/explorer/account/${address}`)
	}

	openGithub(version: string) {
		openURL(`https://github.com/wBanano/wban-dApp/releases/tag/v${version}`)
	}

	openNftPage() {
		if (this.isUserConnected && this.currentBlockchain.chainIdNumber === POLYGON_MAINNET.chainIdNumber) {
			router.push('/nft')
		} else {
			openURL('https://opensea.io/collection/wban')
		}
	}

	async onProviderChange() {
		await ban.init()
		await accounts.initWalletProvider()
		await backend.initBackend(this.banAddress)
		await prices.loadPrices()
		/*
		if (this.setupDone === false) {
			if (router.currentRoute.path !== '/setup') {
				router.push('/setup')
			}
		}
		*/
		try {
			const qrcode: string = await QRCode.toDataURL(this.banWalletForTips, {
				scale: 6,
				color: {
					dark: '2A2A2E',
					light: 'FBDD11',
				},
			})
			this.banWalletForTipsQRCode = `img:${qrcode}`
		} catch (err) {
			console.error(err)
		}
	}

	async mounted() {
		await this.onProviderChange()
		document.addEventListener('web3-connection', this.onProviderChange)
	}

	async unmounted() {
		await backend.closeStreamConnection()
		document.removeEventListener('web3-connection', this.onProviderChange)
	}
}
</script>

<style lang="sass">
@import '@/styles/quasar.sass'

.bg-toolbar
	background-color: $positive !important

.q-drawer
	background-color: $secondary
	.q-item
		color: $primary
	.q-separator
		background-color: darken($primary, 15%)
		margin-left: -10px
		margin-right: 10px

.bg-footer
	background-color: lighten($secondary, 20%) !important
	text-align: center
	font-size: 1em
	padding-top: 5px
	padding-bottom: 5px
.social
	.q-icon
		margin-right: 5px
	a
		font-weight: normal
		color: white
		margin-left: 5px
		margin-right: 10px

.btn-disconnect
	text-transform: none
	font-family: $monospaced-font
	margin-bottom: -5px

.danger
	background-color: $negative
	padding-bottom: 5px

@media (max-width: 900px)
	#donations
		max-width: 100% !important
		.column, .row
			display: block
	#settings
		max-width: 50% !important
@media (min-width: 900px)
	#donations
		max-width: 1000px
	#settings
		max-width: 400px !important

@media (min-width: $breakpoint-md-min)
	.q-page
		// background-image: url('../../public/blocklettuce.png')
		background-position: top right
		background-repeat: no-repeat
		background-attachment: fixed
		background-size: contain
</style>
