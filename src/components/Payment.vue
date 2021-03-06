<template>
  <div class="payment-container">
    <div class="music">
      <div class="music-download">
        <slot :callback="buyClick">
          <v-btn
            block
            large
            light
            outlined
            rounded
            color="#E56D9B"
            :height="44"
            @click.stop="buyClick"
          >
            {{ $t('buy') }}
          </v-btn>
        </slot>
      </div>
    </div>
    <v-dialog
      v-model="showWallet"
      width="360"
      @click:outside="outsideReset"
    >
      <v-card dark class="wallet">
        <h3 class="payment-title">
          {{ $t('paymentOf') }} 「{{ item.name }}」
        </h3>
        <div class="file-input-area" id="file-input-area">
          <v-icon v-if="!file" >mdi-plus</v-icon>
          <span v-else >{{ fileName }}</span>
          <input class="file-input" id="file-input" type="file" accept="application/json">
        </div>
        <p class="payment-content">
          {{ $t('pleaseInsertYourWalletKey') }}
        </p>
        <v-btn class="wallet-upload-button" depressed color="#E56D9B" :disabled="disAllowStep2" block @click="step2">
          Upload key
        </v-btn>
      </v-card>
    </v-dialog>
    <v-dialog
      v-model="showConfirm"
      width="360"
      :persistent="shouldLock"
      @click:outside="outsideReset"
    >
      <v-card dark class="confirm">
        <h3 class="confirm-title">
          {{ $t('paymentOf') }} 「{{ item.name }}」
        </h3>
        <p class="payment-content">
          {{ $t('orderReviewWarningPart1') }} <strong>{{ $t('confirm') }}</strong> {{ $t('orderReviewWarningPart2') }}
        </p>
        <div class="confirm-price">
          <div class="price-line">
            <span class="left-content">{{ $t('price') }}</span>
            <h4>{{ priceDisplay }} AR</h4>
          </div>
          <div class="price-line">
            <span class="left-content">{{ $t('fee') }}</span>
            <h4>{{ fee }} AR</h4>
          </div>
          <div class="price-line">
            <span class="left-content price-line-title-container">
              <span class="price-line-title-container-main-title-container">
                <span class="price-line-maintitle">{{ $t('tip') }}</span>
                <h4>{{ tip }} AR</h4>
              </span>
              <span class="price-line-title-container-sub-title-container">
                <span class="price-line-subtitle">{{ $t('toDeveloper') }} (~3%)</span>
                <span>{{ tipToDeveloperDisplay }} AR</span>
              </span>
              <span class="price-line-title-container-sub-title-container">
                <span class="price-line-subtitle">{{ $t('toCommunity') }} (~1%)</span>
                <span>{{ tipToCommunityDisplay }} AR</span>
              </span>
            </span>
          </div>
          <v-divider dark style="margin: 10px 0 10px"></v-divider>
          <div class="price-line">
            <span class="left-content">{{ $t('paymentTotal') }}</span>
            <h4>{{ total.toFixed(12).replace(/\.?0+$/, '') }} AR</h4>
          </div>
        </div>
        <v-btn class="wallet-upload-button" depressed color="#E56D9B" block :loading="paymentConfirm" @click="step3">
          {{ $t('confirm') }}
        </v-btn>
      </v-card>
    </v-dialog>
    <v-dialog
      v-model="showDialog"
      width="360"
      @click:outside="outsideReset"
    >
      <v-card dark>
        <div class="pay">
          <h3 class="pay-title">
            Payment of 「Test Music」
          </h3>
          <div class="pay-icon">
            <img src="@/assets/image/paymentCompleted.png" alt="Completed" />
          </div>
          <p class="pay-intro">
            Succeed to unlock the music！<br>
            <br>
            Id: {{ paymentId }}<br>
            <br>
            Your purchase will be available soon after next block.
          </p>

          <v-btn class="pay-button" depressed color="#E56D9B" block @click="showDialog = false">
            BACK TO MUSIC PLAYER
            <v-icon class="pay-button-icon">mdi-arrow-right</v-icon>
          </v-btn>
        </div>
      </v-card>
    </v-dialog>
    <v-snackbar
      v-model="failSnackbar"
      color="#E53935"
      timeout="3000"
      top="top"
      style="margin-top: 16px;"
    >
      {{ failMessage }}

      <template v-slot:action="{ attrs }">
        <v-btn
          dark
          text
          v-bind="attrs"
          @click="failSnackbar = false"
        >
          Close
        </v-btn>
      </template>
    </v-snackbar>
  </div>
</template>

<script>

import { mapActions, mapState } from 'vuex'

import API from '@/api/api'

