<template>
  <li v-if="showBet" class="bet container">
    <div class="row">
      <div class="five columns">
        <h2>{{ p1 }}</h2>
      </div>
      <div class="two columns">&nbsp;</div>
      <div class="five columns">
        <h2>{{ p2 }}</h2>
      </div>
    </div>
    <div class="row relative">
      <div class="five columns">
        <img
          class="flag selectable u-max-full-width"
          v-bind:src="require(`@/assets/teams/${p1}.png`)"
          v-on:click.stop.prevent="select(p1)"
          v-bind:class="{ notselected : isP2Selected, selected : isP1Selected }">
      </div>
      <div class="two columns">
        <div class="vs">VS</div>
        <div class="match-context">
          {{ matchContext }}
        </div>
        <div class="small">
          Match {{ matchId+1 }}<span class="gray">/64</span>
        </div>
      </div>
      <div class="five columns">
        <img
          class="flag selectable u-max-full-width"
          v-bind:src="require(`@/assets/teams/${p2}.png`)"
          v-on:click.stop.prevent="select(p2)"
          v-bind:class="{ notselected : isP1Selected, selected : isP2Selected }">
      </div>
    </div>

    <template v-if="showBetStats">
      <div class="row" v-show="isMetamaskNetworkLoginReady">
        <div class="five columns">
          <i>{{ toEther(myBetsP1).toFixed(3) }} {{ unit }}</i>
        </div>
        <div class="two columns small gray yourbets">Your&nbsp;bets</div>
        <div class="five columns">
          <i>{{ toEther(myBetsP2).toFixed(3) }} {{ unit }}</i>
        </div>
      </div>
      <div class="stats">
        <div class="row">
          <div class="eight columns offset-by-two">
            <div class="left">
              Pot: {{ pot }} {{ unit }}
            </div>
            <div class="statsbar-right">
              <div class="statsbar-left" v-bind:style="getLeftBarWidthPool"></div>
            </div> 
          </div>
        </div>
        <div class="row">
          <div class="eight columns offset-by-two">
            <div class="left">
              Number of bets: {{ numBetsP1+numBetsP2 }}
            </div>
            <div class="statsbar-right">
              <div class="statsbar-left" v-bind:style="getLeftBarWidthNum"></div>
            </div> 
          </div>
        </div>
      </div>
    </template>
    
    <!-- The order of these if checks is important, as each check is narrowing down the bet's status further -->
    <div class="row phase">
      <div class="eight columns offset-by-two">

        <!-- CANCELLED -->

        <div v-if="isCancelled">
          <BetPhaseCancelled
            :instance="instance"
            :timeMatchStarts="timeMatchStarts"
            :hasPayouts="hasPayouts"
            @claimed="updateContractBetState()"/>
        </div>

        <!-- INACTIVE -->

        <div v-else-if="isInactive">
          <BetPhaseInactive
            :timeBettingOpens="timeBettingOpens"
            :timeMatchStarts="timeMatchStarts"/>
        </div>

        <!-- CLAIM EXPIRED -->

        <div v-else-if="isClaimExpired">
          <BetPhaseExpired
            :timeClaimsExpire="timeClaimsExpire"
            :getWinnerPhrase="getWinnerPhrase"/>
        </div>

        <!-- BETTING OPEN -->

        <div v-else-if="isBettingOpen">
          <BetPhaseOpen
            :instance="instance"
            :p1="p1"
            :p2="p2"
            :betTeam="betTeam"
            :timeBettingCloses="timeBettingCloses"
            :timeMatchStarts="timeMatchStarts"
            @bet-placed="updateContractBetState">
          </BetPhaseOpen>
        </div>

        <!-- BETTING CLOSED -->

        <div v-else-if="isBettingClosed">
          <BetPhaseClosed
            :timeMatchStarts="timeMatchStarts"/>
        </div>

        <!-- PAYOUT -->

        <div v-else-if="isPayout">
          <BetPhasePayout
            v-bind:instance="instance"
            v-bind:timeClaimsExpire="timeClaimsExpire"
            v-bind:hasPayouts="hasPayouts"
            v-bind:getWinnerPhrase="getWinnerPhrase"
            @claimed="getContractState()">
          </BetPhasePayout>
        </div>

        <!-- WAIT CONFIRM -->

        <div v-else-if="isWaitingForConfirm">
          <BetPhaseWaitForConfirm
            :timeFetchConfirmEnds="timeFetchConfirmEnds"
            :getWinnerPhrase="getWinnerPhrase"/>
        </div>

        <!-- TOO LATE FOR CONFIRM -->

        <div v-else-if="isTooLateForConfirm">
          <BetPhaseTooLateConfirm
            :instance="instance"
            :timeFetchConfirmEnds="timeFetchConfirmEnds"
            @claimed="getContractState()"/>
        </div>

        <!-- MATCH IN PLAY -->

        <div v-else-if="isPlayingForSure">
          <BetPhaseMatchInPlay
            :timeMatchStarts="timeMatchStarts"
          />
        </div>

        <!-- IS FUNDING NEEDED -->
        <div v-else-if="isFundingNeeded">
          <BetPhaseFundingNeeded
            :instance="instance"
            :timeFetchConfirmEnds="timeFetchConfirmEnds"
            :fundingNeeded="fundingNeeded"
            @funded="getContractState()"/>
        </div>


        <!-- IS TOO LATE FOR FETCHING -->

        <div v-else-if="isTooLateForSuggest">
          <BetPhaseTooLateSuggest
            :instance="instance"
            :timeFetchConfirmEnds="timeFetchConfirmEnds"
            @claimed="getContractState()"/>
        </div>

        <!-- IS FETCHING -->

        <div v-else-if="isFetching">
          <BetPhaseFetching
            :timeFetchConfirmEnds="timeFetchConfirmEnds"/>
        </div>

        <!-- SHOULD START FETCH -->

        <div v-else-if="isShouldStartFetch">
          <BetPhaseShouldStartFetch
            v-bind:instance="instance"
            v-bind:matchId="matchId"
            v-bind:account="account"
            v-bind:timeFetchConfirmEnds="timeFetchConfirmEnds"
            @claimed="getContractState()">
            >
          </BetPhaseShouldStartFetch>
        </div>

        <!-- UNKNOWN -->

        <div v-else>
          <BetPhaseUnknown
            :timeMatchStarts="timeMatchStarts"/>
        </div>

      </div>
    </div>
    <div class="row monospace">
      <a v-bind:href="etherscanURL" target="_blank" class="gray">
        {{ address }}
      </a>
    </div>
    <template v-if="(isOwner && canBeCancelled)">
      <ActionCancel
        :instance="instance"
        @success="getContractState()"/>
    </template>
    <template v-if="(isOwner && canBeConfirmed)" class="confirm">
      <ActionConfirm
        :instance="instance"
        :p1="p1"
        :p2="p2"
        @success="getContractState()"/>
    </template>
  </li>
