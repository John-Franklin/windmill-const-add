<script lang="ts">
	import { getContext, afterUpdate } from 'svelte'
	import type { App, AppEditorContext, AppViewerContext, GridItem } from '../types'
	import { classNames } from '$lib/utils'
	import { columnConfiguration, disableDrag, enableDrag, isFixed, toggleFixed } from '../gridUtils'
	import { twMerge } from 'tailwind-merge'

	import RecomputeAllComponents from './RecomputeAllComponents.svelte'
	import type { Policy } from '$lib/gen'
	import HiddenComponent from '../components/helpers/HiddenComponent.svelte'
	import Component from './component/Component.svelte'
	import { deepEqual } from 'fast-equals'
	import { push } from '$lib/history'
	import { expandGriditem, findGridItem } from './appUtils'
	import Grid from '../svelte-grid/Grid.svelte'

	export let policy: Policy

	const {
		selectedComponent,
		app,
		mode,
		connectingInput,
		staticOutputs,
		runnableComponents,
		summary,
		focusedGrid,
		parentWidth,
		breakpoint
	} = getContext<AppViewerContext>('AppViewerContext')

	const { history } = getContext<AppEditorContext>('AppEditorContext')

	// The drag is disabled when the user is connecting an input
	// or when the user is previewing the app
	// or when the focused grid is a subgrid
	$: setAllDrags($mode === 'preview' || $connectingInput.opened)

	function setAllDrags(enable: boolean) {
		const fct = enable ? disableDrag : enableDrag

		$app.grid.map((gridItem) => {
			const disabledGridItem = fct(gridItem)

			if (disabledGridItem?.data?.subGrids) {
				disabledGridItem.data.subGrids = disabledGridItem.data.subGrids.map(
					(subgrid: GridItem[]) => subgrid?.map((subgridItem: GridItem) => fct(subgridItem)) ?? []
				)
			}

			return disabledGridItem
		})

		Object.values($app.subgrids ?? {}).map(
			(subgrid: GridItem[]) => subgrid?.map((subgridItem: GridItem) => fct(subgridItem)) ?? []
		)
	}

	function removeGridElement(component) {
		if (component) {
			$app.grid = $app.grid.filter((gridComponent) => {
				if (gridComponent.data.id === component.id) {
					if (
						gridComponent.data.componentInput?.type === 'runnable' &&
						gridComponent.data.componentInput?.runnable?.type === 'runnableByName' &&
						gridComponent.data.componentInput?.runnable.inlineScript
					) {
						const { name, inlineScript } = gridComponent.data.componentInput?.runnable

						if (!$app.unusedInlineScripts) {
							$app.unusedInlineScripts = []
						}

						$app.unusedInlineScripts.push({
							name,
							inlineScript
						})

						$app = $app
					}
				}

				return gridComponent.data.id !== component?.id
			})

			// Delete static inputs
			delete $staticOutputs[component.id]
			$staticOutputs = $staticOutputs

			delete $runnableComponents[component.id]
			$runnableComponents = $runnableComponents

			$selectedComponent = undefined
		}
	}

	function selectComponent(id: string) {
		// Component selection is handled manually in the Map component (pointerdown
		// event propagation is stopped to enable paning).
		// Update the 'selectComponent()' function as well when this is updated.

		if (!$connectingInput.opened) {
			$selectedComponent = id
			$focusedGrid = undefined
		}
	}

	let pointerdown = false
	let lastapp: App | undefined = undefined
	const onpointerdown = () => {
		if (history) {
			lastapp = JSON.parse(JSON.stringify($app))
			pointerdown = true
		}
	}
	const onpointerup = () => {
		if (history) {
			pointerdown = false
			if (!deepEqual(lastapp, $app)) {
				push(history, lastapp, false, true)
			}
		}
	}

	afterUpdate(() => {
		if ($selectedComponent) {
			const parents = document.querySelectorAll<HTMLElement>('.svlt-grid-item')
			parents.forEach((parent) => {
				const hasActiveChild = !!parent.querySelector('.active-grid-item')
				if (hasActiveChild) {
					parent.style.setProperty('z-index', '100')
				} else {
					parent.style.removeProperty('z-index')
				}
			})
		}
	})