export default {
  props: {
    price: {
      type: Number,
      default: 0
    },
    artist: {
      type: Object,
      default: () => {}
    },
    item: {
      type: Object,
      default: () => {}
    },
    type: {
      type: String,
      default: ''
    },
    wallet: {
      type: String,
      default: ''
    },
    itemId: {
      type: String,
      default: ''
    },
    trackNumber: {
      type: String,
      default: '0'
    }
  },
  data () {
    return {
      priceDisplay: 0,
      balance: 0,
      fee: 0.00000076,
      showWallet: false,
      disAllowStep2: true,
      showConfirm: false,
      showDialog: false,
      owned: false,
      paymentConfirm: false,
      failSnackbar: false,
      failMessage: '',
      fileName: '',
      keyFileContent: '',
      paymentAddress: '',
      file: '',
      tip: 0,
      tipToDeveloper: 0,
      tipToCommunity: 0,
      total: 0,
      shouldLock: false
    }
  },
  computed: {
    ...mapState(['username', 'paymentId', 'purchaseComplete']),
    tipToDeveloperDisplay () {
      return parseFloat(API.arweave.getArFromWinston(this.tipToDeveloper))
    },
    tipToCommunityDisplay () {
      return parseFloat(API.arweave.getArFromWinston(this.tipToCommunity))
    },
    purchaseSlogan () {
      if (this.item.duration !== -1) return `Pay ${this.price} AR for Full Version`
      else return `Pay ${this.price} AR to Download`
    }
  },
  watch: {
    showWallet (val) {
      if (val) {
        setTimeout(() => {
          const fileInput = document.getElementById('file-input')
          const droparea = document.getElementById('file-input-area')

          fileInput.addEventListener('dragenter', () => {
            this.addClass(droparea, 'is-active')
          })
          fileInput.addEventListener('click', () => {
            this.addClass(droparea, 'is-active')
          })
          fileInput.addEventListener('focus', () => {
            this.addClass(droparea, 'is-active')
          })

          fileInput.addEventListener('dragleave', () => {
            this.removeClass(droparea, 'is-active')
          })
          fileInput.addEventListener('blur', () => {
            this.removeClass(droparea, 'is-active')
          })
          fileInput.addEventListener('drop', () => {
            this.removeClass(droparea, 'is-active')
          })

          fileInput.addEventListener('change', () => {
            this.disAllowStep2 = true
            this.file = fileInput.files[0]
            this.fileName = this.file.name
            const reader = new FileReader()
            reader.readAsText(this.file)
            reader.onload = async (e) => {
              try {
                this.keyFileContent = JSON.parse(e.target.result)
                this.paymentAddress = await API.arweave.getAddress(this.keyFileContent)
                this.disAllowStep2 = false
              } catch (err) {
                this.failSnackbar = true
                this.failMessage = 'Unsupported File Type, Key file must be JSON'
              }
            }
          })
        }, 500)
      }
    },
    purchaseComplete (val) {
      if (val) {
        this.paymentConfirm = false
        this.showConfirm = false
        this.showDialog = true
        this.$emit('purchase-complete', true)
      }
    }
  },
  methods: {
    ...mapActions(['purchaseForItem']),
    buyClick () {
      if (!this.username) {
        this.failSnackbar = true
        this.failMessage = 'Please Login First'
        this.showWallet = false
        return
      }
      this.showWallet = true
      this.priceDisplay = this.price.toFixed(12).replace(/\.?0+$/, '')
    },
    async step2 () {
      if (this.type === 'album-info') {
        const trackStatus = await API.arweave.getAlbumItemPurchaseStatus(this.wallet, this.itemId, this.trackNumber)
        if (trackStatus) {
          this.failSnackbar = true
          this.failMessage = 'You have already bought this song'
          this.showWallet = false
          return
        }
      } else {
        let status
        if (this.type === 'album-full') status = await API.arweave.getAlbumPurchaseStatus(this.wallet, this.itemId)
        else status = await API.arweave.getItemPurchaseStatus(this.wallet, this.itemId)
        if (status) {
          this.failSnackbar = true
          this.failMessage = 'You have already bought this song'
          if (this.type === 'album-full') this.failMessage = 'You have already bought this album'
          this.showWallet = false
          return
        }
      }

      const balance = await API.arweave.getBalance(this.keyFileContent)
      if (balance < this.price) {
        this.failSnackbar = true
        this.failMessage = 'Insufficient funds, try another wallet'
        return
      }
      if (this.artist.id === this.paymentAddress) {
        this.failSnackbar = true
        this.failMessage = 'You cannot pay to your self'
        return
      }

      const res = await this.getFee(this.artist.id)
      this.fee = parseInt(API.arweave.getWinstonFromAr(this.fee)) + parseInt(res)
      this.fee = parseFloat(API.arweave.getArFromWinston(this.fee)).toFixed(12).replace(/\.?0+$/, '')

      const fullPrice = API.arweave.getWinstonFromAr(this.price)
      this.tipToDeveloper = parseInt(fullPrice * 0.03)
      this.tipToCommunity = parseInt(fullPrice * 0.01)
      const total = parseInt(fullPrice) + this.tipToDeveloper + this.tipToCommunity
      this.tip = this.tipToDeveloper + this.tipToCommunity
      this.tip = parseFloat(API.arweave.getArFromWinston(this.tip))
      this.tip = Number(this.tip)
      this.total = API.arweave.getArFromWinston(total)
      this.total = Number(this.total)

      this.showWallet = false
      this.showConfirm = true
    },
    step3 () {
      this.shouldLock = true
      const data = {
        target: this.artist.id,
        source: this.wallet,
        price: this.price,
        item: this.itemId,
        key: this.keyFileContent,
        type: this.type,
        toDeveloper: this.tipToDeveloper,
        toCommunity: this.tipToCommunity
      }
      if (this.type === 'album-info') {
        data.trackNumber = this.trackNumber
      }
      this.paymentConfirm = true
      this.purchaseForItem(data)
    },
    addClass (elem, className) {
      elem.classList.add(className)
    },
    removeClass (elem, className) {
      elem.classList.remove(className)
    },
    outsideReset () {
      const fileInput = document.getElementById('file-input')
      const droparea = document.getElementById('file-input-area')
      fileInput.removeEventListener('dragenter', () => {})
      fileInput.removeEventListener('click', () => {})
      fileInput.removeEventListener('focus', () => {})
      fileInput.removeEventListener('dragleave', () => {})
      fileInput.removeEventListener('blur', () => {})
      fileInput.removeEventListener('drop', () => {})
      fileInput.removeEventListener('change', () => {})
      if (fileInput) {
        this.removeClass(droparea, 'is-active')
      }

      this.showWallet = false
      this.showConfirm = false
      this.showDialog = false
    },
    toPlainString (num) {
      return ('' + num).replace(/(-?)(\d*)\.?(\d+)e([+-]\d+)/,
        function (a, b, c, d, e) {
          return e < 0
            ? b + '0.' + Array(1 - e - c.length).join(0) + c + d
            : b + c + d + Array(e - d.length + 1).join(0)
        })
    },
    async getFee (address) {
      const res = await API.arweave.getPaymentPrice(address)
      if (res) {
        if (res > 250000000000) return 250000000000
        else return 0
      }
    }
  },
  mounted () {
    this.fee = this.toPlainString(this.fee)
    this.priceDisplay = this.toPlainString(this.price)
  }
}
</script>

