<template lang="pug">
  article.green.fixed.z3.overlay.bg-white.flex
    .col-12.max-width-3.mx-auto.flex.flex-column.justify-between.outline
      header
        .h-header.border-bottom.flex.justify-between.items-center
          .col-4.pl2
            button.pointer(@click="$emit('close')") Back
          h1.col-4.font-exp.center.nowrap {{invalidClover ? 'Not Found' : ''}}
          .col-4.pr2.right-align
            router-link.font-mono(:to="{name: 'Account/Trade'}") {{ prettyUserBalance }} ♣
      figure.flex-auto.relative
        //- image
        .keep__figure__img.absolute.bg-contain.bg-no-repeat.bg-center(role="img", v-if="clover && !invalidClover", :style="'background-image:url(' + cloverImage(clover) + ')'")
        .keep__figure__img.absolute.flex.items-center.justify-center.h1(v-else) <div class="h1">:-(</div>
        //- fav btn
        .absolute.bottom-0.right-0.p2(@click="saveClover(clover)", v-if="!invalidClover")
          <heart-icon class="green h1" :active="isSaved" />
      //- invalid clover
      footer(v-if="invalidClover").bg-green.white
        router-link(:to="{name: 'Field'}")
          button.col-12.h-bttm-bar.font-exp.pointer Find More
      //- keep / sell actions
      footer(v-else-if="!submitted")
        small.border-top.center.p2.block.h6.bg-white(v-if="sellValue > 0") You found a symmetrical clover! Keep it or claim a reward.
        .flex.border-top
          .col-6.flex-grow.p3.relative.pointer(@click="action = 'keep'")
            div(:class="{'opacity-25': action !== 'keep'}")
              small.block.lh1 Keep for ♣
              .font-exp.mt1.truncate {{keepValue}}
          .col-6.flex-grow.p3.relative.pointer.border-left(v-if="sellValue > 0", @click="action = 'sell'")
            div(:class="{'opacity-25': action !== 'sell'}")
              small.block.lh1 Claim reward ♣
              .font-exp.mt1.truncate {{sellValue}}
        //- confirm btn
        .bg-green.white
          button.col-12.h-bttm-bar.font-exp.pointer(@click="btnClick", :class="{'pointer-events-none': submitting}")
            span.block.m-auto.capitalize(v-show="!submitting") {{action}}
            wavey-menu.m-auto(v-show="submitting", :isWhite="true")
          transition(name="fade")
            .p2.center.font-mono(v-if="submitting")
              p {{ infoText }}
      //- submitting
      footer(v-else)
        .bg-green.white.col-12.h--bar.font-mono.items-center.pointer
          router-link.p3.center(:to="('/clovers/' + clover.board)")
            p.m-auto.pt2 Transaction complete! Click here to view Clover.
</template>

<script>
import Vue from 'vue'
import WaveyMenu from '@/components/Icons/WaveyMenu'
import HeartIcon from '@/components/Icons/HeartIcon'
import { mapGetters, mapMutations, mapActions } from 'vuex'
import { cloverImage, fetchCloudImage, prettyBigNumber, toDec } from '@/utils'
import { fromWei } from 'web3-utils'
import Reversi from 'clovers-reversi'
import BigNumber from 'bignumber.js'
const reversi = new Reversi()
let lastRt = null

export default {
  name: 'KeepClover',
  props: {
    movesString: {type: String, required: true}
  },
  head: {
    meta () {
      if (lastRt || !this.clover.board) return
      const img = fetchCloudImage(cloverImage({board: this.clover.board}, 640))
      return [{ p: 'og:image', c: img, id: 'og-img' }]
    }
  },
  data () {
    return {
      action: 'keep',
      value: null,
      reward: null,
      submitting: false,
      submitted: false
    }
  },
  watch: {
    _reversi () {
      this.checkClover()
    }
  },
  computed: {
    _reversi () {
      return reversi.playGameMovesString(this.movesString)
    },
    clover () {
      const id = `0x${this._reversi.byteBoard}`
      return {board: id, movesString: this.movesString}
    },
    invalidClover () {
      return this._reversi && this._reversi.error
    },
    keepValue () {
      // in club tokens
      return this.value ? toDec(fromWei(this.value.toString(10))) : '...'
    },
    sellValue () {
      // in club tokens
      return this.reward ? toDec(fromWei(this.reward.toString(10))) : '...'
    },
    infoText () {
      return this.action === 'keep'
        ? 'Your Clover is being submitted to the Contract. Once the Clover is verified by our Oracle, you will be confirmed as the owner.'
        : 'This reward is based on the rarity of the Clover. The Contract will buy this from you with Club Token (♣). Once the Oracle has verified the Clover you will receive the payout.'
    },
    isSaved () {
      if (!this.picks.length) return false
      return this.picks.findIndex(c => c.board === this.clover.board) >= 0
    },

    ...mapGetters(['picks', 'prettyUserBalance'])
  },
  methods: {
    cloverImage,

    btnClick () {
      if (this.action === 'keep') this.keep()
      if (this.action === 'sell') this.sellToBank()
    },
    async keep () {
      this.submitting = true
      try {
        const tx = await this.buy(this.clover)
        console.log(tx)
        this.submitting = false
        this.handleSuccess(
          `Success! You kept ${this.clover.board}`,
          this.clover
        )
      } catch (error) {
        console.log(error)
        this.submitting = false
        // notification
        this.handleError(error)
      }
    },
    async sellToBank () {
      this.submitting = true
      try {
        const tx = await this.sell({ clover: this.clover })
        console.log(tx)
        this.submitting = false
        this.handleSuccess(
          `Success! You sold ${this.clover.board} to the bank`,
          this.clover
        )
      } catch (error) {
        console.log(error)
        this.submitting = false
        // notification
        this.handleError(error)
      }
    },
    checkClover () {
      if (!this.clover) return null
      this.cloverExists('0x' + this._reversi.byteBoard).then((exists) => {
        if (!exists) return
        this.addMessage({
          type: 'error',
          title: 'This Clover already exists',
          msg: 'Click here to view the original',
          link: '/clovers/0x' + this._reversi.byteBoard
        })
      })
      const syms = this._reversi.returnSymmetriesAsBN()
      this.$store.dispatch('getReward', syms).then(wei => {
        wei = new BigNumber(wei)
        this.reward = wei
        this.value = wei.plus(this.$store.state.basePrice)
      })
    },
    handleError ({ message }) {
      this.selfDestructMsg({
        msg: message.replace('Error: ', ''),
        type: 'error'
      })
    },
    handleSuccess (msg, clover) {
      this.selfDestructMsg({msg, type: 'success'})
      this.$store.commit('REMOVE_SAVED_CLOVER', this.clover)
      this.submitted = true
    },

    ...mapMutations({saveClover: 'SAVE_CLOVER'}),
    ...mapActions(['buy', 'sell', 'addMessage', 'selfDestructMsg', 'cloverExists'])
  },
  beforeRouteEnter (to, from, next) {
    lastRt = from && from.name
    next()
  },
  mounted () {
    this.checkClover()
  },
  components: { WaveyMenu, HeartIcon }
}
</script>

<style lang="css">
.keep__figure__img {
  width: calc(100% - 2rem);
  height: calc(100% - 4rem);
  top: 2rem;
  left: 1rem;
}
</style>
