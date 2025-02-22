<template>
  <div v-if="isListReady" class="container--assets">
    <div v-if="isXcmAssets" class="container">
      <div class="row">
        <div>
          <span class="text--title">
            {{ $t(width > screenSize.sm ? 'assets.xcmAssets' : 'assets.xcmAssetsShort') }}</span
          >
        </div>
        <asset-search-option
          :toggle-is-hide-small-balances="toggleIsHideSmallBalancesXcm"
          :is-hide-small-balances="isHideSmallBalancesXcm"
          :is-import-modal="false"
          :is-search="isSearchXcm"
          :set-search="setSearchXcm"
          :set-is-search="setIsSearchXcm"
        />
      </div>

      <div v-for="t in filteredXcmTokens" :key="t.symbol">
        <erc-20-currency :token="t" :is-xcm="true" />
      </div>
      <div v-if="searchXcm.length > 0 && filteredXcmTokens.length === 0" class="box--no-result">
        <span class="text--xl">{{ $t('assets.noResults') }}</span>
      </div>
    </div>

    <div class="container">
      <div class="row">
        <div>
          <span class="text--title">{{ $t('assets.assets') }}</span>
        </div>

        <asset-search-option
          :toggle-is-hide-small-balances="toggleIsHideSmallBalances"
          :is-hide-small-balances="isHideSmallBalances"
          :tokens="tokens"
          :is-import-modal="true"
          :is-search="isSearch"
          :set-search="setSearch"
          :set-is-search="setIsSearch"
        />
      </div>

      <div class="border--separator" />

      <div v-if="isDisplayNativeToken" class="rows">
        <div class="row row--details">
          <div class="row__left">
            <div class="column--currency">
              <img class="token-logo" :src="nativeTokenImg" :alt="nativeTokenSymbol" />
              <div v-if="nativeTokenSymbol && currentNetworkName" class="column--ticker">
                <span class="text--title">{{ nativeTokenSymbol }}</span>
                <span class="text--label">{{ currentNetworkName }}</span>
              </div>
              <div v-else>
                <q-skeleton animation="fade" class="skeleton--md" />
              </div>
            </div>
          </div>
          <div class="row__right row__right--evm">
            <div class="column column--balance">
              <div class="column__box">
                <div class="text--accent">
                  <token-balance :balance="String(bal)" :symbol="nativeTokenSymbol" />
                </div>
                <div class="text--label">
                  <span>{{ $n(balUsd) }} {{ $t('usd') }}</span>
                </div>
              </div>
            </div>
            <div class="column--asset-buttons column--buttons--native-token">
              <router-link :to="buildTransferPageLink(nativeTokenSymbol)">
                <button class="btn btn--sm">
                  {{ $t('assets.transfer') }}
                </button>
              </router-link>
              <!-- Only SDN is able to bridge via cBridge at this moment -->
              <a
                v-if="nativeTokenSymbol === 'SDN'"
                :href="cbridgeAppLink"
                target="_blank"
                rel="noopener noreferrer"
              >
                <button class="btn btn--sm">
                  {{ $t('assets.bridge') }}
                </button>
              </a>

              <button
                v-if="isFaucet"
                class="btn btn--sm"
                @click="handleModalFaucet({ isOpen: true })"
              >
                {{ $t('assets.faucet') }}
              </button>
            </div>
          </div>
        </div>
      </div>

      <div v-for="t in filteredTokens" :key="t.symbol">
        <div v-if="checkIsCbridgeToken(t)">
          <evm-cbridge-token v-if="t.symbol !== nativeTokenSymbol" :token="t" />
        </div>
        <div v-else>
          <erc-20-currency :token="t" />
        </div>
      </div>
      <div v-if="search.length > 0 && filteredTokens.length === 0" class="box--no-result">
        <span class="text--xl">{{ $t('assets.noResults') }}</span>
      </div>
    </div>
    <modal-faucet :is-modal-faucet="isModalFaucet" :handle-modal-faucet="handleModalFaucet" />
  </div>
</template>
<script lang="ts">
import { ethers } from 'ethers';
import { $web3 } from 'src/boot/api';
import { cbridgeAppLink, checkIsCbridgeToken } from 'src/c-bridge';
import AssetSearchOption from 'src/components/assets/AssetSearchOption.vue';
import Erc20Currency from 'src/components/assets/Erc20Currency.vue';
import EvmCbridgeToken from 'src/components/assets/EvmCbridgeToken.vue';
import ModalFaucet from 'src/components/assets/modals/ModalFaucet.vue';
import TokenBalance from 'src/components/common/TokenBalance.vue';
import { faucetBalRequirement } from 'src/config/wallets';
import { useAccount, useBalance, useBreakpoints, useNetworkInfo, usePrice } from 'src/hooks';
import { Erc20Token, getTokenImage } from 'src/modules/token';
import { buildTransferPageLink } from 'src/router/routes';
import { useStore } from 'src/store';
import { computed, defineComponent, PropType, ref, watchEffect } from 'vue';

