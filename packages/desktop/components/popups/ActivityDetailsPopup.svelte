<script lang="ts">
    import { localize } from '@core/i18n'
    import { getOfficialExplorerUrl } from '@core/network/utils'
    import {
        Text,
        Button,
        FontWeight,
        TextType,
        BasicActivityDetails,
        AliasActivityDetails,
        FoundryActivityDetails,
        GovernanceActivityDetails,
        NftActivityDetails,
        ConsolidationActivityDetails,
        ActivityInformation,
    } from 'shared/components'
    import { openUrlInBrowser } from '@core/app'
    import {
        ActivityAsyncStatus,
        ActivityDirection,
        ActivityType,
        claimActivity,
        rejectActivity,
        selectedAccountActivities,
    } from '@core/wallet'
    import { activeProfile, checkActiveProfileAuth } from '@core/profile'
    import { setClipboard } from '@core/utils'
    import { truncateString } from '@core/utils'
    import { closePopup, openPopup, PopupId } from '@auxiliary/popup'
    import { onMount } from 'svelte'
    import { ExplorerEndpoint } from '@core/network'
    import { TextHintVariant } from 'shared/components/enums'

    export let activityId: string
    export let _onMount: (..._: any[]) => Promise<void> = async () => {}

    const explorerUrl = getOfficialExplorerUrl($activeProfile?.network?.id)

    $: activity = $selectedAccountActivities.find((_activity) => _activity.id === activityId)
    $: isTimelocked = activity?.asyncData?.asyncStatus === ActivityAsyncStatus.Timelocked
    $: isActivityIncomingAndUnclaimed =
        activity?.asyncData &&
        (activity?.direction === ActivityDirection.Incoming ||
            activity?.direction === ActivityDirection.SelfTransaction) &&
        activity?.asyncData?.asyncStatus === ActivityAsyncStatus.Unclaimed

    function onExplorerClick(): void {
        openUrlInBrowser(`${explorerUrl}/${ExplorerEndpoint.Transaction}/${activity?.transactionId}`)
    }

    function onTransactionIdClick(): void {
        setClipboard(activity?.transactionId)
    }

    async function claim(): Promise<void> {
        await claimActivity(activity)
        openPopup({
            id: PopupId.ActivityDetails,
            props: { activityId },
        })
    }

    async function onClaimClick(): Promise<void> {
        await checkActiveProfileAuth(claim, { stronghold: true, ledger: false })
    }

    function onRejectClick(): void {
        openPopup({
            id: PopupId.Confirmation,
            props: {
                title: localize('actions.confirmRejection.title'),
                description: localize('actions.confirmRejection.description'),
                hint: localize('actions.confirmRejection.node'),
                confirmText: localize('actions.reject'),
                variant: TextHintVariant.Warning,
                onConfirm: () => {
                    rejectActivity(activityId)
                    closePopup()
                },
                onCancel: () =>
                    openPopup({
                        id: PopupId.ActivityDetails,
                        props: { activityId },
                    }),
            },
        })
    }

    onMount(async () => {
        try {
            await _onMount()
        } catch (err) {
            console.error(err)
        }
    })
</script>

<activity-details-popup class="w-full h-full space-y-6 flex flex-auto flex-col shrink-0">
    <div class="flex flex-col">
        <Text type={TextType.h3} fontWeight={FontWeight.semibold} classes="text-left">
            {localize('popups.transactionDetails.title')}
        </Text>
        {#if explorerUrl && activity?.transactionId}
            <button
                class="action w-max flex justify-start text-center font-medium text-14 text-blue-500"
                on:click={onExplorerClick}
            >
                {localize('general.viewOnExplorer')}
            </button>
        {:else if activity?.transactionId}
            <button
                class="action w-fit flex justify-start text-center font-medium text-14 text-blue-500"
                on:click={onTransactionIdClick}
            >
                {truncateString(activity.transactionId, 12, 12)}
            </button>
        {/if}
    </div>
    <activity-details class="w-full h-full space-y-6 flex flex-auto flex-col shrink-0">
        {#if activity.type === ActivityType.Basic}
            <BasicActivityDetails {activity} />
        {:else if activity.type === ActivityType.Foundry}
            <FoundryActivityDetails {activity} />
        {:else if activity.type === ActivityType.Governance}
            <GovernanceActivityDetails {activity} />
        {:else if activity.type === ActivityType.Consolidation}
            <ConsolidationActivityDetails {activity} />
        {:else if activity.type === ActivityType.Nft}
            <NftActivityDetails {activity} />
        {:else if activity.type === ActivityType.Alias}
            <AliasActivityDetails {activity} />
        {/if}
        <ActivityInformation {activity} />
    </activity-details>
    {#if !isTimelocked && isActivityIncomingAndUnclaimed}
        <popup-buttons class="flex flex-row flex-nowrap w-full space-x-4">
            <Button
                outline
                classes="w-full"
                disabled={activity.asyncData?.isClaiming || activity.asyncData?.isRejected}
                onClick={onRejectClick}
            >
                {localize('actions.reject')}
            </Button>
            <Button
                classes="w-full"
                disabled={activity.asyncData?.isClaiming}
                onClick={onClaimClick}
                isBusy={activity.asyncData?.isClaiming}
            >
                {localize('actions.claim')}
            </Button>
        </popup-buttons>
    {/if}
</activity-details-popup>
