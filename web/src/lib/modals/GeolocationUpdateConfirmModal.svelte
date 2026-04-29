<script lang="ts">
  import type { LatLng } from '$lib/types';
  import { ConfirmModal, Icon } from '@immich/ui';
  import { mdiAlertCircle } from '@mdi/js';
  import { t } from 'svelte-i18n';

  type Props = {
    point: LatLng;
    assetCount: number;
    locationName?: string | null;
    hasCityOrCountry?: boolean;
    hasExistingLocations?: boolean;
    onClose: (confirm: boolean) => void;
  };

  let {
    point,
    assetCount,
    locationName = null,
    hasCityOrCountry = false,
    hasExistingLocations = false,
    onClose,
  }: Props = $props();
</script>

<ConfirmModal title={$t('confirm')} size="small" confirmColor="primary" {onClose}>
  {#snippet prompt()}
    {#if hasExistingLocations}
      <div class="flex items-start gap-3 p-4 mb-4 rounded-lg bg-warning/20 border border-warning">
        <Icon icon={mdiAlertCircle} size="24px" class="text-warning shrink-0 mt-1" />
        <div class="text-warning font-semibold">
          {$t('some_assets_already_have_a_location_warning')}
        </div>
      </div>
    {/if}

    <p class="mb-2">{$t('update_location_action_prompt', { values: { count: assetCount } })}</p>

    <ul class="list-disc ml-8">
      {#if hasCityOrCountry && locationName}
        <li>{locationName} ({point.lat.toFixed(3)}, {point.lng.toFixed(3)})</li>
      {:else}
        <li>{$t('latitude')}: {point.lat.toFixed(3)}</li>
        <li>{$t('longitude')}: {point.lng.toFixed(3)}</li>
      {/if}
    </ul>
  {/snippet}
</ConfirmModal>