</template>

<script>
import { mapState, mapGetters } from 'vuex'
import Helpers from '../utils/Helpers.js'
import BetPhaseCancelled from './phases/BetPhaseCancelled.vue'
import BetPhaseMatchInPlay from './phases/BetPhaseMatchInPlay.vue'
import BetPhaseExpired from './phases/BetPhaseExpired.vue'
import BetPhaseInactive from './phases/BetPhaseInactive.vue'
import BetPhasePayout from './phases/BetPhasePayout.vue'
import BetPhaseOpen from './phases/BetPhaseOpen.vue'
import BetPhaseClosed from './phases/BetPhaseClosed.vue'
import BetPhaseShouldStartFetch from './phases/BetPhaseShouldStartFetch.vue'
import BetPhaseWaitForConfirm from './phases/BetPhaseWaitForConfirm'
import BetPhaseTooLateConfirm from './phases/BetPhaseTooLateConfirm.vue'
import BetPhaseUnknown from './phases/BetPhaseUnknown.vue'
import BetPhaseFundingNeeded from './phases/BetPhaseFundingNeeded.vue'
import BetPhaseTooLateSuggest from './phases/BetPhaseTooLateSuggest.vue'
import BetPhaseFetching from './phases/BetPhaseFetching.vue'
import ActionCancel from './phases/bases/ActionCancel.vue'
import ActionConfirm from './phases/bases/ActionConfirm.vue'

