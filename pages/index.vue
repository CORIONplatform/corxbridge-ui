<template>
  <v-app>
    <v-app-bar clipped-left app elevate-on-scroll color="#503FA1">
      <a href="http://corion.io" style="height: 100%; display: flex; align-items: center" target="blank">
        <v-img v-if="smAndUp" :src="assetsReqs[2]" max-height="60" max-width="220" style="position: absolute; left: 24px" />
        <v-img v-else :src="assetsReqs[3]" max-height="42" max-width="42" style="position: absolute; left: 24px" />
      </a>
      <v-spacer />
      <v-toolbar-title style="color: white; white-space: nowrap; overflow: hidden; text-overflow: ellipsis">
        {{ smAndUp ? dirChainName[0] : { EB: 'ETH', BE: 'BSC' }[direction] }}
        <v-icon color="white">mdi-chevron-right</v-icon>
        {{ smAndUp ? dirChainName[1] : { EB: 'BSC', BE: 'ETH' }[direction] }}
      </v-toolbar-title>
      <v-spacer />
      <v-dialog v-model="dialog" max-width="290">
        <template #activator="{ on, attrs }">
          <v-btn v-if="!paramsInfo || paramsInfo.PSD" style="position: absolute !important; right: 24px" large rounded disabled><b>Connect</b></v-btn>
          <v-btn v-else-if="smAndUp" color="#F53A1F" class="corx-button" large rounded dark v-bind="attrs" v-on="on"><b>Connect</b></v-btn>
          <v-btn v-else color="#F53A1F" class="corx-button" fab small dark v-bind="attrs" v-on="on"><v-icon>mdi-link</v-icon></v-btn>
        </template>
        <v-card>
          <v-card-title class="headline">Connect using:</v-card-title>
          <v-card-text class="flex-column">
            <v-btn
              color="orange"
              class="my-3"
              :style="providerType === 'M' ? 'color: #A5D6A7 !important; border-color: #A5D6A7' : ''"
              block
              outlined
              x-large
              :disabled="providerType === 'M'"
              @click="clickConnectMetamask"
            >
              Metamask
              <v-icon v-if="providerType === 'M'" right dark style="color: #a5d6a7 !important">mdi-check-circle-outline</v-icon>
            </v-btn>
            <v-btn
              color="blue"
              class="my-3"
              :style="providerType === 'W' ? 'color: #A5D6A7 !important; border-color: #A5D6A7' : ''"
              block
              outlined
              x-large
              :disabled="providerType === 'W'"
              @click="clickConnectWalletconnect"
            >
              Walletconnect
              <v-icon v-if="providerType === 'W'" right dark style="color: #a5d6a7 !important">mdi-check-circle-outline</v-icon>
            </v-btn>
          </v-card-text>
        </v-card>
      </v-dialog>
    </v-app-bar>
    <v-main>
      <v-container class="fill-height">
        <v-row justify="center" align="center">
          <v-col cols="12" sm="10" xl="7">
            <div style="width: 100%; position: relative; margin-top: 240px">
              <div style="width: 100%; position: absolute; bottom: 0">
                <v-alert v-if="loading_controllerInfo" type="info">Loading controller info...</v-alert>
                <v-alert v-else-if="loading_provider" type="info">Connecting provider...</v-alert>
                <v-alert v-else-if="loading_contractInfo" type="info">Loading contract info...</v-alert>
                <v-alert v-if="errorMessage" type="error">{{ errorMessage }}</v-alert>
                <v-alert v-if="warningMessage" type="warning">{{ warningMessage }}</v-alert>
                <v-alert v-if="inBetween" type="info">Value sent. Sending request to controller. It might take up to 5min, please wait...</v-alert>
                <v-alert v-if="hashes" type="success">Success! Save transaction hashes below</v-alert>
              </div>
            </div>
            <v-card flat>
              <div class="cardtop-address">{{ wallet }}</div>
              <v-row justify="center" justify-lg="space-between" align="stretch">
                <v-col cols="12" lg="3" style="position: relative; min-width: 296px">
                  <v-img
                    :src="dirChainLogo[0]"
                    position="left"
                    contain
                    height="100%"
                    width="300px"
                    style="position: absolute; z-index: 0; left: 16px; top: 16px; opacity: 0.1; overflow: hidden"
                  />
                  <v-container>
                    <v-card-title style="padding-bottom: 0"><b>From:</b></v-card-title>
                    <v-card-title style="font-size: 17px">
                      <b>{{ dirChainName[0] }}</b>
                    </v-card-title>
                    <v-card-text>
                      Balance: <b>{{ BNStrToNumstr(dirBalance[0]) }} CORX</b>
                      <br />
                      Approved: <b>{{ BNStrToNumstr(currentApproved) }} CORX</b>
                      <br />
                      Fee:
                      <b>{{
                        paramsInfo && costsInfo
                          ? `${paramsInfo.FEE > 0 ? `${Number(paramsInfo.FEE) / 100}% + ` : ''}` + `${_feeBNStr ? `${BNStrToNumstr(_feeBNStr)} CORX` : ''}`
                          : '...'
                      }}</b>
                      <br />
                      Min: <b>{{ BNStrToNumstr(minBNStr) }} CORX</b>
                    </v-card-text>
                  </v-container>
                </v-col>
                <v-col class="align-self-center flex-grow-1">
                  <v-container class="px-7 px-lg-3">
                    <v-text-field
                      v-if="contractInfoOk"
                      v-model="inputAmount"
                      :append-icon="approvedEnough && !approvedEqual ? 'mdi-restore' : canMaximizeAmount ? 'mdi-arrow-collapse-up' : ''"
                      :rules="[() => amountValid || 'Enter proper amount', () => amountEnough || 'Amount is too low', () => balanceEnough || 'Balance too low']"
                      label="Amount"
                      placeholder="123.456..."
                      required
                      @click:append="handleClickAppend"
                    />
                    <v-text-field v-else label="Amount" disabled />
                  </v-container>
                </v-col>
                <v-col cols="12" lg="3" style="position: relative; min-width: 296px">
                  <v-img
                    :src="dirChainLogo[1]"
                    position="right"
                    contain
                    height="100%"
                    width="300px"
                    style="position: absolute; z-index: 0; right: 16px; top: 16px; opacity: 0.1; overflow: hidden"
                  />
                  <v-container>
                    <v-card-title style="padding-bottom: 0"><b>To:</b></v-card-title>
                    <v-card-title style="font-size: 17px">
                      <b>{{ dirChainName[1] }}</b>
                    </v-card-title>
                    <v-card-text>
                      <span>
                        Balance: <b>{{ BNStrToNumstr(dirBalance[1]) }} CORX</b>
                        <br />
                      </span>
                      <span>
                        Will receive:
                        <b>
                          {{ amountEnough && controllerInfoOk ? '~' + BNStrToNumstr(substractFee(amountBN.toString(), feeBNStr).toString()) : '...' }}
                          CORX
                        </b>
                      </span>
                    </v-card-text>
                  </v-container>
                </v-col>
              </v-row>
              <v-card-text v-if="!!hashes">
                Transaction hashes:
                <br />
                Collect ({{ dirChainName[0] }})
                <a :href="`${dirChainExplorer[0]}/tx/${hashes.txHashCollect}`" target="_blank" rel="noopener noreferrer">
                  {{ hashes.txHashCollect }}
                </a>
                <br />
                Dispense ({{ dirChainName[1] }})
                <a :href="`${dirChainExplorer[1]}/tx/${hashes.txHashDispense}`" target="_blank" rel="noopener noreferrer">
                  {{ hashes.txHashDispense }}
                </a>
              </v-card-text>
              <v-card-actions>
                <v-spacer />
                <v-btn color="success" class="px-8" x-large :loading="loading_swapping" :disabled="requestDisabled" @click="handleClickRequest">
                  <b>Bridge</b>
                </v-btn>
                <v-spacer />
              </v-card-actions>
            </v-card>
            <v-row justify="center" style="margin: 16px 0 48px">
              <v-col align="right">
                <v-btn color="white" large text :disabled="loading_request || loading_contractInfo || preShutDown" @click="clickChangeDirection">
                  <b>Change direction{{ '  ' }}</b>
                  <v-icon>mdi-swap-horizontal</v-icon>
                </v-btn>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
    <v-footer color="#0c091d" class="py-15" style="border-top: 4px solid #f53a1f">
      <v-container>
        <v-row justify="center" style="color: white" dark>
          <v-col cols="12" sm="4">
            <a href="http://corion.io" style="height: 240px; width: 48px" target="blank">
              <v-img :src="assetsReqs[4]" width="240" height="48" />
            </a>
          </v-col>
          <v-col cols="12" sm="8">
            <v-row>
              <v-col class="corx-footer-sector corx-footer-sector-icons" cols="12" sm="4">
                <h3>SOCIAL</h3>
                <a target="blank" :href="footerLinks[0]">
                  <div><v-icon small>fab fa-facebook-square</v-icon>Facebook</div>
                </a>
                <a target="blank" :href="footerLinks[1]">
                  <div><v-icon small>fab fa-twitter</v-icon>Twitter</div>
                </a>
                <a target="blank" :href="footerLinks[2]">
                  <div><v-icon small>fab fa-telegram-plane</v-icon>Telegram</div>
                </a>
                <a target="blank" :href="footerLinks[3]">
                  <div><v-icon small>fab fa-medium-m</v-icon>Medium</div>
                </a>
                <a target="blank" :href="footerLinks[4]">
                  <div><v-icon small>fab fa-youtube</v-icon>Youtube</div>
                </a>
                <a target="blank" :href="footerLinks[5]">
                  <div><v-icon small>fab fa-linkedin</v-icon>Linkedin</div>
                </a>
                <a target="blank" :href="footerLinks[6]">
                  <div><v-icon small>fab fa-instagram</v-icon>Instagram</div>
                </a>
              </v-col>
              <v-col class="corx-footer-sector" cols="12" sm="4">
                <h3>LEGAL</h3>
                <a target="blank" :href="footerLinks[7]"><div>Terms & Conditions</div></a>
                <a target="blank" :href="footerLinks[8]"><div>Privacy Policy</div></a>
                <a target="blank" :href="footerLinks[9]"><div>Whitepaper</div></a>
              </v-col>
              <v-col class="corx-footer-sector" cols="12" sm="4">
                <h3>HELP</h3>
                <a target="blank" :href="footerLinks[10]"><div>Support</div></a>
                <a target="blank" :href="footerLinks[11]"><div>WALLET</div></a>
                <a target="blank" :href="footerLinks[12]"><div>FAQ</div></a>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-container>
      <div class="footer-tag">
        Powered by
        <a href="https://corion.io/" target="blank">CorionX</a> & <a href="https://gotbit.io/" target="blank">Gotbit</a> developers
      </div>
    </v-footer>
  </v-app>
