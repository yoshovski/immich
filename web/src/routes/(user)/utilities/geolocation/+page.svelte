<script lang="ts">
  import { isDefined } from '$lib';
  import UserPageLayout from '$lib/components/layouts/UserPageLayout.svelte';
  import EmptyPlaceholder from '$lib/components/shared-components/EmptyPlaceholder.svelte';
  import Timeline from '$lib/components/timeline/Timeline.svelte';
  import { AssetAction } from '$lib/constants';
  import { assetMultiSelectManager } from '$lib/managers/asset-multi-select-manager.svelte';
  import { authManager } from '$lib/managers/auth-manager.svelte';
  import type { TimelineDay } from '$lib/managers/timeline-manager/timeline-day.svelte';
  import { TimelineManager } from '$lib/managers/timeline-manager/timeline-manager.svelte';
  import type { TimelineAsset } from '$lib/managers/timeline-manager/types';
  import GeolocationPointPickerModal from '$lib/modals/GeolocationPointPickerModal.svelte';
  import GeolocationUpdateConfirmModal from '$lib/modals/GeolocationUpdateConfirmModal.svelte';
  import type { LatLng } from '$lib/types';
  import { setQueryValue } from '$lib/utils/navigation';
  import { toTimelineAsset } from '$lib/utils/timeline-util';
  import { AssetVisibility, getAssetInfo, reverseGeocode, updateAssets } from '@immich/sdk';
  import { Button, LoadingSpinner, modalManager, Text, Tooltip } from '@immich/ui';
  import { mdiContentCopy, mdiMapMarkerMultipleOutline, mdiPencilOutline, mdiSelectRemove } from '@mdi/js';
  import { t } from 'svelte-i18n';
  import type { PageData } from './$types';

  type Props = {
    data: PageData;
  };

  let { data }: Props = $props();

  let isLoading = $state(false);
  let point = $state<LatLng>();
  let locationUpdated = $state(false);
  let selectedLocationName = $state('0.000, 0.000');
  let hasCityOrCountry = $derived(!!(selectedCity || selectedCountry));
  let selectedCity = $state<string | null>(null);
  let selectedCountry = $state<string | null>(null);

  let timelineManager = $state<TimelineManager>() as TimelineManager;
  let showOnlyWithoutGps = $state(true);
  const options = $derived({
    visibility: AssetVisibility.Timeline,
    withStacked: true,
    withPartners: true,
    withCoordinates: true,
    hasCoordinates: showOnlyWithoutGps ? false : undefined,
  });

  const handleUpdate = async () => {
    if (!point) {
      return;
    }

    const hasExistingLocations = assetMultiSelectManager.assets.some((asset) => asset.latitude || asset.longitude);

    const confirmed = await modalManager.show(GeolocationUpdateConfirmModal, {
      point,
      assetCount: assetMultiSelectManager.assets.length,
      locationName: hasCityOrCountry ? selectedLocationName : null,
      hasCityOrCountry,
      hasExistingLocations,
    });

    if (!confirmed) {
      return;
    }

    await updateAssets({
      assetBulkUpdateDto: {
        ids: assetMultiSelectManager.assets.map((asset) => asset.id),
        latitude: point.lat,
        longitude: point.lng,
      },
    });

    const lat = point.lat;
    const lng = point.lng;
    const city = selectedCity;
    const country = selectedCountry;

    const assetIds = assetMultiSelectManager.assets.map((a) => a.id);
    timelineManager.update(assetIds, (asset) => {
      asset.latitude = lat;
      asset.longitude = lng;
      if (!asset.city) {
        asset.city = city;
      }
      if (!asset.country) {
        asset.country = country;
      }
    });

    assetMultiSelectManager.clear();
  };

  const onKeyDown = (event: KeyboardEvent) => {
    if (event.key === 'Shift') {
      event.preventDefault();
    }
    if (event.key === 'Escape' && assetMultiSelectManager.selectionActive) {
      assetMultiSelectManager.clear();
    }
  };
  const onKeyUp = (event: KeyboardEvent) => {
    if (event.key === 'Shift') {
      event.preventDefault();
    }
  };

  const updateLocationDisplay = (
    loc: { latitude: number; longitude: number },
    city?: string | null,
    country?: string | null,
  ) => {
    const fallback = `${loc.latitude.toFixed(3)}, ${loc.longitude.toFixed(3)}`;
    selectedCity = city ?? null;
    selectedCountry = country ?? null;

    if (city && country) {
      selectedLocationName = `${city}, ${country}`;
    } else if (city) {
      selectedLocationName = city;
    } else if (country) {
      selectedLocationName = country;
    } else {
      selectedLocationName = fallback;
    }
  };

  const handlePickPoint = async () => {
    const selected = await modalManager.show(GeolocationPointPickerModal, { point });
    if (!selected) {
      return;
    }

    point = selected;
    const fallback = `${point.lat.toFixed(3)}, ${point.lng.toFixed(3)}`;
    try {
      const places = await reverseGeocode({
        lat: point.lat,
        lon: point.lng,
      });
      updateLocationDisplay({ latitude: point.lat, longitude: point.lng }, places?.[0]?.city, places?.[0]?.country);
    } catch (error) {
      console.error('Failed to reverse geocode:', error);
      selectedCity = null;
      selectedCountry = null;
      selectedLocationName = fallback;
    }
  };

  const handleEscape = () => {
    if (assetMultiSelectManager.selectionActive) {
      assetMultiSelectManager.clear();
      return;
    }
  };

  type AssetPoint = { latitude: number; longitude: number };

  const hasGps = (asset: TimelineAsset | AssetPoint): asset is AssetPoint =>
    isDefined(asset.latitude) && isDefined(asset.longitude);

  const handleThumbnailClick = (
    asset: TimelineAsset,
    timelineManager: TimelineManager,
    timelineDay: TimelineDay,
    onClick: (
      timelineManager: TimelineManager,
      assets: TimelineAsset[],
      groupTitle: string,
      asset: TimelineAsset,
    ) => void,
  ) => {
    onClick(timelineManager, timelineDay.getAssets(), timelineDay.groupTitle, asset);
  };

  const onLocationInfoClick = (asset: TimelineAsset) => {
    if (!hasGps(asset)) {
      return;
    }
    locationUpdated = true;
    setTimeout(() => {
      locationUpdated = false;
    }, 1500);
    point = { lat: asset.latitude!, lng: asset.longitude! };
    updateLocationDisplay({ latitude: asset.latitude!, longitude: asset.longitude! }, asset.city, asset.country);
    void setQueryValue('at', asset.id);
  };
