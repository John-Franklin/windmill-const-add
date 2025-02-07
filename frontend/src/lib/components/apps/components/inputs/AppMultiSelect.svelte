<script lang="ts">
	import { getContext } from 'svelte'
	import Select from 'svelte-select'
	import type { AppInput } from '../../inputType'
	import type { Output } from '../../rx'
	import type { AppViewerContext, ComponentCustomCSS } from '../../types'
	import { concatCustomCss } from '../../utils'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import InputValue from '../helpers/InputValue.svelte'
	import { SELECT_INPUT_DEFAULT_STYLE } from '../../../../defaults'

	export const staticOutputs: string[] = ['result']
	export let id: string
	export let configuration: Record<string, AppInput>
	export let horizontalAlignment: 'left' | 'center' | 'right' | undefined = undefined
	export let verticalAlignment: 'top' | 'center' | 'bottom' | undefined = undefined
	export let customCss: ComponentCustomCSS<'input'> | undefined = undefined
	export let render: boolean

	const { app, worldStore, connectingInput, selectedComponent } =
		getContext<AppViewerContext>('AppViewerContext')
	let items: { label: string; value: string }[]
	let placeholder: string = 'Select an item'

	$: outputs = $worldStore?.outputsById[id] as {
		result: Output<string[] | undefined>
	}

	// $: outputs && handleOutputs()

	// function handleOutputs() {
	// 	value = outputs.result.peak()
	// }

	let value: { value: string }[] | undefined = outputs?.result.peak()

	$: labels && handleItems()

	let labels: string[] | undefined = []

	function handleItems() {
		if (labels) {
			items = labels?.map((label) => {
				return {
					label,
					value: label
				}
			})
		}
	}

	$: value && outputs?.result.set(value.map((v) => v.value))

	$: css = concatCustomCss($app.css?.selectcomponent, customCss)
</script>

<InputValue {id} input={configuration.items} bind:value={labels} />
<InputValue {id} input={configuration.placeholder} bind:value={placeholder} />

<AlignWrapper {render} {horizontalAlignment} {verticalAlignment}>
	<div class="app-select w-full mx-0.5" style="height: 34px" on:pointerdown|stopPropagation>
		{#if !value || Array.isArray(value)}
			<Select
				--border-radius="0"
				--border-color="#999"
				multiple
				on:change={(e) => e.stopPropagation()}
				{items}
				class={css?.input?.class ?? ''}
				inputStyles={SELECT_INPUT_DEFAULT_STYLE.inputStyles}
				containerStyles={'border-color: #999;' +
					SELECT_INPUT_DEFAULT_STYLE.containerStyles +
					css?.input?.style}
				bind:value
				{placeholder}
				on:click={() => {
					if (!$connectingInput.opened) {
						$selectedComponent = id
					}
				}}
				on:focus={() => {
					$selectedComponent = id
				}}
				floatingConfig={{
					strategy: 'fixed'
				}}
			/>
		{:else}
			Value {value} is not an array
		{/if}
	</div>
</AlignWrapper>

<style global>
	.app-select .value-container {
		padding: 0 !important;
	}
	.svelte-select-list {
		z-index: 1000 !important;
	}
</style>