</template>

<script lang="ts">
/* eslint-disable camelcase */
import Vue from 'vue'
import { Contract, providers, BigNumber } from 'ethers'
import Axios from 'axios'
import firebase from 'firebase/app'
import 'firebase/firestore'
import Token from '~/abis/Token.json'
import infuraKey from '~/infuraKey.json'
if (!firebase.apps.length)
  firebase.initializeApp({
    authDomain: 'corxbridge.firebaseapp.com',
    projectId: 'corxbridge',
    storageBucket: 'corxbridge.appspot.com',
    messagingSenderId: '878478192824',
    appId: '1:878478192824:web:cf731e5732124b0592444d',
  })
const db = firebase.firestore()
const address_BAB = '0xEFA7eAb30F3DdDFC3926F4083f319d7B6238BFD7' // BSC Mainnet: 0xEFA7eAb30F3DdDFC3926F4083f319d7B6238BFD7 | BSC Testnet: 0xCA661795d34535dB71bB09e58fE32a5c654Ce7b8
const address_BAE = '0x7302b2F207b02Bcc2dea68C0950EB0Dc6C695b84' // ETH Mainnet: 0x7302b2F207b02Bcc2dea68C0950EB0Dc6C695b84 | ETH Ropsten: 0xE5B9DD83e066650804fC28BCE24d372f09Fd5228
const address_TKNB = '0x36184181FA321E350aaAF88dad723E281365c1Ac' // BSC Mainnet: 0x36184181FA321E350aaAF88dad723E281365c1Ac | BSC Testnet: 0x00EE45b4dED7df3B85a0305a5c7014c7dA455ad3
const address_TKNE = '0x26a604DFFE3ddaB3BEE816097F81d3C4a2A4CF97' // ETH Mainnet: 0x26a604DFFE3ddaB3BEE816097F81d3C4a2A4CF97 | ETH Ropsten: 0x604B20031d473bfEc55f8546fA11b6E328E8f1e0
const _providerE = new providers.InfuraProvider(1, infuraKey)
const _providerB = new providers.JsonRpcProvider('https://bsc-dataseed.binance.org/')
const TKNE = new Contract(address_TKNE, Token.abi, _providerE)
const TKNB = new Contract(address_TKNB, Token.abi, _providerB)

