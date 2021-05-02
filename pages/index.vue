<template>
  <div>
    <v-data-table
      :headers="headers"
      :items="items"
      :page="page"
      :items-per-page="itemsPerPage"
      show-expand
      :single-expand="singleExpand"
      :loading="$apollo.loading"
      :expanded.sync="expanded"
      :server-items-length="totalPairs"
      :footer-props="{
        showFirstLastPage: true,
        'items-per-page-options': [5, 10, 20, 50, -1],
      }"
      @update:options="onClickPage"
      class="elevation-1"
    >
      <template v-slot:top>
        <v-toolbar flat>
          <v-toolbar-title>Uniswap</v-toolbar-title>
          <v-spacer></v-spacer>
          <v-switch
            v-model="singleExpand"
            label="Single expand"
            class="mt-2"
          ></v-switch>
        </v-toolbar>
      </template>
      <template v-slot:[`item.id`]="{ item }">
        <v-btn icon @click="onClickEtherscan(item)">
          <etherscan-icon />
        </v-btn>
        <v-btn icon @click="onClickUniswap(item)">
          <uniswap-icon />
        </v-btn>
      </template>
      <template v-slot:expanded-item="{ headers, item }">
        <td :colspan="headers.length" class="py-2">
          <v-row>
            <v-col cols="12" md="6">
              <v-card>
                <v-card-title>
                  <a :href="`https://etherscan.io/address/${item.token0.id}`">{{
                    `${item.token0.name} - ${item.token0.symbol}`
                  }}</a>
                </v-card-title>
                <v-card-text>
                  <div>Pooled: {{ item.reserve0 }}</div>
                  <div>
                    {{
                      `1 ${item.token0.symbol} = ${
                        item.reserve1 / item.reserve0
                      } ${item.token1.symbol}`
                    }}
                  </div>
                  <div>Decimals: {{ item.token0.decimals }}</div>
                  <div>Total Supply: {{ item.token0.totalSupply }}</div>
                </v-card-text>
              </v-card>
            </v-col>
            <v-col cols="12" md="6">
              <v-card>
                <v-card-title>
                  <a :href="`https://etherscan.io/address/${item.token1.id}`">{{
                    `${item.token1.name} - ${item.token1.symbol}`
                  }}</a>
                </v-card-title>
                <v-card-text>
                  <div>Pooled: {{ item.reserve1 }}</div>
                  <div>
                    {{
                      `1 ${item.token1.symbol} = ${
                        item.reserve0 / item.reserve1
                      } ${item.token0.symbol}`
                    }}
                  </div>
                  <div>Decimals: {{ item.token1.decimals }}</div>
                  <div>Total Supply: {{ item.token1.totalSupply }}</div>
                </v-card-text>
              </v-card>
            </v-col>
          </v-row>
        </td>
      </template>
    </v-data-table>
  </div>
</template>

<script>
import gql from "graphql-tag";
import { formatCurrency, timeDifference } from "../utils";

export default {
  data() {
    return {
      itemsPerPage: 20,
      page: 1,
      singleExpand: false,
      sortBy: "createdAtTimestamp",
      sortDesc: "desc",
      expanded: [],
      headers: [
        {
          text: "Pairs",
          value: "name",
          sortable: false,
        },
        {
          text: "Link",
          value: "id",
          sortable: false,
        },
        {
          text: "TX Count",
          value: "txCount",
        },
        {
          text: "Total Liquidity",
          value: "reserveUSD",
        },
        {
          text: "Volume",
          value: "volumeUSD",
        },
        {
          text: "Created Time",
          value: "createdAtTimestamp",
        },
      ],
    };
  },
  apollo: {
    pairs: {
      query: gql`
        query getPairs(
          $first: Int
          $skip: Int
          $orderBy: Pair_orderBy
          $orderDirection: OrderDirection
        ) {
          pairs(
            first: $first
            skip: $skip
            orderBy: $orderBy
            orderDirection: $orderDirection
          ) {
            id
            txCount
            volumeUSD
            reserveUSD
            reserve0
            reserve1
            createdAtTimestamp
            token0 {
              id
              symbol
              name
              decimals
              totalSupply
            }
            token1 {
              id
              symbol
              name
              decimals
              totalSupply
            }
          }
        }
      `,
      variables() {
        return {
          first: this.itemsPerPage,
          skip: (this.page - 1) * this.itemsPerPage,
          orderBy: this.sortBy,
          orderDirection: this.sortDesc,
        };
      },
    },
    uniswapFactory: gql`
      query {
        uniswapFactory(id: "0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f") {
          pairCount
        }
      }
    `,
  },
  computed: {
    totalPairs() {
      return typeof this.uniswapFactory === "undefined"
        ? 0
        : this.uniswapFactory.pairCount;
    },
    items() {
      if (typeof this.pairs === "undefined") {
        return [];
      }
      return this.pairs.map((pair) => {
        const {
          id,
          txCount,
          createdAtTimestamp,
          reserveUSD,
          volumeUSD,
          token0,
          token1,
          reserve0,
          reserve1,
        } = pair;

        const createAt = timeDifference(
          new Date().getTime(),
          createdAtTimestamp * 1000
        );

        return {
          name: `${pair.token0.symbol}-${pair.token1.symbol}`,
          id,
          txCount,
          reserveUSD: formatCurrency(reserveUSD),
          createdAtTimestamp: createAt,
          volumeUSD: formatCurrency(volumeUSD),
          token0,
          token1,
          reserve0,
          reserve1,
        };
      });
    },
  },
  methods: {
    onClickPage(options) {
      const { itemsPerPage, page, sortBy, sortDesc } = options;
      if (sortBy.length > 0) {
        this.sortBy = sortBy[0];
        this.sortDesc = sortDesc[0] ? "desc" : "asc";
      }
      this.itemsPerPage = itemsPerPage;
      this.page = page;
    },
    onClickUniswap(item) {
      window.open(`https://info.uniswap.org/pair/${item.id}`, "_blank");
    },
    onClickEtherscan(item) {
      window.open(`https://etherscan.io/address/${item.id}`, "_blank");
    },
  },
};
</script>

<style>
</style>