const monthNames = ["January", "February", "March", "April", "May", "June",
  "July", "August", "September", "October", "November", "December"
];

export default {
  name: 'Bets',
  components: {
    BetPhaseCancelled, BetPhaseMatchInPlay, BetPhaseExpired, BetPhaseInactive, BetPhasePayout, BetPhaseOpen, BetPhaseClosed, BetPhaseShouldStartFetch, BetPhaseWaitForConfirm, BetPhaseTooLateConfirm, BetPhaseFundingNeeded, BetPhaseTooLateSuggest, BetPhaseFetching, BetPhaseUnknown, ActionCancel, ActionConfirm
  },
  mixins: [Helpers],
  props: ['matchId', 'registryInstance', 'hideFinishedMatches'],
  data () {
    return {
      // bet contract
      instance: null,
      address: '',
      // bet state data
      rawStatus: '',
      matchContext: '',
      p1: 'Team 1',
      p2: 'Team 2',
      isFetchingStarted: false,
      fundingNeeded: '0',
      matchFinished: false,
      isWinnerFetched: false,
      isWinnerConfirmed: false,
      myBetsP1: '0',
      myBetsP2: '0',
      totalP1: '0',
      totalP2: '0',
      numBetsP1: 0,
      numBetsP2: 0,
      timeBettingOpens: 1577836800,
      timeBettingCloses: 1577836800,
      timeFetchConfirmEnds: 1577836800,
      timeClaimsExpire: 1577836800, // high inital value to improve UX
      rawWinner: 0, // 1 == p1, 2 == p2, 3 == draw

      // UI
      betTeam: '',
    }
  },
  computed: {
    ...mapState({
      web3: state => state.web3,
      contract: state => state.bets.contract,
      account: state => state.account
    }),
    ...mapGetters({
      isMetamaskNetworkLoginReady: 'isMetamaskNetworkLoginReady'
    }),
    isOwner () {
      return this.account == process.env.ADDRESS_OWNER.toLowerCase()
    },
    etherscanURL () {
      return process.env.ETHERSCAN_URL + this.address + process.env.ETHERSCAN_APPENDIX
    },
    getNow () {
      return (new Date().getTime() / 1000).toFixed(0);
    },
    pot () {
      return (this.toEther(this.totalP1) + this.toEther(this.totalP2)).toFixed(3)
    },
    timeMatchStarts: function() {
      return this.timeBettingCloses + 15*60
    },
    timeMatchEndsEarliest: function() {
      return this.timeMatchStarts + 105*60
    },
    showBet () {
      return !(this.hideFinishedMatches && this.matchFinished)
    },
    showBetStats () {
      return !(this.isInactive || this.isClaimExpired)
    },
    canBeConfirmed () {
      return !this.isWinnerConfirmed && this.isWinnerFetched && !this.isCancelled
    },
    canBeCancelled () {
      return !this.isWinnerConfirmed && !this.isCancelled //&& !this.isClaimExpired // should not be required
    },
    isCancelled () {
      return this.rawStatus == "0"
    },
    isInactive () {
      console.log("call to isInactive")
      return this.getNow < this.timeBettingOpens
    },
    isClaimExpired () {
      console.log("call to isClaimExpired")
      return this.getNow > this.timeClaimsExpire
    },
    isBettingOpen () {
      console.log("call to isBettingOpen")
      return this.getNow >= this.timeBettingOpens &&
        this.getNow < this.timeBettingCloses
    },
    isBettingClosed () {
      console.log("call to isBettingClosed")
      return this.getNow >= this.timeBettingCloses && this.getNow < this.timeMatchStarts
    },
    isPayout () {
      console.log("call to isPayout")
      return this.matchFinished == true &&
        this.isWinnerConfirmed == true// && this.getNow < this.timeClaimsExpire
    },
    isWaitingForConfirm () {
      console.log("call to isWaitingForConfirm")
      return this.matchFinished == true &&
        this.isWinnerFetched == true &&
        this.isWinnerConfirmed == false &&
        this.getNow < this.timeFetchConfirmEnds
    },
    isTooLateForConfirm () {
      console.log("call to isTooLateForConfirm")
      return this.matchFinished == true &&
        this.isWinnerFetched == true &&
        this.isWinnerConfirmed == false &&
        this.getNow >= this.timeFetchConfirmEnds
    },
    isPlayingForSure () {
      console.log("call to isPlayingForSure")
      return this.getNow >= this.timeMatchStarts &&
        this.getNow <= this.timeMatchEndsEarliest
    },
    isFundingNeeded () {
      console.log("call to isFundingNeeded")
      return this.isWinnerFetched == false && Number(this.fundingNeeded) > 0 && this.getNow < this.timeFetchConfirmEnds
    },
    isTooLateForSuggest () {
      console.log("call to isTooLateForSuggest")
      return this.isWinnerFetched == false &&
        this.getNow >= this.timeFetchConfirmEnds
    },
    isFetching () {
      console.log("call to isFetching")
      return this.isFetchingStarted == true && this.isWinnerFetched == false &&
        this.getNow < this.timeFetchConfirmEnds
    },
    isShouldStartFetch () {
      return this.isFetchingStarted == false &&
        this.getNow > this.timeMatchEndsEarliest
    },
    getWinnerPhrase () {
      if (this.rawWinner == 1) {
        return this.p1 + ' won the match!'
      } else if (this.rawWinner == 2) {
        return this.p2 + ' won the match!'
      } else {
        return 'Match resulted in a draw!'
      }
    },
    getLeftBarWidthPool () {
      if (this.totalP1 != '0' || this.totalP2 != '0') {
          return "width: " + ((this.toEther(this.totalP1) / (this.toEther(this.totalP1)+this.toEther(this.totalP2))) * 100) + "%;"
      } else {
          return "width: 100%; background-color: #bbb;"
      }
    },
    getLeftBarWidthNum () {
      if (this.numBetsP1 != '0' || this.numBetsP2 != '0') {
        return "width: " + ((this.numBetsP1 / (this.numBetsP1+this.numBetsP2)) * 100) + "%;"
      } else {
        return "width: 100%; background-color: #bbb;"
      }
    },
    isP1Selected () {
      return this.betTeam === this.p1
    },
    isP2Selected () {
      return this.betTeam === this.p2
    },
    hasPayouts () {
      if (this.rawWinner == 1 && this.myBetsP1 != '0') {
        return true
      } else if (this.rawWinner == 2 && this.myBetsP2 != '0') {
        return true
      } else if (this.rawWinner == 3 && (this.myBetsP1 != '0' || this.myBetsP2 != '0')) {
        return true
      } else {
        return false
      }
    }
  },
  methods: {
    getContractState: async function() {
      this.getStatus()
      this.getMatchContext()
      this.getP1()
      this.getP2()
      this.updateContractBetState()
      this.getMatchFinished()
      this.getTimeBettingOpens()
      this.getTimeBettingCloses()
      this.getTimeFetchConfirmEnds()
      this.getTimeClaimsExpire()
      this.getIsFetchingStarted()
      this.getIsWinnerFetched()
      this.getIsWinnerConfirmed()
      this.getWinner()
      this.getFundingNeeded()
    },
    updateContractBetState: async function() {
      this.getMyBetsP1()
      this.getMyBetsP2()
      this.getTotalP1()
      this.getTotalP2()
      this.getNumBetsP1()
      this.getNumBetsP2()
    },
    select: async function (player) {
      if(this.isBettingOpen) {
        if (player == this.p1) {
          this.betTeam = this.p1
        } else {
          this.betTeam = this.p2
        }
        console.log(this.betTeam + " selected.")
      }
    },
    getMatchContext: function () {
      this.instance.matchContext.call().then(mc => {
        this.matchContext = mc
      })
    },
    getStatus: function () {
      this.instance.status.call().then(s => {
        this.rawStatus = s
      })
    },
    getP1: function () {
      this.instance.p1.call().then(s => {
        this.p1 = s
      })
    },
    getP2: function () {
      this.instance.p2.call().then(s => {
        this.p2 = s
      })
    },
    getMatchFinished: function() {
      this.instance.matchFinished.call().then(mf => {
        this.matchFinished = mf
      })
    },
    getMyBetsP1: async function () {
      this.myBetsP1 = (await this.instance.betsP1.call(this.account)).toString()
    },
    getMyBetsP2: async function () {
      this.myBetsP2 = (await this.instance.betsP2.call(this.account)).toString()
    },
    getTotalP1: async function () {
      this.totalP1 = (await this.instance.totalP1.call()).toString()
    },
    getTotalP2: async function () {
      this.totalP2 = (await this.instance.totalP2.call()).toString()
    },
    getNumBetsP1: async function () {
      this.numBetsP1 = Number(await this.instance.numBetsP1.call())
    },
    getNumBetsP2: async function () {
      this.numBetsP2 = Number(await this.instance.numBetsP2.call())
    },
    getTimeBettingOpens: async function () {
      this.timeBettingOpens = Number(await this.instance.timeBettingOpens.call())
    },
    getTimeBettingCloses: async function() {
      this.timeBettingCloses = Number(await this.instance.timeBettingCloses.call())
    },
    getTimeFetchConfirmEnds: async function() {
      this.timeFetchConfirmEnds = Number(await this.instance.timeFetchConfirmEnds.call())
    },
    getTimeClaimsExpire: async function() {
      this.timeClaimsExpire = Number(await this.instance.timeClaimsExpire.call())
    },
    getIsWinnerFetched: async function() {
      this.isWinnerFetched = await this.instance.winnerFetched.call()
    },
    getIsWinnerConfirmed: async function() {
      this.isWinnerConfirmed = await this.instance.winnerConfirmed.call()
    },
    getWinner: async function() {
      this.rawWinner = Number(await this.instance.winner.call())
    },
    getFundingNeeded: async function() {
      this.instance.fundingNeeded.call().then(result => {
        this.fundingNeeded = String(result)
      }).catch(err => {
        console.warn("getFundingNeeded behaves unexpectedly: " + err)
      })
    },
    getIsFetchingStarted: async function() {
      this.isFetchingStarted = await this.instance.isFetchingStarted.call()
    }
  },
  created () {
    this.registryInstance.betContracts.call(this.matchId).then(address => {
      this.address = address
      this.contract.at(address).then(instance => {
        this.instance = instance
        this.getContractState()
      })
    })
  },
  watch: {
    account: function (newAccount) {
      this.getMyBetsP1()
      this.getMyBetsP2()
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
.bet {
  position: relative;
  margin: 15px 0px;
  padding: 15px;
  border: 1px solid #bbb;
  background-color: rgb(250,250,250);
}
.yourbets {
  line-height: 2.5rem !important;
}
.flag {
  border: 1px solid #ddd;
}
.vs {
  font-size: 4rem;
  font-weight: bold;
}
.match-context {
  margin-top: 5px;
}
.stats {
  margin: 20px 0px;
}
.statsbar-right {
  background-color: rgb(168, 220, 255);
}
.statsbar-left {
  background-color: rgb(207, 255, 168);
  height: 8px;
}
.phase {
  margin-top: 25px;
}
.selectable {
  cursor: pointer;
}
.notselected {
  filter: grayscale(50%);
}
.selected {
  border: none;
  outline: 6px solid rgb(111, 175, 38);
}
</style>

<!-- TODO: improve default rendered content that shows at the very beginning of the loading. -->