function removeTrailingZeros(str: string): string {
  if (str === '0') return str
  if (str.slice(-1) === '0') return removeTrailingZeros(str.substr(0, str.length - 1))
  if (str.slice(-1) === '.') return str.substr(0, str.length - 1)
  return str
}
function numstrToBN(input: string): BigNumber {
  const spl = input.split('.')
  if (spl[1]) spl[1] = spl[1].substr(0, 8)
  return BigNumber.from(spl.join('') + '00000000'.substr(0, 8 - (spl[1] || '').length))
}
function BNStrToNumstr(str: string, precision = 3): string {
  if (str === '0') return str
  if (isNaN(Number(str))) return 'NaN'
  if (str.length <= 8) return removeTrailingZeros(('0.' + '00000000'.substr(0, 8 - str.length) + str).substr(0, 8 - str.length + precision + 2))
  else return [str.substr(0, str.length - 8), str.slice(-8)].join('.').substr(0, str.length - 8 + precision + 1)
}
function substractFee(amount: string, fee: string) {
  return BigNumber.from(amount).sub(fee)
}
const footerLinks = [
  'https://www.facebook.com/CorionFoundation',
  'https://twitter.com/CorionPlatform',
  'https://t.me/corionx',
  'https://medium.com/@Corion',
  'https://www.youtube.com/channel/UCD9royvfm_-02vV9eHvZk8A',
  'https://www.linkedin.com/company/corionplatform/',
  'https://www.instagram.com/corionplatform/',
  'https://corion.io/wp-content/uploads/2020/05/Terms-and-Conditions-of-CorionX-Token.pdf',
  'https://corion.io/wp-content/uploads/2020/05/Privacy-Policy.pdf',
  'https://corion.io/corionx-whitepaper-en/#',
  'https://t.me/corionx%20',
  'https://share.trustwallet.com/yQi5KRh%20',
  'https://docs.google.com/document/d/1Cuh6JtXKaO_N9x1-W1Cdmgnj7yye6_RykvlZS91TmHM/edit?usp=sharing',
]
const assetsReqs = [
  require('@/assets/ethereumlogo.png'),
  require('@/assets/binancelogo.png'),
  require('@/assets/corxbridgelogo.png'),
  require('@/assets/corxlogo.png'),
  require('@/assets/corionx-1.svg'),
]
export default Vue.extend({
  data() {
    return {
      footerLinks,
      assetsReqs,
      provider: null as providers.Web3Provider | null,
      signer: null as providers.JsonRpcSigner | null,
      wallet: '',
      direction: 'EB' as 'EB' | 'BE',
      approvedE: '',
      approvedB: '',
      balanceE: '',
      balanceB: '',
      inputAmount: '',
      paramsInfo: null as {
        CTF: number
        FTM: number
        FEE: number
        PSD: boolean
      } | null,
      costsInfo: null as {
        BE: string
        EB: string
      } | null,
      errorMessoge_misc: '',
      warningMessage: '',
      loading_swapping: false,
      loading_approve: false,
      loading_request: false,
      loading_provider: false,
      loading_controllerInfo: true,
      loading_contractInfo: false,
      inBetween: false,
      hashes: null as { txHashCollect: string; txHashDispense: string } | null,
      dialog: false,
      chainId: null as null | number,
    }
  },
  computed: {
    // ERRORS
    err_controllerInfo(): boolean {
      return !this.loading_controllerInfo && !this.controllerInfoOk
    },
    err_shutDown(): boolean {
      return this.preShutDown
    },
    err_chainId(): boolean {
      return !!this.chainId && this.chainId !== { EB: 1, BE: 56 }[this.direction]
    },
    err_balanceLow(): boolean {
      return !!this.dirBalance[0] && BigNumber.from(this.dirBalance[0]).lt(this.minBNStr)
    },
    errorMessage(): string {
      return this.err_controllerInfo
        ? 'Could not load info from controller. Usage is blocked. Try refreshing the page'
        : this.err_shutDown
        ? 'Controller will soon shut down for maintenance. Usage is blocked. Please wait'
        : this.err_chainId
        ? `You selected wrong network for this direction. Select ${this.dirChainName[0]}${this.providerType === 'M' ? '' : ' and refresh the page'}`
        : this.err_balanceLow
        ? 'Your balance is lower than minimum amount. Usage is blocked'
        : this.errorMessoge_misc
    },
    smAndUp(): boolean {
      return (this as any).$vuetify.breakpoint.smAndUp
    },
    providerType(): 'N' | 'W' | 'M' {
      return !this.provider ? 'N' : this.provider.provider.isMetaMask ? 'M' : 'W'
    },
    dirChainName(): string[] {
      return { EB: ['Ethereum', 'Binance Smart Chain'], BE: ['Binance Smart Chain', 'Ethereum'] }[this.direction]
    },
    dirChainExplorer(): string[] {
      return { EB: ['https://etherscan.io', 'https://bscscan.com'], BE: ['https://bscscan.com', 'https://etherscan.io'] }[this.direction]
    },
    dirChainLogo(): string[] {
      return { EB: [assetsReqs[0], assetsReqs[1]], BE: [assetsReqs[1], assetsReqs[0]] }[this.direction]
    },
    dirBalance(): string[] {
      return { EB: [this.balanceE, this.balanceB], BE: [this.balanceB, this.balanceE] }[this.direction]
    },
    currentApproved(): string {
      return { EB: this.approvedE, BE: this.approvedB }[this.direction]
    },
    // TECHNICAL OK
    preShutDown(): boolean {
      return !!this.paramsInfo?.PSD
    },
    contractInfoOk(): boolean {
      return !!this.currentApproved && !!this.dirBalance[0]
    },
    controllerInfoOk(): boolean {
      return !!this.paramsInfo && !!this.costsInfo
    },
    allSafe(): boolean {
      return (
        !!this.wallet &&
        this.controllerInfoOk &&
        this.contractInfoOk &&
        !this.errorMessage &&
        !this.loading_controllerInfo &&
        !this.loading_provider &&
        !this.loading_contractInfo
      )
    },
    // VALID DATA
    approvedEqual(): boolean {
      return this.amountBN.eq(this.currentApproved)
    },
    amountValid(): boolean {
      return !!Number(this.inputAmount)
    },
    approvedNonZero(): boolean {
      return this.approvedBN.gt(0)
    },
    // ENOUGH
    amountEnough(): boolean {
      return this.minBNStr ? this.amountBN.gte(this.minBNStr) : false
    },
    balanceEnough(): boolean {
      return this.amountBN.lte(this.dirBalance[0])
    },
    approvedEnough(): boolean {
      if (!this.controllerInfoOk) return false
      return this.approvedBN.gte(this.minBNStr)
    },
    // BUTTONS DISABLED
    requestDisabled(): boolean {
      return !(this.allSafe && this.amountValid && this.amountEnough && this.balanceEnough)
    },
    canMaximizeAmount(): boolean {
      return !this.approvedNonZero && this.amountBN.lt(this.dirBalance[0])
    },
    // CONVERTERS
    amountBN(): BigNumber {
      return this.amountValid ? numstrToBN(this.inputAmount) : BigNumber.from(0)
    },
    approvedBN(): BigNumber {
      return BigNumber.from(this.currentApproved || 0)
    },
    _feeBNStr(): string {
      if (!this.controllerInfoOk) return ''
      return BigNumber.from(this.costsInfo![this.direction]).mul(this.paramsInfo!.CTF).div(100).toString()
    },
    feeBNStr(): string {
      if (!this.controllerInfoOk) return ''
      return this.amountBN.mul(this.paramsInfo!.FEE).div(10000).add(this._feeBNStr).toString()
    },
    minBNStr(): string {
      if (!this.controllerInfoOk) return ''
      return BigNumber.from(this.costsInfo![this.direction]).mul(this.paramsInfo!.CTF).mul(this.paramsInfo!.FTM).mul(120).div(1000000).toString()
    },
  },
  async mounted() {
    if (!this.smAndUp) window.scrollTo(0, 160)
    await this.loadControllerInfo()
  },
  methods: {
    BNStrToNumstr,
    substractFee,
    async clickChangeDirection() {
      this.switchDirection()
      this.chainId = (await this.provider?.getNetwork())?.chainId || null
    },
    switchDirection() {
      this.warningMessage = ''
      this.hashes = null
      this.direction = this.direction === 'EB' ? 'BE' : 'EB'
    },
    async clickConnectWalletconnect() {
      if (this.providerType === 'M') this.wallet = ''
      await this.connectWalletconnect()
      if (this.wallet) await this.loadContractInfo()
      if (this.contractInfoOk && this.approvedNonZero) this.restoreInputAmount()
    },
    async connectWalletconnect() {
      this.loading_provider = true
      try {
        const WalletConnectProvider = require('@walletconnect/web3-provider').default
        const wc = new WalletConnectProvider({ infuraId: infuraKey, qrcode: true })
        await wc.enable()
        this.provider = new providers.Web3Provider(wc)
        this.signer = this.provider.getSigner()
        this.wallet = await this.signer.getAddress()
        this.chainId = (await this.provider.getNetwork())?.chainId || null
      } catch (error) {
        this.errorMessoge_misc = 'Could not connect Walletconnect. Error: ' + error.message
        console.error(error)
      }
      this.loading_provider = false
      this.dialog = false
    },
    async clickConnectMetamask() {
      if (this.providerType === 'W') this.wallet = ''
      await this.connectMetamask()
      ;(window.ethereum as any).on('chainChanged', async (chainId: string) => {
        console.log({ chainId })
        this.direction = chainId === '0x38' ? 'BE' : 'EB'
        await this.connectMetamask()
        if (this.contractInfoOk && this.approvedNonZero) this.restoreInputAmount()
      })
      if (this.wallet) await this.loadContractInfo()
      if (this.contractInfoOk && this.approvedNonZero) this.restoreInputAmount()
    },
    async connectMetamask() {
      this.loading_provider = true
      try {
        if (!window.ethereum) throw new Error('Please set up MetaMask properly')
        await (window.ethereum as any).enable()
        this.provider = new providers.Web3Provider((window.ethereum as any) || window.web3)
        this.signer = this.provider.getSigner()
        this.wallet = await this.signer.getAddress()
        this.chainId = (await this.provider.getNetwork())?.chainId || null
        this.direction = this.chainId === 56 ? 'BE' : 'EB'
      } catch (error) {
        this.errorMessoge_misc = 'Could not connect MetaMask. Error: ' + error.message
        console.error(error)
      }
      this.loading_provider = false
      this.dialog = false
    },
    restoreInputAmount() {
      this.inputAmount = removeTrailingZeros(BNStrToNumstr(this.currentApproved, 8))
    },
    handleClickAppend() {
      if (this.approvedEnough && !this.approvedEqual) this.restoreInputAmount()
      else if (this.canMaximizeAmount) this.inputAmount = removeTrailingZeros(BNStrToNumstr(this.dirBalance[0], 8))
    },
    async loadControllerInfo() {
      this.loading_controllerInfo = true
      try {
        this.paramsInfo = ((await db.collection('config').doc('changeables').get()).data() as any) || null
        this.costsInfo = (await Axios.get('https://corxbridge.uc.r.appspot.com/info/costs')).data
        if (!this.paramsInfo) throw new Error('Received invalid data from firestore')
        if (!this.costsInfo) throw new Error('Received invalid data from controller')
        console.log(this.costsInfo)
      } catch (error) {
        console.error(error)
      }
      this.loading_controllerInfo = false
    },
    async loadContractInfo() {
      this.loading_contractInfo = true
      try {
        const w = this.wallet
        const [Bb, Eb, Ba, Ea] = (
          await Promise.all([TKNB.balanceOf(w), TKNE.balanceOf(w), TKNB.allowance(w, address_BAB), TKNE.allowance(w, address_BAE)])
        ).map((r) => r.toString())
        this.approvedB = Ba
        this.approvedE = Ea
        this.balanceB = Bb
        this.balanceE = Eb
      } catch (error) {
        console.error(error)
      }
      this.loading_contractInfo = false
    },
    async handleClickRequest() {
      this.loading_swapping = true
      try {
        if (!this.approvedNonZero || !this.approvedEqual) {
          await this.approve()
          this.inBetween = true
        }
        this.inputAmount = ''
        await this.requestSwap()
      } catch (error) {
        this.warningMessage = error.message
        console.error(error)
      }
      this.inBetween = false
      this.loading_swapping = false
    },
    async approve() {
      this.loading_approve = true
      const TKN = new Contract({ EB: address_TKNE, BE: address_TKNB }[this.direction], Token.abi, this.signer!)
      console.log({
        amt: this.amountBN.toString(),
        bal: this.dirBalance[0],
      })
      const tx = await TKN.approve({ EB: address_BAE, BE: address_BAB }[this.direction], this.amountBN)
      console.log({ tx })
      const receipt = await tx.wait()
      console.log({ receipt })
      await this.loadContractInfo()
      this.loading_approve = false
    },
    async requestSwap() {
      this.loading_request = true
      try {
        this.hashes = (await Axios.get(`https://corxbridge.uc.r.appspot.com/process?direction=${this.direction}&address=${this.wallet}`)).data
        console.log(this.hashes)
      } catch (error) {
        console.error({ ...error })
        throw new Error(error.response.data)
      }
      await this.loadContractInfo()
      this.loading_request = false
    },
  },
})
</script>

