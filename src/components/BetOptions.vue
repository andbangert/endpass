<template>
  <div class="card">
    <div class="card-header">
      <div class="row">
        <div class="col-md-8"><div>Manual Betting</div></div>
        <div class="col-md-4">
          <div class="custom-control custom-checkbox float-right">
            <label class="custom-control-label" for="diceBot">Dice Bot</label>
            <input type="checkbox" class="custom-control-input" id="diceBot">
          </div>
      </div>
      </div>
    </div>
    <div class="card-body">
      <div class="row">
        <div class="col-md-6">
          <div class="form-group">
            <label for="txtBetAmout">Bet amount</label>
            <div class="input-group">
              <input id="txtBetAmount" type="text" class="form-control" :class="{ 'is-invalid': (!$v.betAmount.between||!$v.betAmount.required||!$v.betAmount.numeric)}" placeholder="Bet amount" aria-label="Bet amount" aria-describedby="basic-addon2" v-model.trim="$v.betAmount.$model">
              <div class="invalid-tooltip" :style="{ display: (!$v.betAmount.between ? 'block' : 'none')}">
                Please input bet amount between 1 and Balance
              </div>
              <div class="invalid-tooltip" :style="{ display: (!$v.betAmount.required ? 'block' : 'none')}">
                Please input bet amount
              </div>
              <div class="invalid-tooltip" :style="{ display: (!$v.betAmount.numeric ? 'block' : 'none')}">
                Input value must be a Number
              </div>
              <div class="input-group-append">
                <button class="btn btn-outline-secondary" type="button" @click="divideHalf()">1/2</button>
                <button class="btn btn-outline-secondary" type="button" @click="multiply()">x2</button>
                <button class="btn btn-outline-secondary" type="button" @click="maxBet()">MAX</button>
              </div>
            </div>
          </div>
          <div class="form-group">
            <label for="txtBetAmout">Win chance</label>
            <div class="input-group">
              <component :initialValue="winChancePercent" :value.sync="winChancePercent" symbol="%" v-bind:is="wcpName"></component>
              <div class="input-group-append">
                <button class="btn btn-outline-secondary" type="button" @click="wcpChange()" :disabled="wcm">{{wcp ? 'APPLY' : 'CHANGE'}}</button>
              </div>
            </div>
          </div>
        </div>
        <div class="col-md-6">
          <div class="form-group">
            <label for="txtBetAmout">Profit</label>
            <div class="input-group">
              <input id="txtBetAmount" type="text" class="form-control" placeholder="Bet amount" aria-label="Bet amount" aria-describedby="basic-addon2" :value="profit" readonly>
            </div>
          </div>
          <div class="form-group">
            <label for="txtBetAmout">Multiplier</label>
            <div class="input-group">
              <component :initialValue="winChanceMultipl" :value.sync="winChanceMultipl" symbol="X" v-bind:is="wcmName"></component>
              <div class="input-group-append">
                <button class="btn btn-outline-secondary" type="button" @click="wcmChange()" :disabled="wcp">{{wcm ? 'APPLY' : 'CHANGE'}}</button>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="row">
        <div class="col-md-6">
          <div>
            <button type="button" class="btn btn-secondary btn-lg btn-block" :disabled="wcm||wcp" @click="hiRoll()">HI {{highPercent}}</button>
          </div>
        </div>
        <div class="col-md-6">
          <div>
            <button type="button" class="btn btn-secondary btn-lg btn-block" :disabled="wcm||wcp" @click="lowRoll()">LO {{lowPercent}}</button>
          </div>
        </div>
      </div>
      <div class="row">
        <div class="col-md-12">
          <div class="mgt">
            <div class="result-area text-light" :class="rollClassObject">
              Rolled: {{rolledPercent}}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex'
import { componentName } from './winChanceInput'
import { NumberUtil } from '../utils/numberUtils'
import { required, numeric, between } from 'vuelidate/lib/validators'
import Balance from './Balance.vue'

const maxPercent = 99.99

export default {
  name: 'BetOptions',
  props: {
    initialBetAmount: {
      type: Number,
      default: 1
    },
    initialWinChancePercent: {
      type: Number,
      default: 50.0
    },
    initialWinChanceMultipl: {
      type: Number,
      default: 2.0
    },
    initialBalance: 100
  },
  data: function () {
    return {
      betAmount: this.initialBetAmount,
      // win chance percent change
      winChancePercent: this.initialWinChancePercent,
      winChanceMultipl: this.initialWinChanceMultipl,
      wcp: false,
      wcm: false,
      wcpName: componentName.winChanceAppliedName,
      wcmName: componentName.winChanceAppliedName
    }
  },
  validations: {
    betAmount: {
      required,
      // Needs to be mount balance (1.0, balance)
      between: between(1.0, 100.0),
      numeric
    },
    winChancePercent: {
      required,
      between: between(0.1, maxPercent),
      numeric
    },
    winChanceMultipl: {
      required,
      between: between(1.01, 990),
      numeric
    }
  },
  computed: {
    ...mapState({
      rolledPercent: 'rolledPercent',
      rollCount: state => state.stats.count,
      lastResult: state => state.stats.lastResult
    }),
    balance: function () {
      return 100
    },
    profit: function () {
      let num = NumberUtil.multiply(this.betAmount, this.winChanceMultipl)
      return NumberUtil.round(num, 8)
    },
    lowPercent: function () {
      return NumberUtil.round(this.winChancePercent, 2)
    },
    highPercent: function () {
      let num = NumberUtil.multiply(this.winChancePercent, -1)
      return NumberUtil.round(NumberUtil.add(maxPercent, num), 2)
    },
    rollClassObject: function () {
      if (this.rollCount === 0) {
        return {
          'bg-secondary': true
        }
      } else {
        return { 'bg-success': this.lastResult, 'bg-danger': !this.lastResult }
      }
    }
  },
  // Arrow functions does not work here!!!
  methods: {
    ...mapActions({
      roll: 'makeRoll'
    }),
    divideHalf: function () {
      let bet = NumberUtil.divide(this.betAmount, 2)
      this.betAmount = bet
    },
    multiply: function () {
      let bet = NumberUtil.multiply(this.betAmount, 2)
      this.betAmount = bet
    },
    maxBet: function () {
      this.betAmount = this.$store.state.balance
    },
    wcpChange: function () {
      this.wcp = !this.wcp
      this.wcpName = this.wcp ? componentName.winChanceChangeName : componentName.winChanceAppliedName
      // Recalculate multiplier
      if (!this.wcm && this.winChanceMultipl > 0) {
        let num = NumberUtil.divide(maxPercent, this.winChancePercent)
        this.winChanceMultipl = num
      }
    },
    wcmChange: function () {
      this.wcm = !this.wcm
      this.wcmName = this.wcm ? componentName.winChanceChangeName : componentName.winChanceAppliedName
      // Recalculate percentage
      if (!this.wcm && this.winChanceMultipl > 0) {
        let num = NumberUtil.divide(maxPercent, this.winChanceMultipl)
        this.winChancePercent = num
      }
    },
    hiRoll: function () {
      this.roll({
        betAmount: this.betAmount,
        isLow: false,
        winChance: this.winChancePercent
      }).then(resp => {}, error => {
        console.log(error)
        if (error) {
          console.log(error.message)
        }
      })
    },
    lowRoll: function () {
      this.roll({
        betAmount: this.betAmount,
        isLow: true,
        winChance: this.winChancePercent
      })
    }
  },
  components: { Balance }
}
</script>