export default defineComponent({
  components: {
    EvmCbridgeToken,
    ModalFaucet,
    Erc20Currency,
    TokenBalance,
    AssetSearchOption,
  },
  props: {
    tokens: {
      type: Object as PropType<Erc20Token[]>,
      required: false,
      default: null,
    },
  },
  setup(props) {
    const isModalFaucet = ref<boolean>(false);
    const isHideSmallBalances = ref<boolean>(false);
    const isHideSmallBalancesXcm = ref<boolean>(false);
    const token = ref<Erc20Token | string | null>(null);
    const symbol = ref<string>('');
    const bal = ref<number>(0);
    const balUsd = ref<number>(0);
    const isShibuya = ref<boolean>(false);
    const isFaucet = ref<boolean>(false);
    const isSearch = ref<boolean>(false);
    const search = ref<string>('');

    const isSearchXcm = ref<boolean>(false);
    const searchXcm = ref<string>('');

    const { width, screenSize } = useBreakpoints();
    const { currentAccount } = useAccount();
    const { nativeTokenUsd } = usePrice();
    const { accountData } = useBalance(currentAccount);

    const store = useStore();
    const isH160 = computed<boolean>(() => store.getters['general/isH160Formatted']);
    const isLoading = computed<boolean>(() => store.getters['general/isLoading']);

    const { currentNetworkName, nativeTokenSymbol, isMainnet } = useNetworkInfo();

    const nativeTokenImg = computed<string>(() =>
      getTokenImage({ isNativeToken: true, symbol: nativeTokenSymbol.value })
    );
    const isListReady = computed<boolean>(() => !!(!isMainnet.value || props.tokens.length > 0));

    const isDisplayNativeToken = computed<boolean>(() => {
      return (
        !search.value || nativeTokenSymbol.value.toLowerCase().includes(search.value.toLowerCase())
      );
    });

    const filteredXcmTokens = computed<Erc20Token[] | []>(() => {
      if (!props.tokens) return [];
      const xcmTokens = props.tokens.filter((it) => {
        if (it.isXC20 && !checkIsCbridgeToken(it)) return it;
      });
      const tokens = isHideSmallBalancesXcm.value
        ? xcmTokens.filter((it) => Number(it.userBalance) > 0)
        : xcmTokens;

      if (!searchXcm.value) return tokens;

      const value = searchXcm.value.toLowerCase();
      const result = tokens
        .map((token: Erc20Token) => {
          const isFoundToken =
            value === token.address.toLowerCase() ||
            token.symbol.toLowerCase().includes(value) ||
            token.name.toLowerCase().includes(value);
          return isFoundToken ? token : undefined;
        })
        .filter((it) => it !== undefined) as Erc20Token[];
      return result.length > 0 ? result : [];
    });

    const filteredTokens = computed<Erc20Token[] | []>(() => {
      if (!props.tokens) return [];
      const erc20Tokens = props.tokens.filter((it) => {
        if (!it.isXC20 || checkIsCbridgeToken(it)) return it;
      });
      const tokens = isHideSmallBalances.value
        ? erc20Tokens.filter((it) => Number(it.userBalance) > 0)
        : erc20Tokens;

      if (!search.value) return tokens;

      const value = search.value.toLowerCase();
      const result = tokens
        .map((token: Erc20Token) => {
          const isFoundToken =
            value === token.address.toLowerCase() ||
            token.symbol.toLowerCase().includes(value) ||
            token.name.toLowerCase().includes(value);
          return isFoundToken ? token : undefined;
        })
        .filter((it) => it !== undefined) as Erc20Token[];
      return result.length > 0 ? result : [];
    });

    const toggleIsHideSmallBalances = (): void => {
      isHideSmallBalances.value = !isHideSmallBalances.value;
    };

    const toggleIsHideSmallBalancesXcm = (): void => {
      isHideSmallBalancesXcm.value = !isHideSmallBalancesXcm.value;
    };

    const handleModalFaucet = ({ isOpen }: { isOpen: boolean }): void => {
      isModalFaucet.value = isOpen;
    };

    const setIsSearch = (isTyping: boolean): void => {
      isSearch.value = isTyping;
    };

    const setIsSearchXcm = (isTyping: boolean): void => {
      isSearchXcm.value = isTyping;
    };

    const setSearch = (event: any): void => {
      search.value = event.target.value;
    };

    const setSearchXcm = (event: any): void => {
      searchXcm.value = event.target.value;
    };

    const updateStates = async (nativeTokenUsd: number): Promise<void> => {
      if (isLoading.value || !nativeTokenSymbol.value || !isH160.value || !$web3.value) return;
      try {
        const balWei = await $web3.value!.eth.getBalance(currentAccount.value);
        bal.value = Number(ethers.utils.formatEther(balWei));
        isShibuya.value = nativeTokenSymbol.value === 'SBY';
        isFaucet.value = isShibuya.value || faucetBalRequirement > bal.value;
        if (nativeTokenUsd) {
          balUsd.value = nativeTokenUsd * bal.value;
        }
      } catch (error: any) {
        console.error(error.message);
      }
    };

    const isXcmAssets = computed<boolean>(() => {
      let result = false;
      if (props.tokens.length === 0) return result;
      props.tokens.some((it) => {
        if (it.isXC20) {
          result = true;
        }
      });
      return result;
    });

    watchEffect(async () => {
      await updateStates(nativeTokenUsd.value);
    });

    return {
      bal,
      nativeTokenSymbol,
      balUsd,
      currentNetworkName,
      isFaucet,
      symbol,
      token,
      nativeTokenImg,
      isListReady,
      isModalFaucet,
      isSearch,
      isSearchXcm,
      search,
      searchXcm,
      filteredXcmTokens,
      filteredTokens,
      isDisplayNativeToken,
      accountData,
      cbridgeAppLink,
      isHideSmallBalances,
      isHideSmallBalancesXcm,
      isLoading,
      isXcmAssets,
      width,
      screenSize,
      buildTransferPageLink,
      setIsSearch,
      handleModalFaucet,
      checkIsCbridgeToken,
      toggleIsHideSmallBalances,
      setSearch,
      setSearchXcm,
      setIsSearchXcm,
      toggleIsHideSmallBalancesXcm,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/assets/styles/asset-list.scss';
</style>