<style lang="less" scoped>
.wallet {
  padding: 24px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  .payment-content {
    margin-top: 24px;
    margin-bottom: 24px;
    text-align: left;
  }
}
.confirm {
  padding: 24px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  &-title {
    text-align: left;
  }
  &-price {
    margin-bottom: 16px;
    width: 100%;
  }
  .payment-content {
    margin-top: 24px;
    margin-bottom: 24px;
    text-align: left;
  }
  .price-line {
    display: flex;
    &-title-container-main-title-container {
      display: flex;
      .price-line-maintitle {
        flex: 1;
      }
    }
    &-title-container {
      display: flex;
      flex-direction: column;
      &-sub-title-container {
        text-indent: 10px;
        font-size: 14px;
        font-weight: 500;
        display: flex;
        .price-line-subtitle {
          flex: 1
        }
      }
    }
  }
  .left-content {
    flex: 1;
    text-align: left;
  }
}

.payment-title {
  margin-bottom: 24px;
  text-align: left;
}

#file-input-area {
  padding: 76px 24px 76px 24px;
  margin: 0 auto;
  height: 156px;
  width: 312px;
  border-radius: 4px;
  border: 2px dashed #B2B2B2;
  display: flex;
  justify-content: center;
  align-items: center;
  &.is-active {
    background-color: rgba(255, 255, 255, 0.171);
  }
  .file-input {
    position: absolute;
    left: 24px;
    top: 76px;
    height: 156px;
    width: 312px;
    cursor: pointer;
    opacity: 0;
    &:focus {
      outline: none;
    }
  }
}

.music {
  &-download {
    margin: 0 auto 0;
    max-width: 240px;
  }
}

.pay {
  padding: 24px;
  &-title {
    text-align: left;
    font-size: 20px;
    font-weight: 500;
    color: white;
    line-height: 28px;
    margin: 0 0 24px;
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 1;
    overflow: hidden;
    word-break: break-all;
  }
  &-icon {
    text-align: center;
    margin-bottom: 16px;
    img {
      width: 128px;
      height: 128px;
      min-width: 128px;
      min-height: 128px;
    }
  }
  &-intro {
    text-align: center;
    font-size: 16px;
    font-weight: 500;
    color: white;
    line-height: 22px;
    margin: 0 0 54px;
  }
  &-button {
    &-icon {
      font-size: 16px;
    }
  }
}
</style>