</script>

<svelte:document onkeydown={onKeyDown} onkeyup={onKeyUp} />

<UserPageLayout title={data.meta.title} scrollbar={true}>
  {#snippet buttons()}
    <div class="flex gap-2 justify-end place-items-center">
      <div
        class="flex w-fit rounded-full bg-gray-200 dark:bg-gray-700 p-1 text-sm font-medium font-mono overflow-hidden"
      >
        <Button
          size="small"
          variant={showOnlyWithoutGps ? 'ghost' : 'filled'}
          color={showOnlyWithoutGps ? 'secondary' : 'primary'}
          class={`flex-1 px-4 py-1 text-center transition-colors duration-200 rounded-full ${!showOnlyWithoutGps ? 'bg-primary text-light' : ''}`}
          onclick={() => (showOnlyWithoutGps = false)}
        >
          <Text class="whitespace-nowrap">{$t('all')}</Text>
        </Button>
        <Button
          size="small"
          variant={showOnlyWithoutGps ? 'filled' : 'ghost'}
          color={showOnlyWithoutGps ? 'primary' : 'secondary'}
          class={`flex-1 px-4 py-1 text-center transition-colors duration-200 rounded-full ${showOnlyWithoutGps ? 'bg-primary text-light' : ''}`}
          onclick={() => (showOnlyWithoutGps = true)}
        >
          <Text class="whitespace-nowrap">{$t('no_gps')}</Text>
        </Button>
      </div>
      <div class="border flex place-items-center place-content-center px-2 py-1 bg-primary/10 rounded-2xl">
        <Text class="hidden md:inline-block font-mono mr-5 ml-2" color="muted" size="tiny">
          {$t('selected_gps_location')}
        </Text>
        <Text
          title={hasCityOrCountry ? `${point?.lat.toFixed(3)}, ${point?.lng.toFixed(3)}` : $t('latitude_longitude')}
          class="rounded-3xl font-mono text-sm text-primary px-2 py-1 transition-all duration-100 ease-in-out {locationUpdated
            ? 'bg-primary/90 text-light font-semibold scale-105'
            : ''}"
        >
          {#if point}
            {selectedLocationName}
          {:else}
            {$t('none')}
          {/if}
        </Text>
      </div>

      <Button
        size="small"
        color="secondary"
        variant="ghost"
        leadingIcon={mdiMapMarkerMultipleOutline}
        onclick={handlePickPoint}
      >
        <Text class="hidden sm:inline-block">{$t('location_picker_choose_on_map')}</Text>
      </Button>
      <Button
        leadingIcon={mdiSelectRemove}
        size="small"
        color="secondary"
        variant="ghost"
        disabled={!assetMultiSelectManager.selectionActive}
        onclick={() => assetMultiSelectManager.clear()}
      >
        {$t('unselect_all')}
      </Button>
      <Button
        leadingIcon={mdiPencilOutline}
        size="small"
        color="primary"
        disabled={assetMultiSelectManager.assets.length === 0}
        onclick={() => handleUpdate()}
      >
        <Text class="hidden sm:inline-block">
          {$t('apply_count', { values: { count: assetMultiSelectManager.assets.length } })}
        </Text>
      </Button>
    </div>
  {/snippet}

  {#if isLoading}
    <div class="h-full w-full flex items-center justify-center">
      <LoadingSpinner size="giant" />
    </div>
  {/if}

  <Timeline
    isSelectionMode={true}
    enableRouting={true}
    bind:timelineManager
    {options}
    assetInteraction={assetMultiSelectManager}
    removeAction={AssetAction.ARCHIVE}
    onEscape={handleEscape}
    withStacked
    onThumbnailClick={handleThumbnailClick}
  >
    {#snippet customThumbnailLayout(asset: TimelineAsset)}
      {#if hasGps(asset)}
        <div
          class="absolute bottom-1 left-1/2 -translate-x-1/2 z-10 pointer-events-auto w-max max-w-[calc(100%-2.5rem)]"
        >
          <Tooltip text={$t('copy_location_coordinates')}>
            {#snippet child({ props })}
              <button
                {...props}
                type="button"
                class="px-3 py-1 rounded-xl text-xs transition-colors bg-success text-black flex items-center gap-2 cursor-pointer group hover:scale-105 w-full"
                onclick={(e) => {
                  e.stopPropagation();
                  onLocationInfoClick(asset);
                }}
              >
                <div class="flex-grow truncate">
                  {#if asset.city && asset.country}
                    {asset.city}, {asset.country}
                  {:else if asset.city}
                    {asset.city}
                  {:else if asset.country}
                    {asset.country}
                  {:else}
                    {$t('gps')}
                  {/if}
                </div>

                <svg class="h-4 w-4 shrink-0" viewBox="0 0 24 24">
                  <path fill="currentColor" d={mdiContentCopy} />
                </svg>
              </button>
            {/snippet}
          </Tooltip>
        </div>
      {:else}
        <div
          class="absolute bottom-1 left-1/2 -translate-x-1/2 z-10 pointer-events-auto px-4 py-1 rounded-xl text-xs transition-colors bg-danger text-light w-max max-w-[calc(100%-2.5rem)] truncate"
        >
          {$t('gps_missing')}
        </div>
      {/if}
    {/snippet}
    {#snippet empty()}
      <EmptyPlaceholder text={$t('no_assets_message')} onClick={() => {}} class="mt-10 mx-auto" />
    {/snippet}
  </Timeline>
</UserPageLayout>
