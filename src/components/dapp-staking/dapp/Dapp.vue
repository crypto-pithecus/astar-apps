<template>
  <div v-if="dapp && dapp.dapp" class="container--dapp-staking">
    <back-to-page :text="$t('dappStaking.stakePage.backToDappList')" :link="Path.DappStaking" />
    <dapp-avatar :dapp="dapp" />
    <dapp-statistics :dapp="dapp" />
    <dapp-images :dapp="dapp" />
    <builders :dapp="dapp" />
    <div class="row--project-overview">
      <project-overview :dapp="dapp" />
      <project-details :dapp="dapp" />
    </div>
    <dapp-stats-charts :dapp="dapp" />
    <div class="bottom--links">
      <router-link :to="buildStakePageLink(dapp.dapp.address)">
        <astar-irregular-button :disabled="isH160" :height="28" class="btn--stake-switch">
          {{ $t('dappStaking.dappPage.stakeOrSwitchTo') }} {{ dapp.dapp.name }}
        </astar-irregular-button>
      </router-link>
      <back-to-page :text="$t('dappStaking.stakePage.backToDappList')" :link="Path.DappStaking" />
    </div>
  </div>
</template>
<script lang="ts">
import BackToPage from 'src/components/common/BackToPage.vue';
import Builders from 'src/components/dapp-staking/dapp/Builders.vue';
import DappAvatar from 'src/components/dapp-staking/dapp/DappAvatar.vue';
import DappImages from 'src/components/dapp-staking/dapp/DappImages.vue';
import DappStatistics from 'src/components/dapp-staking/dapp/DappStatistics.vue';
import DappStatsCharts from 'src/components/dapp-staking/dapp/DappStatsCharts.vue';
import ProjectDetails from 'src/components/dapp-staking/dapp/ProjectDetails.vue';
import ProjectOverview from 'src/components/dapp-staking/dapp/ProjectOverview.vue';
import { useDappRedirect, useDispatchGetDapps, useStakingList } from 'src/hooks';
import { Path } from 'src/router';
import { networkParam } from 'src/router/routes';
import { useStore } from 'src/store';
import { DappCombinedInfo } from 'src/v2/models';
import { computed, defineComponent } from 'vue';
import { useRoute } from 'vue-router';

export default defineComponent({
  components: {
    DappAvatar,
    DappStatistics,
    DappImages,
    Builders,
    ProjectOverview,
    ProjectDetails,
    BackToPage,
    DappStatsCharts,
  },
  setup() {
    const route = useRoute();
    useDappRedirect();
    useDispatchGetDapps();
    const store = useStore();

    const { dapps, stakingList } = useStakingList();
    const dappAddress = computed<string>(() => route.query.dapp as string);
    const isH160 = computed<boolean>(() => store.getters['general/isH160Formatted']);

    const buildStakePageLink = (address: string): string => {
      const base = networkParam + Path.DappStaking + Path.Stake;
      return `${base}?dapp=${address.toLowerCase()}`;
    };

    const goLink = (url: string) => {
      window.open(url, '_blank');
    };

    const dapp = computed(() => {
      console.log('dapps', dapps.value);
      if (dapps.value.length > 0 && dappAddress.value) {
        return dapps.value.find((it: DappCombinedInfo) => {
          try {
            if (!it.dapp) return null;
            return it.dapp.address.toLowerCase() === dappAddress.value.toLowerCase();
          } catch (error) {
            return null;
          }
        });
      }
      return null;
    });

    return {
      Path,
      dapp,
      stakingList,
      goLink,
      buildStakePageLink,
      isH160,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/dapp-staking/dapp/styles/dapp.scss';
</style>
