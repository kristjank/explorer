<template>
  <Loader :data="transactions">
    <TableWrapper
      v-bind="$attrs"
      :columns="columns"
      :rows="transactions"
      :no-data-message="$t('COMMON.NO_RESULTS')"
      @on-sort-change="emitSortChange"
    >
      <template slot-scope="data">
        <div v-if="data.column.field === 'recipientId'">
          <LinkWallet :address="data.row.recipientId" :type="0" :trunc="false" />
        </div>

        <div v-else-if="data.column.field === 'amount'">
          {{ readableCrypto(data.row.amount) }}
        </div>
      </template>
    </TableWrapper>
  </Loader>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from "vue-property-decorator";
import { mapGetters } from "vuex";
import { IDelegate, ISortParameters, ITransaction } from "@/interfaces";
import CryptoCompareService from "@/services/crypto-compare";
import { paginationLimit } from "@/constants";

@Component({
  computed: {
    ...mapGetters("currency", { currencySymbol: "symbol" }),
    ...mapGetters("network", ["isListed"]),
  },
})
export default class MultiPaymentTransactions extends Vue {
  @Prop({ required: true }) public transaction: ITransaction;
  @Prop({ required: false, default: 0 }) public page: number;
  @Prop({ required: false, default: paginationLimit }) public count: number;

  private currencySymbol: string;
  private isListed: boolean;
  private transactions: Array<{ recipientId: string; amount: string; price: number | null }> | null = null;

  get columns() {
    const columns = [
      {
        label: this.$t("TRANSACTION.RECIPIENTS"),
        field: "recipientId",
        thClass: "start-cell",
        tdClass: "start-cell break-all",
      },
      {
        label: this.$t("TRANSACTION.AMOUNT"),
        field: "amount",
        type: "number",
        thClass: "end-cell",
        tdClass: "end-cell",
      },
    ];

    return columns;
  }

  @Watch("transaction")
  public async onTransactionChanged() {
    await this.prepareTransactions();
  }

  @Watch("currencySymbol")
  public async onCurrencySymbolChanged() {
    await this.updatePrices();
  }

  @Watch("page")
  public async onPageChanged() {
    await this.prepareTransactions();
  }

  public async created() {
    this.prepareTransactions();
  }

  private async prepareTransactions() {
    this.transactions =
      this.page > 0
        ? this.transaction.asset.payments.slice((this.page - 1) * this.count, this.page * this.count)
        : this.transaction.asset.payments;
    await this.updatePrices();
  }

  private async fetchPrice(transaction: { recipientId: string; amount: string; price: number | null }) {
    transaction.price = await CryptoCompareService.dailyAverage(this.transaction.timestamp.unix);
  }

  private async updatePrices() {
    if (!this.transactions) {
      return;
    }

    if (this.isListed) {
      const promises = this.transactions.map(this.fetchPrice);
      await Promise.all(promises);
    }
  }

  private emitSortChange(params: ISortParameters[]) {
    this.$emit("on-sort-change", params[0]);
  }
}
</script>
