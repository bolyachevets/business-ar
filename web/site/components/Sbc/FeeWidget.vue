<script setup lang="ts" generic="T">
import { storeToRefs } from 'pinia'
import { useI18n } from 'vue-i18n'
import { computed, onMounted } from 'vue'
import { usePayFeesStore } from '@/stores/pay-fees'

provide('sbc-bar-success-text', 'payment success')
provide('sbc-bar-platform-info', 'platform info')
provide('sbc-bar-index1', 0)
provide('sbc-bar-index2', 1)
const props = defineProps({
  fees: { type: Array<PayFeesWidgetItem>, required: true },
  isLoading: { type: Boolean, default: false },
  altPaymentMethodAllowed: { type: Boolean, default: false },
  selectedPaymentMethod: { type: String, default: '' },
  possiblePaymentMethods: { type: Array<PaymentMethod>, default: [] },
  payMethodLabel: { type: String, default: '' }
})

defineEmits<{(e: 'submit'): void}>()

// const hasEmptyFees = computed(() => !props.fees?.length)

const payFeesStore = usePayFeesStore()
const {
  userSelectedPaymentMethod,
  allowedPaymentMethods,
  allowAlternatePaymentMethod,
  userPaymentAccount
} = storeToRefs(payFeesStore)

const { t } = useI18n()

const paymentMethodLabel = computed(() => {
  return t(`payment.methodLabels.${userSelectedPaymentMethod.value}`, {
    account: userPaymentAccount?.value?.cfsAccount?.bankAccountNumber
  })
})

const baseText = 'Paying with'

const displayCanadianDollars = (amount: number) => {
  if (!amount) {
    return '$0.00'
  }

  return `$${(amount.toFixed(2))}`
}

const total = computed(() => {
  return props.fees.reduce((total, feeInfo) => total + feeInfo.total, 0)
})

onMounted(async () => {
  try {
    await payFeesStore.initAlternatePaymentMethod()
  } catch (e) {
    console.error('Failed to initialize alternate payment method', e)
  }
})

</script>
<template>
  <div class="flex flex-col gap-4 font-bold">
    <UCard
      data-cy="pay-fees-widget"
      :ui="{
        base: 'w-full min-w-[230px] lg:w-[282px]',
        header: {
          base: 'rounded-t-lg',
          background: 'bg-bcGovColor-header',
          padding: 'p-4 sm:px-4'
        },
        body: {
          base: '',
          background: '',
          padding: 'p-4 sm:px-4',
        },
        footer: {
          base: '',
          background: '',
          padding: 'p-4 sm:px-4',
        },
      }"
    >
      <template #header>
        <span data-cy="pay-fees-widget-title" class="text-white">{{ $t('widgets.feeSummary.title') }}</span>
      </template>

      <div
        v-for="fee in fees"
        :key="fee.uiUuid"
        class="flex flex-col gap-3"
      >
        <div class="flex items-center justify-between">
          <span class="mr-auto text-sm text-bcGovColor-darkGray">
            {{ $t(`widgets.feeSummary.itemLabels.${fee.filingTypeCode}`) }}
          </span>
          <span class="whitespace-nowrap text-sm">
            {{ fee.total === 0 ? $t('widgets.feeSummary.noFee') : displayCanadianDollars(fee.filingFees) }}
          </span>
        </div>
        <UDivider class="my-2" :ui="{ border: { base: 'flex border-bcGovGray-200' }}" />
        <div class="ml-4 flex items-center justify-between">
          <span class="mr-auto text-sm text-bcGovColor-darkGray">
            {{ $t(`widgets.feeSummary.serviceFees`) }}
          </span>
          <span class="whitespace-nowrap text-sm">
            {{ fee.total === 0 ? $t('widgets.feeSummary.noFee') : displayCanadianDollars(fee.serviceFees) }}
          </span>
        </div>
      </div>

      <template #footer>
        <div class="space-y-4">
          <div class="flex items-center justify-between" data-cy="pay-fees-widget-total">
            <span class="mr-auto text-sm text-bcGovColor-darkGray">
              {{ $t('widgets.feeSummary.total') }}
            </span>
            <div class="flex items-center gap-2">
              <span class="text-sm font-normal text-gray-700">
                {{ $t('currency.cad') }}
              </span>
              <span
                v-if="fees?.length > 0"
                class="overflow-hidden whitespace-nowrap text-2xl"
              >
                {{ displayCanadianDollars(total) }}
              </span>
              <span
                v-else
                class="overflow-hidden whitespace-nowrap text-2xl"
              >
                -
              </span>
            </div>
          </div>

          <div v-if="allowAlternatePaymentMethod" class="w-full pt-2">
            <UDivider class="my-2" :ui="{ border: { base: 'flex border-bcGovGray-200' }}" />
            <USelectMenu
              v-slot="{ open }"
              v-model="userSelectedPaymentMethod"
              :options="allowedPaymentMethods"
              value-attribute="value"
              class="w-full"
              :ui-menu="{
                option: {
                  base: 'cursor-pointer font-normal',
                  size: 'text-xs'
                },
                base: '-mt-1',
                rounded: 'rounded-b',
                width: 'w-full'
              }"
              :ui="{
                size: { lg: 'h-min' },
                color: {
                  gray: {
                    outline: 'bg-white ring-0 hover:bg-gray-200 hover:border-gray-600 focus:border-none focus:ring-0',
                  }
                },
                border: 'border-0',
                ring: 'ring-0'
              }"
            >
              <button
                class="flex w-full items-center justify-between gap-4 py-2 px-0 text-left hover:bg-gray-100"
              >
                <span class="mr-auto text-sm font-normal text-bcGovColor-darkGray">
                  {{ baseText }} {{ paymentMethodLabel }}
                </span>
                <UIcon
                  name="i-mdi-caret-down"
                  class="text-blue-500 transition-transform !size-5"
                  :class="[open && 'rotate-180']"
                  style="min-width: 1rem; min-height: 1rem;"
                />
              </button>
            </USelectMenu>
          </div>
        </div>
      </template>
    </UCard>
    <UButton
      :label="$t('btn.submitAndPay')"
      :loading="isLoading"
      block
      @click="$emit('submit', userSelectedPaymentMethod.value)"
    />
  </div>
</template>

<style scoped>
.payment-method-section {
  border-top: 1px solid #e5e7eb;
  padding-top: 1rem;
  margin-top: 1rem;
}
</style>
