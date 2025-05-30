<template>
  <common-popup>
    <template #header>
      <sign-logo class="common-popup__logo" />
      <div class="common-popup__network">
        <img :src="network.icon" />
        <p>{{ network.name_long }}</p>
      </div>
    </template>

    <template #content>
      <h2>Connect to {{ Options.title }}</h2>

      <div class="common-popup__block no-inset">
        <div class="common-popup__account">
          <img :src="Options.faviconURL" />
          <div class="common-popup__account-info">
            <h4>{{ Options.title }}</h4>
            <p>{{ Options.domain }}</p>
          </div>
        </div>
      </div>

      <div class="provider-connect-dapp__link">
        <link-icon />
      </div>

      <div class="common-popup__block no-inset no-padding">
        <select-account-input
          :name="accountHeaderData.selectedAccount?.name"
          :address="displayAddress"
          :identicon="identicon"
          @toggle:select-accounts="toggleAccounts"
        />
      </div>

      <div class="provider-connect-dapp__info">
        <info-icon-gray />
        <p>
          This will reveal your public address, wallet balance and activity to
          {{ Options.domain }}
        </p>
      </div>

      <modal-accounts
        :show-accounts="showAccounts"
        :close="toggleAccounts"
        :network="network"
        :account-info="accountHeaderData"
        @address-changed="onSelectedAddressChanged"
      />
    </template>

    <template #button-left>
      <base-button title="Decline" :click="decline" :no-background="true" />
    </template>

    <template #button-right>
      <base-button title="Connect" :click="connect" />
    </template>
  </common-popup>
</template>

<script setup lang="ts">
import { onBeforeMount, ref } from 'vue';
import SignLogo from '@action/icons/common/sign-logo.vue';
import BaseButton from '@action/components/base-button/index.vue';
import commonPopup from '@action/views/common-popup/index.vue';
import LinkIcon from '@action/icons/connect/link-icon.vue';
import InfoIconGray from '@action/icons/common/info-icon-gray.vue';
import SelectAccountInput from '@action/components/select-account-input/index.vue';
import ModalAccounts from '@action/views/modal-accounts/index.vue';
import { AccountsHeaderData } from '@action/types/account';
import { EnkryptAccount } from '@enkryptcom/types';
import {
  DEFAULT_SOLANA_NETWORK,
  getNetworkByName,
} from '@/libs/utils/networks';
import { WindowPromiseHandler } from '@/libs/window-promise';
import { ProviderRequestOptions } from '@/types/provider';
import PublicKeyRing from '@/libs/keyring/public-keyring';
import { fromBase } from '@enkryptcom/utils';
import { getError } from '@/libs/error';
import { ErrorCodes } from '@/providers/ethereum/types';
import AccountState from '../libs/accounts-state';
import { SolanaNetwork } from '../types/sol-network';

const windowPromise = WindowPromiseHandler(1);
const network = ref<SolanaNetwork>(DEFAULT_SOLANA_NETWORK);
const identicon = ref<string>('');
const displayAddress = ref<string>('');
const accountHeaderData = ref<AccountsHeaderData>({
  activeAccounts: [],
  inactiveAccounts: [],
  selectedAccount: {
    name: '',
    address: '',
  } as EnkryptAccount,
  activeBalances: [],
});

const Options = ref<ProviderRequestOptions>({
  domain: '',
  faviconURL: '',
  title: '',
  url: '',
  tabId: 0,
});

onBeforeMount(async () => {
  const { Request, options } = await windowPromise;
  network.value = (await getNetworkByName(
    Request.value.params![0],
  )) as SolanaNetwork;
  const keyring = new PublicKeyRing();
  Options.value = options;
  const accounts = await keyring.getAccounts(network.value.signer);
  displayAddress.value = network.value.displayAddress(accounts[0].address);
  identicon.value = network.value.identicon(displayAddress.value);
  accountHeaderData.value = {
    activeAccounts: accounts,
    selectedAccount: accounts[0],
    activeBalances: accounts.map(() => '~'),
    inactiveAccounts: [],
  };
  try {
    const api = await network.value.api();
    const activeBalancePromises = accountHeaderData.value.activeAccounts.map(
      acc => api.getBalance(acc.address),
    );
    Promise.all(activeBalancePromises).then(balances => {
      accountHeaderData.value.activeBalances = balances.map(bal =>
        fromBase(bal, network.value.decimals),
      );
    });
  } catch (e) {
    console.error(e);
  }
});

const showAccounts = ref(false);

const decline = async () => {
  const { Resolve } = await windowPromise;
  Resolve.value({
    error: getError(ErrorCodes.userRejected),
  });
};

const connect = async () => {
  const { Resolve } = await windowPromise;
  const accountState = new AccountState();
  await accountState.addApprovedAddress(
    accountHeaderData.value.selectedAccount!.address,
    Options.value.domain,
  );
  Resolve.value({
    result: JSON.stringify([accountHeaderData.value.selectedAccount!.address]),
  });
};

const toggleAccounts = () => {
  showAccounts.value = !showAccounts.value;
};

const onSelectedAddressChanged = async (newAccount: EnkryptAccount) => {
  accountHeaderData.value.selectedAccount = newAccount;
  displayAddress.value = network.value.displayAddress(newAccount.address);
  identicon.value = network.value.identicon(displayAddress.value);
};
</script>

<style lang="less">
@import '@/providers/ethereum/ui/styles/common-popup.less';
@import '@action/styles/provider-connect-dapp.less';
</style>
