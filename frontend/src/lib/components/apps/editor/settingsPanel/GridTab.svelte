<script lang="ts">
	import Button from '$lib/components/common/button/Button.svelte'
	import CloseButton from '$lib/components/common/CloseButton.svelte'
	import { faPlus, faTrashAlt } from '@fortawesome/free-solid-svg-icons'
	import { CrossIcon } from 'lucide-svelte'
	import { getContext } from 'svelte'
	import type { AppViewerContext } from '../../types'
	import { deleteGridItem } from '../appUtils'
	import type { AppComponent } from '../component'
	import PanelSection from './common/PanelSection.svelte'

	export let tabs: string[]
	export let component: AppComponent

	const { app, staticOutputs, runnableComponents, focusedGrid } =
		getContext<AppViewerContext>('AppViewerContext')

	function addTab() {
		const numberOfTabs = tabs.length
		tabs = [...tabs, `Tab ${numberOfTabs + 1}`]

		if (!$app.subgrids) {
			$app.subgrids = {}
		}

		$app.subgrids[`${component.id}-${numberOfTabs}`] = []
		component.numberOfSubgrids = tabs.length
	}

	function deleteSubgrid(index: number) {
		let subgrid = `${component.id}-${index}`
		for (const item of $app!.subgrids![subgrid]) {
			const components = deleteGridItem($app, item.data, subgrid, false)
			console.log(components)
			for (const key in components) {
				delete $staticOutputs[key]
				delete $runnableComponents[key]
			}
		}
		$staticOutputs = $staticOutputs
		$runnableComponents = $runnableComponents
		for (let i = index; i < tabs.length - 1; i++) {
			$app!.subgrids![`${component.id}-${i}`] = $app!.subgrids![`${component.id}-${i + 1}`]
		}
		tabs.splice(index, 1)
		tabs = tabs
		component.numberOfSubgrids = tabs.length
		$app = $app
	}
</script>

<PanelSection title={`Tabs ${tabs.length > 0 ? `(${tabs.length})` : ''}`}>
	{#if tabs.length == 0}
		<span class="text-xs text-gray-500">No Tabs</span>
	{/if}
	<div class="w-full flex gap-2 flex-col mt-2">
		{#each tabs as value, index (index)}
			<div class="w-full flex flex-row gap-2 items-center relative">
				<input on:keydown|stopPropagation type="text" bind:value />

				<div class="absolute right-1">
					<CloseButton noBg on:close={() => deleteSubgrid(index)} />
				</div>
			</div>
		{/each}
		<Button
			size="xs"
			color="light"
			variant="border"
			startIcon={{ icon: faPlus }}
			on:click={addTab}
			iconOnly
		/>
	</div>
</PanelSection>
