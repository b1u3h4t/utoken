<template>
  <div class="trade">
    <van-popup
      v-model="showPop"
      position="bottom"
      class="popup-wapper"
      style="width:100%;height: 100%;">
      <van-nav-bar
        @click-left="close()">
        <div slot="title">
          <span>{{tradepair.baseCode}}</span>
          <span>/</span>
          <span>{{tradepair.counterCode}}</span>
        </div>
        <span slot="left"><i class="ultfont ult-left"></i></span>
        <i slot="right" @click="toAddTradepair" class="ultfont ult-plus"></i>
      </van-nav-bar>
      <pl-content-block :offsetTop="46">
        <div style="padding-top: 10px;padding-left: 20px;text-align: center;">
          <span class="bs-btn buy-btn" :class="bsFlag" @click="toBs('buy')" v-text="$t('trade.buying')"></span>
          <span class="bs-btn sell-btn" :class="bsFlag" @click="toBs('sell')" v-text="$t('trade.selling')"></span>
        </div>
        <div class="bs-block item-block">
          <div class="text-muted b-white item">
            <span>{{$t('trade.youAsset')}}&nbsp;&nbsp;&nbsp;{{balances[bsCode] | cutTail}}&nbsp;{{bsCode}}</span>
          </div>
          <div class="item">
            <van-row>
              <van-col span="11">
                <span v-if="bsFlag === 'buy'" v-text="$t('trade.buyPrice')"></span><span v-else-if="bsFlag === 'sell'" v-text="$t('trade.sellPrice')"></span>({{tradepair.counterCode}})
              </van-col>
              <van-col span="11" offset="2">
                {{$t('trade.amount')}}({{tradepair.baseCode}})
              </van-col>
            </van-row>
            <van-row>
              <van-col span="11">
                <van-field
                  v-if="bsFlag === 'buy'"
                  v-model="form.price"
                  type="number"
                  class="large-font text-primary"
                  style="background: #f1f8fb"
                  :placeholder="`${$t('trade.buyPrice')}(${tradepair.counterCode})`">
                </van-field>
                <van-field
                  v-if="bsFlag === 'sell'"
                  type="number"
                  v-model="form.price"
                  class="large-font text-primary"
                  style="background: #f1f8fb"
                  :placeholder="`${$t('trade.sellPrice')}(${tradepair.counterCode})`">
                </van-field>
              </van-col>
              <van-col span="11" offset="2">
                <van-field
                  class="large-font text-primary"
                  v-model.number="form.amount"
                  type="number"
                  style="background: #f1f8fb"
                  :placeholder="`${$t('trade.amount')}(${tradepair.baseCode})`">
                </van-field>
              </van-col>
            </van-row>
          </div>
          <div class="item">
            <van-row>
              <van-col span="11" class="text-muted" v-text="$t('trade.estVal')"></van-col>
            </van-row>
            <van-row>
              <van-col span="11" class="text-muted">
                {{expectedTradePrice | cutTail}}&nbsp;{{tradepair.counterCode}}
              </van-col>
              <van-col span="11" offset="2">
                <vue-slider
                  style="position: relative;top: -12px;"
                  v-model="form.amount"
                  :debug="false"
                  :height="2"
                  :tooltip="false"
                  :interval.sync="step"
                  :max.sync="maxVal"></vue-slider>
              </van-col>
            </van-row>
          </div>
        </div>
        <div class="single-btn margin-top margin-bottom">
          <van-button size="large" round type="primary" @click="toInputPassword" v-if="bsFlag === 'buy'">{{$t('trade.buying')}}{{tradepair.baseCode}}</van-button>
          <van-button size="large" round type="danger" @click="toInputPassword" v-else-if="bsFlag === 'sell'">{{$t('trade.selling')}}{{tradepair.baseCode}}</van-button>
        </div>
        <van-tabs v-model="tabActive" swipeable sticky @change="tabChange">
          <van-tab  :title="$t('trade.marketDepth')">
            <market-dept ref="marketDept" v-model="form.price" :tradePair.sync="tradepair"></market-dept>
          </van-tab>
          <van-tab  :title="$t('trade.myOffer')">
            <my-offers
              ref="myOffers"
              :tradePair.sync="tradepair"
              :address.sync="currentAccount.address"
              :secret.sync="currentAccount.secret"
              @savePasswordInMemory="savePasswordInMemory"></my-offers>
          </van-tab>
        </van-tabs>
      </pl-content-block>
    </van-popup>
    <tradepair-add-pop ref="tradepairAddPop" @done="pairAddDone"></tradepair-add-pop>
    <password-dialog ref="pwdDialog" @done="tradeTx"></password-dialog>
  </div>
