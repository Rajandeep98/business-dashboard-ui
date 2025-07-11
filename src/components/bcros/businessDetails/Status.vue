<script setup lang="ts">
import { FilingTypes } from '@bcrs-shared-components/enums'
import { AuthorizedActionsE } from '@/enums/authorized-actions-e'
import { isAuthorized } from '@/utils/authorizations'

const t = useNuxtApp().$i18n.t
const {
  currentBusiness,
  currentBusinessIdentifier,
  stateFiling,
  isInLimitedRestoration,
  isFirm
} = storeToRefs(useBcrosBusiness())
const { filings } = storeToRefs(useBcrosFilings())

const { isAuthorizedToAmalgamateOut } = storeToRefs(useBcrosFilings())
const { isAuthorizedToContinueOut } = storeToRefs(useBcrosFilings())

const putBackOff = computed(() =>
  filings.value.find(filing => filing.name === FilingTypes.PUT_BACK_OFF)?.data?.putBackOff || null
)

const getReasonText = computed(() => {
  if (currentBusiness.value.state !== BusinessStateE.HISTORICAL) {
    return ''
  }
  const enDash = '–' // ALT + 0150
  // reason for amalgamation
  if (currentBusiness.value.amalgamatedInto) {
    const name = t('filing.name.amalgamation')
    const amalgamationDate = apiToDate(currentBusiness.value.amalgamatedInto.amalgamationDate)
    if (!amalgamationDate) {
      throw new Error('Invalid amalgamation date')
    }
    const date = dateToPacificDate(amalgamationDate, true)
    const identifier = currentBusiness.value.amalgamatedInto.identifier || t('label.general.unknownCompany')
    return `${name} ${enDash} ${date} ${enDash} ${identifier}`
  }

  const filingType = stateFiling.value?.header?.name
  if (!filingType) {
    return ''
  }

  // reason for dissolution
  if (filingType === FilingTypes.DISSOLUTION) {
    let reason = t('filing.name.unknown')

    const subType = stateFiling.value?.dissolution?.dissolutionType as FilingSubTypeE
    switch (subType) {
      case FilingSubTypeE.DISSOLUTION_ADMINISTRATIVE:
        reason = t('filing.reason.dissolutionAdministrative')
        break
      case FilingSubTypeE.DISSOLUTION_INVOLUNTARY:
        reason = t('filing.reason.involuntaryDissolution')
        break
      case FilingSubTypeE.DISSOLUTION_VOLUNTARY:
        reason = isFirm.value ? t('filing.reason.dissolutionFirm') : t('filing.reason.voluntaryDissolution')
    }

    const dissolutionDate = yyyyMmDdToDate(stateFiling.value?.dissolution?.dissolutionDate)
    const date = !dissolutionDate
      ? dateToPacificDate(new Date(stateFiling.value?.dissolution?.dissolutionDate), true)
      : dateToPacificDate(new Date(dissolutionDate), true)
    if (!dissolutionDate) {
      console.error('Invalid dissolution date')
    }
    return `${reason} ${enDash} ${date}`
  }

  // reason for put back off
  let reason = ''
  if (filingType === FilingTypes.PUT_BACK_OFF && putBackOff.value) {
    const yyyyMmDd = putBackOff.value.expiryDate
    const date = yyyyMmDdToDate(yyyyMmDd)
    const pacificDate = dateToPacificDate(date, true)
    return `${putBackOff.value.reason} on ${pacificDate}`
  }
  // reason for continuation out and default 'reason'
  const effectiveDate = apiToDate(stateFiling.value?.header?.effectiveDate)
  if (!effectiveDate) {
    throw new Error('Invalid effective date')
  }
  const date = dateToPacificDateTime(effectiveDate)
  if (filingType === FilingTypes.CONTINUATION_OUT) {
    reason = t('filing.reason.continuationOut')
  } else {
    reason = t(`filing.name.${filingType}`)
    if (reason === `filing.name.${filingType}`) {
      reason = t('filing.name.unknown')
    }
  }
  return `${reason} ${enDash} ${date}`
})

</script>

<template>
  <div class="flex flex-row gap-1.5 text-sm">
    <template v-if="!!currentBusinessIdentifier">
      <div v-if="currentBusiness.state === BusinessStateE.HISTORICAL" class="flex flex-row gap-1.5">
        <BcrosChips :label="$t('label.business.status.historical')" data-cy="badge.historical" />
        <span>{{ getReasonText }}</span>
      </div>
      <div v-if="currentBusiness.state === BusinessStateE.ACTIVE && isInLimitedRestoration">
        <BcrosChips :label="$t('label.business.status.limitedRestoration')" data-cy="badge.limitedRestoration" />
      </div>
      <div
        v-if="
          currentBusiness.state === BusinessStateE.ACTIVE && isAuthorizedToContinueOut &&
            isAuthorized(AuthorizedActionsE.CONSENT_CONTINUATION_OUT_FILING)
        "
      >
        <BcrosChips
          :label="$t('label.business.status.authorizedToContinueOut')"
          data-cy="badge.authorizedToContinueOut"
        />
      </div>
      <div
        v-if="
          currentBusiness.state === BusinessStateE.ACTIVE && isAuthorizedToAmalgamateOut &&
            isAuthorized(AuthorizedActionsE.CONSENT_AMALGAMATION_OUT_FILING)
        "
      >
        <BcrosChips
          :label="$t('label.business.status.authorizedToAmalgamateOut')"
          data-cy="badge.authorizedToAmalgamateOut"
        />
      </div>
    </template>
  </div>
</template>