</script>

<div class="relative w-full z-20 overflow-visible">
	<div
		class="w-full sticky top-0 flex justify-between border-b {$connectingInput?.opened
			? ''
			: 'bg-gray-50 '} px-4 py-1 items-center gap-4"
		style="z-index: 1000;"
	>
		<h2 class="truncate">{$summary}</h2>
		{#if !$connectingInput.opened}
			<RecomputeAllComponents />
		{/if}
		<div class="text-2xs text-gray-600">
			{policy.on_behalf_of ? `on behalf of ${policy.on_behalf_of_email}` : ''}
		</div>
	</div>

	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<div
		style={$app.css?.['app']?.['grid']?.style}
		class={twMerge('px-4 pt-4 pb-2 overflow-visible', $app.css?.['app']?.['grid']?.class ?? '')}
		on:pointerdown={onpointerdown}
		on:pointerleave={onpointerup}
		on:pointerup={onpointerup}
		on:click={() => {
			$selectedComponent = undefined
			$focusedGrid = undefined
		}}
		bind:clientWidth={$parentWidth}
	>
		<div class={!$focusedGrid && $mode !== 'preview' ? 'border-gray-400 border border-dashed' : ''}>
			<Grid
				onTopId={$selectedComponent}
				fillSpace={false}
				items={$app.grid}
				on:redraw={(e) => {
					$app.grid = e.detail
				}}
				let:dataItem
				rowHeight={36}
				cols={columnConfiguration}
				fastStart={true}
				gap={[4, 2]}
			>
				<!-- svelte-ignore a11y-click-events-have-key-events -->
				{#if $connectingInput.opened}
					<div
						on:pointerenter={() => ($connectingInput.hoveredComponent = dataItem.id)}
						on:pointerleave={() => ($connectingInput.hoveredComponent = undefined)}
						class="absolute w-full h-full bg-black border-2 bg-opacity-25 z-20 flex justify-center items-center"
					/>
					<div
						style="transform: translate(-50%, -50%);"
						class="absolute w-fit justify-center bg-indigo-500/90 left-[50%] top-[50%] z-50 px-6 rounded border text-white py-2 text-5xl center-center"
					>
						{dataItem.data.id}
					</div>
				{/if}
				<!-- svelte-ignore a11y-click-events-have-key-events -->
				<div
					on:click|stopPropagation
					on:pointerdown={() => selectComponent(dataItem.data.id)}
					class={classNames(
						'h-full w-full center-center',
						$selectedComponent === dataItem.data.id ? 'active-grid-item' : ''
					)}
				>
					<Component
						render={true}
						{pointerdown}
						component={dataItem.data}
						selected={$selectedComponent === dataItem.data.id}
						locked={isFixed(dataItem)}
						on:delete={() => removeGridElement(dataItem.data)}
						on:lock={() => {
							const gridItem = findGridItem($app, dataItem.data.id)
							if (gridItem) {
								toggleFixed(gridItem)
							}
							$app = $app
						}}
						on:expand={() => {
							push(history, $app)
							$selectedComponent = dataItem.data.id
							expandGriditem($app.grid, dataItem, $breakpoint)
							$app = $app
						}}
					/>
				</div>
			</Grid>
		</div>
	</div>
</div>

{#if $app.hiddenInlineScripts}
	{#each $app.hiddenInlineScripts as script, index}
		{#if script}
			<HiddenComponent
				id={`bg_${index}`}
				inlineScript={script.inlineScript}
				name={script.name}
				fields={script.fields}
				autoRefresh={script.autoRefresh ?? false}
				bind:staticOutputs={$staticOutputs[`bg_${index}`]}
			/>
		{/if}
	{/each}
{/if}

<style global>
	.svlt-grid-shadow {
		/* Back shadow */
		background: #93c4fdd0 !important;
	}
	.svlt-grid-active {
		opacity: 1 !important;
	}
</style>