</template>
<script>
  // todo: 加入撤单，成交历史记录
  import tradepairAddPop from './popup/tradepair-add-pop';
  import marketDept from './components/market-dept';
  import myOffers from './components/my-offers';
  import passwordDialog from '../ui/password-dialog';
  import Big from 'big.js';
  import cryptor from 'core/utils/cryptor';
  import vueSlider from 'vue-slider-component';
  export default {
    components: {tradepairAddPop, passwordDialog, vueSlider, marketDept, myOffers},
    data () {
      return {
        showPop: false,
        tabActive: 0, /*tab激活*/
        // bsNum: 0, /*滑动百分比*/
        bsFlag: 'buy', /*选中买入按钮还是卖出按钮, 默认buy*/
        bsCode: '', /*买入Code，卖出Code*/
        tradepair: {}, /*交易对*/
        form: {
          price: '',
          amount: ''
        }
      };
    },
    computed: {
      currentAccount () {
        return this.$store.state.account;
      },
      balances () {
        let account = this.$store.state.account;
        if (account) {
          let bs = this.$store.state.balances[account.address];
          if (bs) {
            let result = {};
            bs.forEach((item) => {
              result[item.code] = item.value;
            });
            return result;
          }
        }
        return {};
      },
      expectedTradePrice () {
        if (this.form.price && this.form.amount) {
          return new Big(this.form.price).times(this.form.amount).toFixed(8);
        }
        return 0.000000;
      },
      errMsg () {
        return  {
          'Bad Request': 'Bad Request',
          'NotFoundError': this.$t('exchange.notFoundError'),
          'Network Error': this.$t('transaction.networkError'),
          'manageOfferSellNoTrust': this.$t('trade.manageOfferNoTrust'),
          'manageOfferBuyNoTrust':this.$t('trade.manageOfferNoTrust'),
          'manageOfferMalformed': this.$t('trade.manageOfferMalformed'),
          'manageOfferUnderfunded': this.$t('trade.balancesTip')
        };
      },
      maxVal () {
        if (this.bsFlag === 'buy' && this.form.price > 0) {
          let amount = this.balances[this.bsCode];
          if (amount) {
            return Number(new Big(amount).div(this.form.price).toFixed(8));
          }
        } else if (this.bsFlag === 'sell') {
          return Number(this.balances[this.bsCode]);
        }
        return 100;
      },
      step () {
        if (this.maxVal > 0) {
          return Number(new Big(this.maxVal).div(10).toFixed(8));
        }
        return 1;
      }
    },
    watch: {
      showPop () {
        if (this.$refs.marketDept) {
          this.$refs.marketDept.clearTimer();
        }
        if (this.$refs.myOffers) {
          this.$refs.myOffers.clearTimer();
        }
      }
    },
    methods: {
      show () {
        this.showPop = true;
        this.init();
      },
      init () {
        this.toBs('buy');
        this.form.price = '';
        let account = this.$store.state.account;
        if (account) {
          let tradepair = this.$collecitons.tradepair.findByAcctTypeAndAddress(account.type, account.address);
          if (tradepair && tradepair.length > 0) {
            this.tradepair = tradepair[0];
          } else {
            this.saveDefaultTradepair(account);
          }
          this.bsCode = this.tradepair.counterCode;
          this.$nextTick(() => {
            this.$refs.marketDept.getBooks();
          });
        }
      },
      saveDefaultTradepair (account) {
        let bs = this.$store.state.balances[account.address];
        if (bs && bs.length > 1) {
          let oneBs = bs[1];
          this.tradepair = {
            acctType: account.type,
            acctAddress: account.address,
            baseCode: 'XLM',
            baseIssuer: '',
            counterCode: oneBs.code,
            counterIssuer: oneBs.issuer ? oneBs.issuer: ''
          };
          this.$collecitons.tradepair.insertTradepair(this.tradepair);
        } else {
          this.tradepair = {
            acctType: account.type,
            acctAddress: account.address,
            baseCode: 'XLM',
            baseIssuer: '',
            counterCode: 'CNY',
            counterIssuer: 'GAREELUB43IRHWEASCFBLKHURCGMHE5IF6XSE7EXDLACYHGRHM43RFOX'
          };
        }
      },
      toAddTradepair () {
        this.$refs.tradepairAddPop.show(this.tradepair);
      },
      toBs (flag) {
        this.form.amount = ''; //重置滑块
        this.bsFlag = flag;
        if (flag === 'buy') {
          this.bsCode = this.tradepair.counterCode;
        } else if (flag === 'sell') {
          this.bsCode = this.tradepair.baseCode;
        }
      },
      pairAddDone (tradePair) {
        this.tradepair = tradePair;
        this.bsCode = this.tradepair.counterCode;
        this.tabActive == 0;
        this.toBs('buy');
        this.$nextTick(() => {
          this.$refs.marketDept.getBooks();
        });
      },
      checkForm () {
        if (!this.form.price) {
          this.$toast(this.$t('trade.priceTip'));
          return false;
        }
        if (!this.form.amount) {
          this.$toast(this.$t('trade.amountTip'));
          return false;
        }
        if (this.bsFlag === 'buy') {
          if (Number(this.balances[this.bsCode]) < Number(this.expectedTradePrice)) {
            this.$toast(this.$t('trade.balancesTip'));
            return false;
          } else {
            return true;
          }
        } else if (this.bsFlag === 'sell') {
          if (Number(this.balances[this.bsCode]) < Number(this.form.amount)) {
            this.$toast(this.$t('trade.balancesTip'));
            return false;
          } else {
            return true;
          }
        }
        return true;
      },
      toInputPassword () {
        if (!this.checkForm()) {
          return;
        }
        if (this.$store.state.passwordMap[this.currentAccount.address]) {
          this.tradeTx (this.$store.state.passwordMap[this.currentAccount.address]);
        } else {
          this.$refs.pwdDialog.show();
        }
      },
      getErrMsg (err) {
        if (err && this.errMsg[err]) {
          return this.errMsg[err];
        }
        return null;
      },
      savePasswordInMemory (password) {
        let map = {};
        map[this.currentAccount.address] = password;
        this.$store.dispatch('setPasswordMap', map);
      },
      tradeTx (password) {
        this.savePasswordInMemory(password);
        this.loading = true;
        const toast = this.$toast.loading({
          duration: 0,
          forbidClick: true,
          loadingType: 'circular'
        });
        let selling = {};
        let buying = {};
        let amount = '';
        let price = '';
        if (this.bsFlag === 'buy') {
          selling.code = this.tradepair.counterCode;
          selling.issuer = this.tradepair.counterIssuer;
          buying.code = this.tradepair.baseCode;
          buying.issuer = this.tradepair.baseIssuer;
          amount = this.form.amount * this.form.price;
          price = 1 / this.form.price;
        } else if (this.bsFlag === 'sell') {
          selling.code = this.tradepair.baseCode;
          selling.issuer = this.tradepair.baseIssuer;
          buying.code = this.tradepair.counterCode;
          buying.issuer = this.tradepair.counterIssuer;
          amount = this.form.amount;
          price = this.form.price;
        }
        this.$wallet.sendOffer(selling, buying, amount, price, this.currentAccount.address, cryptor.decryptAES(this.currentAccount.secret, password)).then(ret => {
          if (ret) {
            toast.message = this.$t('trade.offerSuccess');
            setTimeout(() => {
              toast.clear();
              this.tabActive = 1;
            }, 1000);
          }
          this.loading = false;
        }).catch(err => {
          console.info(err);
          let errMsg = this.getErrMsg(err);
          if (errMsg) {
            this.$toast(errMsg);
          } else {
            this.$toast(this.$t('trade.offerFail'));
          }
          setTimeout(() => {
            toast.clear();
          }, 2000);
          this.loading = false;
        });
      },
      tabChange () {
        if (this.tabActive === 0) {
          this.$nextTick(() => {
            this.$refs.myOffers.clearTimer();
            this.$refs.marketDept.getBooks();
          });
        } else if (this.tabActive === 1) {
          this.$nextTick(() => {
            this.$refs.marketDept.clearTimer();
            this.$refs.myOffers.getOffers();
          });
        }
      },
      close () {
        this.showPop = false;
      }
    }
  };
</script>
<style lang="scss" rel="stylesheet/scss" scoped>
  @import "~assets/scss/variables";
  .trade {
    .bs-btn {
      display: inline-block;
      width: 90px;
      text-align: center;
      padding: 5px 10px;
    }
    .bs-block {
      .item {
        padding-top:5px;
        padding-bottom: 5px;
      }
    }
    .buy-btn {
      margin-right: 10px;
      border: 1px solid $primary-color;
      color: $primary-color;
      &.buy {
        background-color: $primary-color;
        color: #ffffff;
      }
    }
    .sell-btn {
      border: 1px solid $danger-color;
      color: $danger-color;
      &.sell {
        background-color: $danger-color;
        color: #ffffff;
      }
    }
  }
</style>