<style>
.v-application {
  font-family: 'AvenirNext';
}
div[data-app='true'] {
  background: url('~assets/bg.png') no-repeat center center fixed !important;
  background-size: cover !important;
}
.corx-button {
  position: absolute !important;
  right: 24px;
  box-shadow: 0px 2px 10px 0px rgb(255 82 33 / 50%) !important;
}
.corx-footer-button {
  box-shadow: 0px 2px 10px 0px rgb(255 82 33 / 50%) !important;
}
.corx-footer-sector {
  display: flex;
  flex-direction: column;
}
.corx-footer-sector a {
  color: white !important;
  text-decoration: none;
}
.corx-footer-sector a:hover {
  color: #f53a1f !important;
}
.corx-footer-sector h3 {
  color: #5b586d;
  font-size: 14px;
  margin-bottom: 16px;
}
.corx-footer-sector-icons div {
  position: relative;
  padding-left: 40px;
}
.corx-footer-sector-icons div .fab {
  color: white !important;
  position: absolute;
  left: 2px;
}
.corx-footer-sector div {
  height: 28px;
  display: flex;
  flex-direction: row;
  align-items: center;
}
.cardtop-address {
  position: absolute;
  top: 0;
  color: #aaa;
  width: 100%;
  padding: 10px 28px;
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 14px;
}
.footer-tag {
  margin: 40px 0 -40px;
  width: 100%;
  text-align: center;
  font-size: 15px;
  font-family: 'AvenirNext';
  color: #ccc !important;
}
.footer-tag a {
  text-decoration: none;
  color: rgb(108, 167, 255) !important;
}
</style>
